# Lab 2b: Conducting a Stack Buffer Overflow Attack on Ibex
<!-- Please write one sentence per line, as this facilitates version control. -->
<!-- Put sample solution in comment below a question. -->

In this lab, you will modify and compile a C program that exploits a buffer overflow vulnerability in `strcpy()` to perform a code reuse attack. 

The directory `ibex-development/sw/c/tests/bof-attack` contains a source file in which you will conduct the explloit. The function `vulnerable()` calls `strcpy()` without verifying the size of the source or destination strings. The program is configured to pass the string variable `attack_payload` to the vulnerable function. Your task is to modify `attack_payload` with a value that will successfully overflow the stack buffer to redirect program execution to `target()`. 

To build the source file `main.c`, run the following command from `sw/c/build`:
```
cmake ..
make
```

To disassemble the resulting ELF file `bof` for inspection, run the following command from `sw/c/build/tests/bof-attack`:
```bash
riscv32-unknown-elf-objdump -d bof > bof.dump
```

You will need to understand the stack layout of the vulnerable function in order to successfully conduct the attack!