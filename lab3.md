# Lab 3: Programming the Arty-7 FPGA Board

In this lab, we will:
- Program our board with a bitstream.
- Run the software on the board.
- Read from the UART.
- Interact with the Ibex using GDB.

This lab assumes that you have C programs compiled for RISC-V and have the Docker container running (both found in Lab 1.)

***WARNING: This lab is under development***


</br>


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
The Docker container is configured to automatically load the user device permissions for the FPGA board, located at `/etc/udev/rules.d/90-arty-a7.rules`. 

Run the following to reload the rules for the container: 
_If using a docker container, the following commands should be run from inside the container._

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
sudo usermod -a -G plugdev dev
```
You can ensure that the current user `dev` has been added to the plugdev group using the command:
```bash
groups dev
```


### 3. Getting the FPGA bitstream
Download the [FPGA bitstream from GitHub (v0.0.3)](https://github.com/lowRISC/ibex-demo-system/releases/download/v0.0.3/lowrisc_ibex_demo_system_0.bit). Put the bitstream at the root of your demo system repository.

Alternatively, you can build your own bitstream if you have access to Vivado by following the instructions in [the README](https://github.com/lowRISC/ibex-demo-system/blob/main/README.md). This is not needed for the current lab. 


### 4. Programming the FPGA (FAILED HERE, possibly because of USB Hub use)
Then, inside the docker container at [localhost:6080/vnc.html](http://localhost:6080/vnc.html), we program the FPGA with the following terminal command:
```bash
openFPGALoader -b arty_a7_100t /home/dev/demo/ibex-demo-system/lowrisc_ibex_demo_system_0.bit
```
If you recieve an error stating that "unable to open ftdi device", you may need to run the command with sudo:
```bash
sudo openFPGALoader -b arty_a7_100t /home/dev/demo/ibex-demo-system/lowrisc_ibex_demo_system_0.bit
```

### 5. Loading the software
Before we load the software, please check that you have OpenOCD installed:
```bash
openocd --version
```
Please also check that you have version 0.11.0.

Then you can load and run the program as follows:
```bash
util/load_demo_system.sh run ./sw/c/build/demo/hello_world/demo
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
