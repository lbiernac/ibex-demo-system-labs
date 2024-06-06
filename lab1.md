# Lab 1: Programming your FPGA and interacting with the demo system

Welcome to the first lab on using the [Ibex demo system](https://github.com/lowRISC/ibex-demo-system). In this lab, we will:

- Set up your development environment.
- Learn how to build the software.
- Program our board with a bitstream.
- Run the software on the board.
- Read from the UART.
- Interact with the Ibex using GDB.

</br>

## Getting Started with the Ibex Demo System

### 1. Download the Ibex Demo System Repository
From the Ubuntu-20.04 terminal (either standalone or in VS Code), clone ibex-demo-system repository using the following terminal commands:
```bash
git clone https://github.com/lowRISC/ibex-demo-system
cd ibex-demo-system
```

This tutorial was pulled from commit `c25aeeb` and done on Windows 11. You can pull a specific commit from GitHub using the additional terminal command: 
```bash
git checkout c25aeeba59a695ee7846c8c39a2ddc11230c2656
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
sudo docker run -it --rm \
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

[Verilator](https://verilator.org) is a widely used simulator for Verilog and SystemVerilog.
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



</br></br>
## Running hello_world on Ibex atop the Arty-7 FPGA Board 

### 1. Forwarding the USB Device to WSL and Docker
Plug in the FPGA to a USB port on your machine. To program the FPGA from the docker container, we first have to forward the USB device from Windows to WSL. Then, we have to forward the device from WSL (Ubuntu) to the docker container. 

#### Forward a USB Device from Windows to WSL 
We will first connect a USB device to a Linux distribution running on WSL 2 using the USB/IP open-source project, [usbipd-win](https://github.com/dorssel/usbipd-win).
These instructions are copied from [https://learn.microsoft.com/en-us/windows/wsl/connect-usb](https://learn.microsoft.com/en-us/windows/wsl/connect-usb). 

Go to the latest release page for the [usbipd-win project](https://github.com/dorssel/usbipd-win/releases). 
Select the .msi file, which will download the installer. (You may get a warning asking you to confirm that you trust this download).
Run the downloaded `usbipd-win_x.msi` installer file.

Now, we must identify and bind the FPGA USB port to WSL using `usbipd`. From Windows _PowerShell_ in administrator mode, enter the following command: 
```bash
usbipd list
```

Identify the BUS-ID corresponding to the Arty FPGA board. If it is unclear which device is the board, you can take an experimentalist approach to identify the board by running `usbipd list` while the board is disconnected, then connected, to note which entry appears. 

Once you have the BUS-ID of the FPGA, run the following command to share the device and then bind it with WSL. In the following command, replace <BUS-ID> with the one you have identified. 
```bash
usbipd bind --busid <BUS-ID>
usbipd attach --wsl --busid <BUS-ID>
```

Open the Ubuntu-20.04 terminal and check that you can see the FPGA board by running the following command:
```bash
lsusb
...
Bus 001 Device 002: ID 0403:6010 Future Technology Devices International, Ltd FT2232C/D/H Dual UART/FIFO IC
...
```
If the board appears, we have successfully forwarded the USB device to the Ubuntu kernel. In the next step, we will forward it from the Ubuntu kernel to the docker container. 


#### Forward a USB Device from WSL to Docker
Next, we need to make the FPGA accessible from inside the container.
To do this, we find out which bus and device our Arty is on. From the Ubuntu 20.04 terminal, run the following command:
```console
$ lsusb
...
Bus 00X Device 00Y: ID 0403:6010 Future Technology Devices International, Ltd FT2232C/D/H Dual UART/FIFO IC
...
```
The command `lsusb` will list multiple devices, including the Arty board, as shown above. In this example, `X` and `Y` are numbers that we will use in the following command. Note down the value of X and Y for your machine. ***These values will change if you unplug and replug your FPGA.***


Then, exit/stop the current docker instance and re-run it with the following parameters from the Ubuntu 20.04 terminal:
```bash
sudo docker run -it --rm \
  -p 6080:6080 \
  -p 3333:3333 \
  -v $(pwd):/home/dev/demo:Z \
  --privileged \
  --device=/dev/bus/usb/00X/00Y \
  --device=/dev/ttyUSB1 \
  ibex
```

Now you will be able to access the FPGA board via USB from inside of the docker container.

### 2. Add udev rules for our device
You will need to add user device permissions for our FPGA board.
The following instructions are for Linux-based systems and are needed for the programmer to access the development board.
_If using a docker container, the following commands should be run from inside the container._

Arty-A7
```bash

sudo su
cat <<EOF > /etc/udev/rules.d/90-arty-a7.rules
# Future Technology Devices International, Ltd FT2232C/D/H Dual UART/FIFO IC
# used on Digilent boards
ACTION=="add|change", SUBSYSTEM=="usb|tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6010", ATTRS{manufacturer}=="Digilent", MODE="0666"

# Future Technology Devices International, Ltd FT232 Serial (UART) IC
ACTION=="add|change", SUBSYSTEM=="usb|tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", MODE="0666"
EOF

exit
```

openFPGAloader
```bash
sudo su
cat <<EOF > /etc/udev/rules.d/99-openfpgaloader.rules
# Copy this file to /etc/udev/rules.d/

ACTION!="add|change", GOTO="openfpgaloader_rules_end"

# gpiochip subsystem
SUBSYSTEM=="gpio", MODE="0664", GROUP="plugdev", TAG+="uaccess"

SUBSYSTEM!="usb|tty|hidraw", GOTO="openfpgaloader_rules_end"

# Original FT232/FT245 VID:PID
ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", MODE="664", GROUP="plugdev", TAG+="uaccess"

# Original FT2232 VID:PID
ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6010", MODE="664", GROUP="plugdev", TAG+="uaccess"

# Original FT4232 VID:PID
ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6011", MODE="664", GROUP="plugdev", TAG+="uaccess"

# Original FT232H VID:PID
ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6014", MODE="664", GROUP="plugdev", TAG+="uaccess"

# Original FT231X VID:PID
ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6015", MODE="664", GROUP="plugdev", TAG+="uaccess"

# anlogic cable
ATTRS{idVendor}=="0547", ATTRS{idProduct}=="1002", MODE="664", GROUP="plugdev", TAG+="uaccess"

# altera usb-blaster
ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6001", MODE="664", GROUP="plugdev", TAG+="uaccess"
ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6002", MODE="664", GROUP="plugdev", TAG+="uaccess"
ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6003", MODE="664", GROUP="plugdev", TAG+="uaccess"

# altera usb-blasterII - uninitialized
ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6810", MODE="664", GROUP="plugdev", TAG+="uaccess"
# altera usb-blasterII - initialized
ATTRS{idVendor}=="09fb", ATTRS{idProduct}=="6010", MODE="664", GROUP="plugdev", TAG+="uaccess"

# dirtyJTAG
ATTRS{idVendor}=="1209", ATTRS{idProduct}=="c0ca", MODE="664", GROUP="plugdev", TAG+="uaccess"

# Jlink
ATTRS{idVendor}=="1366", ATTRS{idProduct}=="0105", MODE="664", GROUP="plugdev", TAG+="uaccess"

# NXP LPC-Link2
ATTRS{idVendor}=="1fc9", ATTRS{idProduct}=="0090", MODE="664", GROUP="plugdev", TAG+="uaccess"

# NXP ARM mbed
ATTRS{idVendor}=="0d28", ATTRS{idProduct}=="0204", MODE="664", GROUP="plugdev", TAG+="uaccess"

# icebreaker bitsy
ATTRS{idVendor}=="1d50", ATTRS{idProduct}=="6146", MODE="664", GROUP="plugdev", TAG+="uaccess"

# orbtrace-mini dfu
ATTRS{idVendor}=="1209", ATTRS{idProduct}=="3442", MODE="664", GROUP="plugdev", TAG+="uaccess"

LABEL="openfpgaloader_rules_end"

EOF

exit

```

Run the following to reload the rules for the container:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

During these steps, if you encounter the error message:
```console
Failed to send reload request: No such file or directory
```
It may be because _udev_ is not running. You can start it manually via the following command: 
```bash
sudo /lib/systemd/systemd-udevd --daemon
```

Add user to plugdev group:
```bash
sudo usermod -a $USER -G plugdev
```

### 3. Getting the FPGA bitstream
Download the [FPGA bitstream from GitHub (v0.0.3)](https://github.com/lowRISC/ibex-demo-system/releases/download/v0.0.3/lowrisc_ibex_demo_system_0.bit). Put the bitstream at the root of your demo system repository.

Alternatively, you can build your own bitstream if you have access to Vivado by following the instructions in [the README](https://github.com/lowRISC/ibex-demo-system/blob/main/README.md). This is not needed for the current lab. 


### 4. Programming the FPGA (FAILED HERE, possibly because of USB Hub use)
Then, inside the docker container at [localhost:6080/vnc.html](http://localhost:6080/vnc.html), we program the FPGA with the following terminal command:
```bash
openFPGALoader -b arty_a7_100t /home/dev/demo/lowrisc_ibex_demo_system_0.bit
```

### 5. Loading the software
Before we load the software, please check that you have OpenOCD installed:
```bash
openocd --version
```
Please also check that you have version 0.11.0.

Then you can load and run the program as follows:
```bash
util/load_demo_system.sh run ./sw/build/demo/hello_world/demo
```

Congratulations! You should now see the green LEDs zipping through and the RGB LEDs dimming up and down with different colors each time.

### 6. Interacting with the UART
Besides the LEDs, the demo application also prints to the UART serial output. You can see this output using the following command:
```bash
screen /dev/ttyUSB1 115200
```
If you see an immediate `[screen is terminating]`, it may mean that you need super user rights. In this case, you may try adding `sudo` before the `screen` command. To exit from the `screen` command, you should press control and a together, then release these two keys and press d.

### Exercise 1
While the demo application is running try toggling the switches and buttons, you should see changes in the input value that is displayed by `screen`.

### Exercise 2
Adjust the demo system application to print "Hello Ibex" instead of "Hello World". You should adjust the content of the file in `sw/c/demo/hello_world/main.c`. Afterwards, you should rebuild the software, but do not need to rebuild the bitstream. Once you've built your updated software, you can load the software onto the board to see the result.

### Exercise 3
Write to the green LEDs using GDB. Look in `sw/common/demo_system_regs.h` for the value of `GPIO_BASE`. Use the set command above to write `0xa0` to this base address. This should change the green LEDs to be on, off, on and off.




</br> </br>
## Debugging using GDB
We can use OpenOCD to connect to the JTAG on the FPGA. We can then use GDB to connect to OpenOCD and interact with the board as we would when we debug any other application.

First, let's load the software in the halted state:
```bash
util/load_demo_system.sh halt ./sw/build/demo/hello_world/demo
```

In a separate terminal window, you can connect GDB to the OpenOCD server:
```bash
riscv32-unknown-elf-gdb -ex "target extended-remote localhost:3333" \
    ./sw/c/build/demo/hello_world/demo
```

Some useful GDB commands:

- `step`: steps to the next instruction.
- `advance main`: keep running until the start of the main function (you can replace main with another function).
- `set {int}0xhex_addr = 0xhex_val`: write `hex_val` to the memory address `hex_addr`.
- `x/w 0xhex_addr`: read a word (32 bits) from `hex_addr` and display it in hexidecimal.
- `info locals`: shows you the values of the local variables for the current function.
- `backtrace`: shows you the call stack.
- `help`: to find commands you may not yet know.

</br>
