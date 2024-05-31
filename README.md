# Open RISC-V Silicon Development with Ibex

List of labs for the Ibex Demo System:
- [Lab 1: Programming your FPGA and interacting with the demo system](./lab1.md)
- [Lab 2: Simulating with Verilator+GTKWave and modifying a HW peripheral](./lab2.md)
- [Lab 3: Using the LCD Display over SPI](./lab3.md)
- [Lab 4: Adding custom instructions to Ibex](./lab4.md)

This content is copyright lowRISC and licensed under the Creative Commons Attribution 4.0 International Public License, see LICENSE for details.

# Lab Preamble

In this tutorial, lowRISC shows how to work with a minimal SoC built around Ibex (the Ibex Demo System, https://github.com/lowRISC/ibex-demo-system). Ibex is a mature RISC-V (RV32IMCB) core which has seen several tape outs across academia and industry. lowRISC helps maintain Ibex as part of the OpenTitan project. The SoC is fully open source and designed to serve as a starting point for embedded computer architecture research, development, and teaching. The SoC combines Ibex with a number of useful peripherals (such as UART, Timer, PWM and SPI) and a full-featured debug interface.

The SoC has been intentionally kept simple, so people new to hardware development and/or SoC design can quickly adapt it to their needs. The labs will use the affordable (300$) Digilent Arty A7 FPGA board. A ‘simulation first’ approach is taken (using the open source Verilator simulator) to get RTL designed and working before FPGA synthesis (which is done with Xilinx Vivado). This gives a quicker, less frustrating, develop/build/debug loop.

[lowRISC](https://lowrisc.org/) is a not-for-profit company with a full stack engineering team based in Cambridge, UK. We use collaborative engineering to develop and maintain open source silicon designs and tools. Ibex development is funded by the OpenTitan project.

This content is copyright lowRISC and licensed under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/), see LICENSE for details.
