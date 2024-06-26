# Lab 1: Getting Started with Ibex and Simulating with Verilator

Welcome to the first lab on using the [Ibex demo system (public)](https://github.com/lbiernac/ibex-demo-system). In this lab, we will:

- Set up your development environment.
- Learn how to build the software.
- Simulate the system with Verilator.
- Run a hello_world software on Ibex with Verilator.

</br>

## Getting Started with the Ibex Demo System

### 1. Download the Ibex Demo System Repository
From the Ubuntu-20.04 terminal (either standalone or in VS Code), clone the `ibex-demo-system` repository using the following terminal commands:
```bash
git clone https://github.com/lbiernac/ibex-demo-system.git
cd ibex-demo-system
```

This tutorial is based on commit `c25aeeb` of [lowRISC's ibex-demo-system repository](https://github.com/lowRISC/ibex-demo-system) and tested on Windows 11 using WSL with Ubuntu 20.04. The [fork of ibex-demo-system](https://github.com/lbiernac/ibex-demo-system.git) used for this tutorial includes changes to the Docker file to work correctly with this tutorial. 

If using the _private_ fork of `ibex-demo-system` for research development purposes, instead clone [this repository](https://github.com/lbiernac/ibex-development) using the command:
```bash
git clone --recursive git@github.com:lbiernac/ibex-development.git
cd ibex-development
```



### 2. Build the Ibex docker container (via WSL)
Within the ibex-demo-system repository, there is a set of instructions (_container/README.md_) on how to build the docker container locally. These steps are repeated here. 

To build the container, navigate to the directory `ibex-demo-system` in the Linux terminal and run the following command: 
```bash
sudo docker build . -t ibex -f container/Dockerfile
```

This command will install all of the necessary dependencies for the Ibex SoC and will take several minutes. This command may fail on different versions of Ubuntu, such as Ubuntu 22.04. You can check your OS version by running: `cat /etc/os-release`. 


</br></br>

## Running the Container using Docker Desktop and VS Code

### 1. Starting the Container
Once the container is built, you can use the following command in the root of the repository (`ibex-demo-system`) to start the container. This command should be run from the Ubuntu 20.04 terminal (standalone or inside VS Code). 

```sh
sudo docker run -it \
  -p 6080:6080 \
  -p 3333:3333 \
  -v $(pwd):/home/dev/demo:Z \
  ibex
```

To access the container go to [http://localhost:6080/vnc.html](http://localhost:6080/vnc.html). This link provides a GUI interface for the container's OS. 


### 2. Accessing the container's shell from Docker Desktop
After the container is running, in _Docker Desktop_ navigate to "Containers" and clock on the container that is currently running. From here, you can select "Files" to view the file system of the container. The command to start the container will mount the directory "ibex-demo-system" to `home/dev/demo`. Explore `home/dev/demo` to view the mirrored files. 

You can also select "Exec" to access a terminal (shell) and issue Linux commands inside the container. These commands will execute with all the dependent software installed as part of the container. 

Running Linux commands from Docker Desktop may not be ideal for development. The following section details how to view the container terminal from VS Code. 


### 3. Accessing the container's shell from VS Code
If developing inside of VS Code, it can be convenient to access the container terminal from inside of VS Code. To do so, you will need the Docker VS Code extension installed. In _VS Code_, navigate to "Docker" in the left-hand toolbar. Under "Individual Containers", right-click the running container "Ibex" and select "Attach Shell". This will open a shell for the container under VS Code's Terminal pane. Any commands you run here will be run inside of the docker container!


### 4. Make the container's shell prompt look nice
The command prompt for the docker shell defaults to `sh-5.0$`. You can run the following command to set a more readable command prompt: 
```bash
export PS1='\[\033[1;32m\]\u@\h\[\e[0m\]:\[\033[1;34m\]\w\[\e[0m\]\$ '
```
In this command, `\u` refers to the current user's username, `\h` refers to the hostname, and `\w` gets the current working directory. The remaining commands are used for formatting. 


</br></br>

## Building Software for Ibex (C Stack)

First, the software must be built. This can be loaded into an FPGA to run on a synthesized Ibex processor or passed to a verilator simulation model to be simulated on a PC. 

### 1. Compile the C Workloads
To build the software, use the following commands in your terminal from the root of the repository (`ibex-demo-system`).

_If using a docker container, the following commands should be run from inside the container._

```bash
mkdir -p sw/c/build
pushd sw/c/build
cmake ..
make
popd
```

This builds the software that we can later use as memory content for the Ibex running in the demo system. For example, the binary for the demo application is located at `sw/c/build/demo/hello_world/demo`.


### 2. Analyze the RISC-V Binary
The demo application compiled in the previous step, `sw/c/build/demo/hello_world/demo`, is a RISC-V binary file. We can examine the contents of the binary file by disassembling it to read the RISC-V instructions via the following command.  
_If using a docker container, the following commands should be run from inside the container._

```bash
riscv32-unknown-elf-objdump -d demo > demo.dump
```

This command takes the file named `demo` in the current directory, disassembles it, and writes the output to a new file named `demo.dump`. 



</br></br>

## Running hello_world on Ibex simulated with Verilator


[Verilator](https://verilator.org) is a widely used open-source simulator for Verilog and SystemVerilog.
It compiles RTL code to a multithreaded C++ executable, which makes Verilator much faster than other simulators in many cases.
Verilator is free software (licensed LGPLv3). Verilator comes installed in the Ibex docker container. You do not need to install it separately.


### 1. Building the Simulation

The ibex-demo-system simulator binary can be built via FuseSoC. From the container, run the following command inside of the ibex-demo-system repository (`home/dev/demo`).

_If using a docker container, the following commands should be run from inside the container._

```
fusesoc --cores-root=. run --target=sim --tool=verilator --setup --build lowrisc:ibex:demo_system
```

This will create a Verilator executable for the current Ibex build. The executable is located at `/build/lowrisc_ibex_demo_system_0/sim-verilator/Vtop_verilator`, relative to the ibex-demo-system root directory.



### 2. Running the Simulator

In previous steps, we have created a simulated implementation of Ibex and a compiled RISC-V binary. We can now run the simulator by invoking the Verilator executable.

  
#### Running a Simulation with the `demo` Binary

The following command starts the simulation using the `demo` binary, which was compiled from `hello_world.c`. This binary is located at the path `/sw/c/build/demo/hello_world/demo`, relative to the ibex-demo-system root directory. The command to invoke the simulator takes the entire path and is run from the root directory inside of the container. 
```
./build/lowrisc_ibex_demo_system_0/sim-verilator/Vtop_verilator --meminit=ram,./sw/c/build/demo/hello_world/demo
```


#### Running a Simulation with any Binary

The following general command can be used for any desired binary when completing the field `<sw_elf_file>`. 
```
./build/lowrisc_ibex_demo_system_0/sim-verilator/Vtop_verilator [-t] --meminit=ram,<sw_elf_file>
```

This command uses two flags: 
- `-t` (optional): Enables the capturing of an FST trace of execution that can be viewed with [GTKWave](http://gtkwave.sourceforge.net/). GTKWave is used in Lab 2. 
- `--meminit`: Loads the binary at the path `<sw_elf_file>` into program memory


#### Observing the Simulation

As no outputs (such as UART or GPIO) are mapped to the Verilator console, the console output does not indicate much activity.
However, the simulation is actively running and wave traces are continuously being written to a file called `sim.fst`.
Additionally, the UART output is continuously written to a file in the root directory called `uart0.log`. You can watch this file using the following terminal command (run either in the container or in Ubuntu):
```bash
tail -f uart0.log
```

The simulation will run until you press `Ctrl+C`.



### Exercise 1
Adjust the demo system application to print "Hello Ibex" instead of "Hello World". You should adjust the content of the file in `sw/c/demo/hello_world/main.c`. Afterward, ***you should rebuild the software*** using the above instructions. You do not need to rebuild the simulator, since the Ibex source code is unchanged. Once you've built your updated software workload, you can restart the simulator to see the result. 
