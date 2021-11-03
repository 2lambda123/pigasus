# Pigasus

<img align="right" width="200" src="./pigasus_logo.png">

Pigasus is an Intrusion Detection and Prevention System (IDS/IPS) that achieves 100Gbps using a single FPGA-equipped server. Pigasus's FPGA-first design ensures that most packets are processed entirely using the FPGA, while some packets are sent to the CPU for full evaluation. Refer to the [OSDI '20 paper](https://www.usenix.org/conference/osdi20/presentation/zhao-zhipeng) for details about the design.

If you have already configured your system and want to run Pigasus, you can skip directly to [Running Pigasus](#running-pigasus).

## Getting Started

Pigasus has a hardware component, that runs on an FPGA, and a software component which is adapted from [Snort3](https://github.com/snort3/snort3). The current version requires a host with a multi-core CPU and an [Intel Stratix 10 MX FPGA](https://www.intel.com/content/www/us/en/programmable/products/boards_and_kits/dev-kits/altera/kit-s10-mx.html) (with 100Gb Ethernet). Note that Pigasus was initially built for Engineering Sample Board with the part number 1SM21BHU2F53E2VGS1. We have updated the scripts to support the Production Board with the part number 1SM21BHU2F53E1VG. The instructions that follow assume the system runs Ubuntu 16.04. They may be slightly different for other distributions.

To start, clone this repository and go to the `legacy` branch:
```bash
git clone https://github.com/crossroadsfpga/pigasus.git
git checkout legacy
```

If you just want to do RTL simulation, please jump to [Developing Pigasus](#developing-pigasus). The [Software Configuration](#software-configuration) is only necessary for running Pigasus in a real system. 

### Software Configuration

To start, add the following to your `~/.bashrc` or equivalent, make sure you set the `pigasus_rep_dir` variable to point to the Pigasus repository you just cloned:

```bash
#### MAKE SURE THIS POINTS TO THE PIGASUS REPOSITORY
export pigasus_rep_dir=$HOME/pigasus # We assume the pigasus repo is at the home dir, by default
####################################################

export LD_LIBRARY_PATH=/usr/local/lib:${LD_LIBRARY_PATH}

# this is installing snort with pigasus on your home dir, you can specify another location if you want
export installation_path_pigasus=$HOME/pigasus_install

export LUA_PATH="$installation_path_pigasus/include/snort/lua/?.lua;;"

export SNORT_LUA_PATH="$pigasus_rep_dir/software/lua"
export LUA_CONFIG_FILE="$SNORT_LUA_PATH/snort.lua"

alias pigasus="taskset --cpu-list 0 $installation_path_pigasus/bin/snort"

# ensure that aliases also work when using sudo
alias sudo='sudo '
```

Make sure these changes are applied:

```bash
source ~/.bashrc
```

To ensure that the Lua variables are passed to sudo open the sudoers config file with:

```bash
sudo visudo
```

And add the following line to the opened file:

```
Defaults env_keep += "LUA_PATH SNORT_LUA_PATH"
```

Save and close the file.

Then install the dependencies using the provided script:

```bash
cd $pigasus_rep_dir
./install_deps.sh
```

### Building Software

```bash
cd $pigasus_rep_dir/software
./configure_cmake.sh --prefix=$installation_path_pigasus --enable-pigasus --enable-tsc-clock --builddir=build_pigasus
cd build_pigasus
make -j $(nproc) install
```

### Hardware Configuration


Download and install [Quartus 19.3](https://fpgasoftware.intel.com/19.3/?edition=pro) and the Stratix 10 device support (same link). You need Quartus in order to load the bitstream into the FPGA.

Add the following to your `~/.bashrc` or equivalent. Make sure that you set `quartus_dir` to the quartus installation directory:
```bash
#### MAKE SURE THIS POINTS TO THE QUARTUS INSTALLATION DIRECTORY
export quartus_dir=
####################################################
export INTELFPGAOCLSDKROOT="$quartus_dir/19.3/hld"
export QUARTUS_ROOTDIR="$quartus_dir/19.3/quartus"
export QSYS_ROOTDIR="$quartus_dir/19.3/qsys/bin"
export IP_ROOTDIR="$quartus_dir/19.3/ip/"
export PATH=$quartus_dir/19.3/quartus/bin:$PATH
export PATH=$quartus_dir/19.3/modelsim_ase/linuxaloem:$PATH
export PATH=$quartus_dir/19.3/quartus/sopc_builder/bin:$PATH
```

Make sure these changes are applied:

```bash
source ~/.bashrc
```


## Running Pigasus

Once both the software and hardware components are configured, you can run Pigasus by building it, loading the bitstream, and then running the software.

Generate the Quartus IP cores used by Pigasus and create a Quartus project. This step could take about 10 minutes.
```bash
cd $pigasus_rep_dir/hardware/scripts/
./run_ipgen.sh
./run_quartus_create.sh
```

Generate the bitstream. This step could take a few hours. We suggest running this step using a multi-core CPU with a large memory (e.g., 64GB). 
```bash
cd $pigasus_rep_dir/hardware/scripts/
./run_quartus_synth.sh
```

Load the bitstream to the Intel MX development kit.
```bash
cd $pigasus_rep_dir/hardware/hw_test
./load_bitstream.sh
```

Note that the JTAG system console should be closed when loading the bitstream and the USB port number should be adjusted according to the machine.

After the bitstream finishes loading, reboot the machine:
```bash
sudo reboot
```

The PCIe core on the FPGA will be recognized after the reboot.

Once the system finishes booting, run the JTAG system console with:
```bash
cd $pigasus_rep_dir/hardware/hw_test
./run_console
```

After the console starts, run:
```
source path.tcl
```

> It may return some error the first time. Exit it using Ctrl-C. Then relaunch the console (`./run_console`) and rerun `source path.tcl`.

In the tcl console, you will need to type some tcl commands.

In the tcl console, set the buffer size:
```
set_buf_size 262143
```

And set the number of cores:
```
set_core_num 1
```

You can also verify the FPGA counter stats with:
```
get_results
```

You may consider leaving the tcl console open while you run Pigasus to let you check the counters. Once you are ready to exit it, use Ctrl-C.

To run the software, first insert the kernel module:
```bash
cd $pigasus_rep_dir/software/src/pigasus/pcie/kernel/linux
sudo ./install
```

Then execute the software application, specifying the `snort.lua` configuration file and the rule_list file (specifying the `sid` of the rules are used). We only provide a sample rule as the Snort Registered Rules requires registration on [Snort](https://www.snort.org/downloads). This sample rule will be triggered 10 times by our sample trace `$pigasus_rep_dir/hardware/rtl_sim/input_gen/m10_100.pcap`.
```bash
cd $pigasus_rep_dir/software/lua
sudo pigasus -c snort.lua --patterns ./rule_list
```

### Example using DPDK pktgen

We provide an example setup with two machines. One hosts Pigasus (Intel Stratix 10 MX FPGA + Full Matcher running on CPU), the other generates packets. We use DPDK pkt-gen to push empirical traces to the attached 100G Mellanox NIC. The Mellanox NIC is connected back-to-back with the Pigasus FPGA through a 100Gb cable. 

Follow the [instructions to run Pigasus](#running-pigasus) and then start the packet generator in the other machine:
```
cp example.pcap /dev/shm/test.pcap
cd dpdk/pktgen-dpdk/
./run_pktgen.sh
```
> Refer to the [DPDK Pktgen docs](https://pktgen-dpdk.readthedocs.io/en/latest/index.html) for details on how to run it.

Remember to set the number of packets to match the pcap, otherwise the packet generator sends the pcap repeatedly, and set the rate. Y = 1 means 1 Gbps.
```
set 0 count X
set 0 rate Y
str
```

After the packet generator finishes sending the number of packets that you specified, recheck the counters on the FPGA using the tcl console:
```
## recheck the top counters, you can see the recv pkts, processed pkts and dma pkts.
get_results
```

If you are using our sample rule and pcap (`m10_100.pcap`). You should expect to see the results below.
```
IN_PKT: 100
PROCESSED_PKT: 100
DMA_PKT: 10
```

Now exit the software application with Ctrl-C. You should expect to see that the number of `rx_pkt` should match up the `DMA_PKT`. 

### Hardware-only Test

We also provide an easy way to just exercise the FPGA datapath without interacting with the software. In this case, you don't need to install and run the software. 
After loading the bitstream to the FPGA, you can just run the JTAG system console without rebooting, as PCIe is not used in this experiment. 

```bash
cd $pigasus_rep_dir/hardware/hw_test
./run_console
```

After the console starts, run:
```
source path.tcl
disable_pcie
```

Now you can start the packet generator and read the counter on FPGA side. The `DMA_PKT` will be discarded in this experiment. 
This is a faster way to get the FPGA running and an effective way to isolate any potential hardware/software integration issues. 

## Developing Pigasus

New developers may find a description of each hardware component [here](hardware/README.md).

### Hardware Development Requirements

To change the hardware component of Pigasus you will need Quartus as well as an RTL simulation tool. Here we use Modelsim-SE (requires license) as our simulation tool. We do not recommend using the Modelsim-Altera_Start_Edition (Modelsim-ASE) -- the free version that comes with Quartus -- to simulate Pigasus, as Pigasus exceeds the line limit of Modelsim-ASE, which leads to extremely slow simulation. We have not tested other simulation tools like VCS.

If you have not yet installed Quartus, download and install [Quartus 19.3](https://fpgasoftware.intel.com/19.3/?edition=pro) and the Stratix 10 device support (same link). Then follow the next steps to prepare the RTL code for simulation and synthesis.

Before setting up the simulation and synthesis, if you have not yet generated the IP cores, do:
```bash
cd $pigasus_rep_dir/hardware/scripts/
./run_ipgen.sh
```

Simulation Setup:
1. Compile Quartus 19.3 IP library for RTL simulation. Open Quartus 19.3. Select "Launch Simulation Library Compiler" under the "Tools" Tab. Select "ModelSim" as the "Tool name" and specify the path for "Executable location." Then select "Stratix 10" as Library families. Specify the output directory and click "Start Compilation".
2. Add the compiled library path as the environment variable, by `export SIM_LIB_PATH=your_install_path/sim_lib/verilog_libs`


Synthesis Setup:
1. Create a Quartus project.
```bash
cd $pigasus_rep_dir/hardware/scripts/
./run_quartus_create.sh
```

### RTL Simulation

Before you synthesize your changes and run them on the FPGA, you should verify your design using RTL simulation. The simulation testbed does not include the Ethernet IP core, PCIe IP core, and full matcher on CPU. The `testbench.sv` drives the FPGA datapth, including parser, TCP Reassembly and MSPM by dumping the converted pcap file. Once the simulation finishes, the testbench will report statistics of FPGA datapth to help developers verify functionality and diagnose any performance bottlenecks. The following instructions assume that you have Modelsim installed. 

Go to the input generator and convert the example pcap into a pkt format, that can be used as input for the simulation:

```bash
cd $pigasus_rep_dir/hardware/rtl_sim/input_gen/
./run.sh ./m10_100.pcap
cd ..
```

Now run the simulation using Modelsim and your newly generated input:
```bash
./run_vsim.sh ./input_gen/output.pkt
```

After the simulation, you should expect the key counter values to match up with the `input_gen/example_expected_res`

We also provide an example script `run_vsim.bat` for Windows users. Please set up the `SIM_LIB_PATH` and the RTL simulator environment variables before running this script in cmd or powershell. 

Tips:
To enable GUI mode in Modelsim, comment the last line of `run_vsim.sh` and uncomment the line under "GUI full debug."
To speedup the simulation, you can disable MSPM in your first few runs by uncommenting the second line of `mspm/string_matcher/string_matcher.sv`. The MSPM will be a dummy module which just passes the packets. 



### Hardware Synthesis
```bash
cd $pigasus_rep_dir/hardware/scripts
./run_quartus_synth.sh
```


## License

Pigasus is developed at Carnegie Mellon University. The hardware component of Pigasus (`/hardware`) is released under the [BSD 3-Clause Clear License](hardware/LICENSE). The software component (`/software`) is adapted from Snort3 and released under the [GNU General Public License v2.0](software/LICENSE).
