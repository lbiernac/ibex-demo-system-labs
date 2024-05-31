# Lab 1: Programming your FPGA and interacting with the demo system

Welcome to the first lab on using the [Ibex demo system](https://github.com/lowRISC/ibex-demo-system). In this lab, we will:

- Set up your development environment.
- Learn how to build the software.
- Program our board with a bitstream.
- Run the software on the board.
- Read from the UART.
- Interact with the Ibex using GDB.



## Getting started with the Ibex SoC Demo System
From the Ubuntu-20.04 terminal (either standalone or in VS Code), clone ibex-demo-system repository using the following terminal commands:
```bash
git clone https://github.com/lowRISC/ibex-demo-system
cd ibex-demo-system
```

This tutorial was pulled from commit `c25aeeb` and done on Windows 11. You can pull a specific commit from GitHub using the additional terminal command: 
```bash
git checkout c25aeeba59a695ee7846c8c39a2ddc11230c2656
```


### Build the Ibex docker container (via WSL)
Within the ibex-demo-system repository, there is a set of instructions (_container/README.md_) to build the docker container locally. These steps are repeated here. 

To build the container, navigate to the directory `ibex-demo-system/container` in the Linux terminal and run the following command: 
```bash
sudo docker build . -t ibex -f container/Dockerfile
```

This command will install all of the necessary dependencies for the Ibex SoC and will take several minutes. This command may fail on different versions of Ubuntu, such as Ubuntu 22.04. You can check your OS version by running: `cat /etc/os-release`. 



## Running and interacting with the docker container using Docker Desktop and VS Code
Once the container is built, you can use the following command in the root of the repository (`ibex-demo-system`) to start the container. This command should be run from the Ubuntu 20.04 terminal (standalone or inside VS Code). 

```sh
sudo docker run -it --rm \
  -p 6080:6080 \
  -p 3333:3333 \
  -v $(pwd):/home/dev/demo:Z \
  ibex
```

To access the container go to [http://localhost:6080/vnc.html](http://localhost:6080/vnc.html). This link provides a GUI interface for the container's OS. 


### Accessing the container's shell from Docker Desktop
After the container is running, in _Docker Desktop_ navigate to "Containers" and clock on the container that is currently running. From here, you can select "Files" to view the file system of the container. The command to start the container will mount the directory "ibex-demo-system" to `home/dev/demo`. Explore `home/dev/demo` to view the mirrored files. 

You can also select "Exec" to access a terminal (shell) and issue Linux commands inside the container. These commands will execute with all of dependent software installed as part of the container. 

Running Linux commands from Docker Desktop may not be ideal for development. The following section details how to view the container terminal from VS Code. 


### Accessing the container's shell from VS Code
If developing inside of VS Code, it can be convenient to access the container terminal from inside of VS Code. To do so, you must have the Docker VS Code extension installed. In _VS Code_, navigate to "Docker" in the left-hand toolbar. Under "Individual Containers", right-click the running container "Ibex" and select "Attach Shell". This will open a shell for the container under VS Code's Terminal pane. Any commands you run here will be run inside of the docker container!


### Make the container's shell prompt look nice
The terminal for the docker shell defaults to the prompt `sh-5.0$`. You can run the following command to set a more user-friendly command prompt: 
```sh
  export PS1='\[\033[1;32m\]\u@\h\[\e[0m\]:\[\033[1;34m\]\w\[\e[0m\]\$ '
```
In this command, `\u` refers to the username of the current user, `h` refers to the hostname, and `\w` gets the current working directory. The remaining commands are used for formatting. 



## Building software (C Stack)
First the software must be built. This can be loaded into an FPGA to run on a synthesized Ibex processor, or passed to a verilator simulation model to be simulated on a PC. To build the software, use the following commands in your terminal from the root of the repository (`ibex-demo-system`).
_If using a docker container, the following commands should be run from inside the container._

```bash
mkdir -p sw/c/build
pushd sw/c/build
cmake ..
make
popd
```

This builds the software that we can later use as memory content for the Ibex running in the demo system. For example, the binary for the demo application is located at `sw/c/build/demo/hello_world/demo`.



## Analyzing the RISC-V Binary
The demo application compiled in the previous step, `sw/c/build/demo/hello_world/demo`, is a RISC-V binary file. We can examine the contents of the binary file by disassembling it to read the RISC-V instructions via the following command.  
_If using a docker container, the following commands should be run from inside the container._

```bash
riscv-32-unknown-elf-objdump -d demo > demo.dump
```

This command takes the file named `demo` in the current directory, disassembles it, and writes the output to a new file named `demo.dump`. 




# Lab 1 Part A: Running hello_world on Ibex simulated with Verilator
***TODO***




# Lab 1 Part B: Running hello_world on Ibex atop the Arty-7 FPGA Board

## Add udev rules for our device
For both the container and the native setups you will need to add user device permissions for our FPGA board.
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

Run the following to reload the rules:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Add user to plugdev group:
```bash
sudo usermod -a $USER -G plugdev
```


## Getting the FPGA bitstream
Get the FPGA bitstream off of the USB or download the [FPGA bitstream from GitHub](https://github.com/lowRISC/ibex-demo-system/releases/download/v0.0.2/lowrisc_ibex_demo_system_0.bit). Put the bitstream at the root of your demo system repository.

Alternatively, you can build your own bitstream if you have access to Vivado by following the instructions in [the README](https://github.com/lowRISC/ibex-demo-system/blob/main/README.md).

## Programming the FPGA
Before we can program the FPGA, we need to make it accessible from inside to the container.
First, lets find out which bus and device our Arty is on:
```console
$ lsusb
...
Bus 00X Device 00Y: ID 0403:6010 Future Technology Devices International, Ltd FT2232C/D/H Dual UART/FIFO IC
...
```
Where X and Y are numbers. Please note down what X and Y is for you (this will change if you unplug and replug your FPGA).

Then exit your docker and run it with the following parameters (assuming you're running Linux):
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

Then inside the container at [localhost:6080/vnc.html](http://localhost:6080/vnc.html), we program the FPGA with the following terminal command:
```bash
openFPGALoader -b arty_a7_35t \
    /home/dev/demo/lowrisc_ibex_demo_system_0.bit
```

## Loading the software
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

## Interacting with the UART
Besides the LEDs, the demo application also prints to the UART serial output. You can see this output using the following command:
```bash
screen /dev/ttyUSB1 115200
```
If you see an immediate `[screen is terminating]`, it may mean that you need super user rights. In this case, you may try adding `sudo` before the `screen` command. To exit from the `screen` command, you should press control and a together, then release these two keys and press d.

### Exercise 1
While the demo application is running try toggling the switches and buttons, you should see changes in the input value that is displayed by `screen`.

### Exercise 2
Adjust the demo system application to print "Hello Ibex" instead of "Hello World". You should adjust the content of the file in `sw/demo/hello_world/main.c`. Afterwards, you should rebuild the software, but do not need to rebuild the bitstream. Once you've built your updated software, you can load the software onto the board to see the result.

## Debugging using GDB
We can use OpenOCD to connect to the JTAG on the FPGA. We can then use GDB to connect to OpenOCD and interact with the board as we would when we debug any other application.

First, let's load the software in the halted state:
```bash
util/load_demo_system.sh halt ./sw/build/demo/hello_world/demo
```

In a separate terminal window, you can connect GDB to the OpenOCD server:
```bash
riscv32-unknown-elf-gdb -ex "target extended-remote localhost:3333" \
    ./sw/build/demo/hello_world/demo
```

Some useful GDB commands:

- `step`: steps to the next instruction.
- `advance main`: keep running until the start of the main function (you can replace main with another function).
- `set {int}0xhex_addr = 0xhex_val`: write `hex_val` to the memory address `hex_addr`.
- `x/w 0xhex_addr`: read a word (32 bits) from `hex_addr` and display it in hexidecimal.
- `info locals`: shows you the values of the local variables for the current function.
- `backtrace`: shows you the call stack.
- `help`: to find commands you may not yet know.

### Exercise 3
Write to the green LEDs using GDB. Look in `sw/common/demo_system_regs.h` for the value of `GPIO_BASE`. Use the set command above to write `0xa0` to this base address. This should change the green LEDs to be on, off, on and off.
