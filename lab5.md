# Lab 5: Adding Custom Instructions to Ibex

In Lab 4 we saw the difference in performance between the fixed and floating point Mandelbrot implementations, can we get further performance improvements?
By adding custom RISC-V instructions, we can create dedicated hardware for computing the Mandelbrot set even more efficiently.
This lab covers the process of adding such custom instructions, though before we discuss how to do that, let's first take a quick look at how you calculate a Mandelbrot set.

## Drawing the Mandelbrot set

This is not a mathematics lab, so we're not going into lots of detail here nor does it matter if you don't really understand it. We're just going to cover the 'nuts and bolts' of the calculation you need to work with, the lowest-level math operations that dominate the runtime of the algorithm.
Before we can draw the Mandelbrot set, we first need to be able to work with complex numbers.

### Complex Number Intro

A complex number can be written as $a + bi$, where $a$ and $b$ are real numbers and $i = \sqrt{-1}$ is the *imaginary unit*. (We call it imaginary because there is no physical way to visualize what $\sqrt{-1}$ is!)
$a$ is called the *real part*, and $b$ is called the *imaginary part*.

To add complex numbers, simply sum the real and imaginary parts:

$$(a + bi) + (c + di) = (a + c) + (b + d)i$$

To mutiply complex numbers, the components are multiplied by standard algebraic rules (where multiplication distributes over addition):

$$(a + bi) (c + di) = ac + adi + bci + bdi^2 = (ac - bd) + (ad + bc)i$$

Noting that $i^2 = (\sqrt{-1})^2 = -1$.

Finally, the absolute value $|C|$ of a complex number $C = a + bi$ (you can also think this as the magnitude of a 2D vector in the real and imaginary axis) is defined as:

$$|C|^2 = a^2 + b^2$$

Noting we would need a square root to get the absolute value, but for our purposes the squared value is fine.

### Mandelbrot Set Calculation

The Mandelbrot set is a set of complex numbers.
A number $C$ is in the set if the recurrance relationship below never diverges to infinity (i.e., $|Z_n|$ is a finite number for all $n$).

$$Z_0 = C$$

$$Z_{n+1} = Z_n^2 + C$$

It can be shown that if $|Z_n| > 2$ for any $n$, then the recurrence will diverge and hence $C$ is not part of the Mandelbrot set.

To draw the set, we map pixels to real and imaginary numbers and calculate a certain number of iterations of the $Z_n$ recurrence for each point to decide if the point is in the set:

- If $|Z_n| \leq 2$ (noting we can just test the square result against 4 to avoid a square root) for all iterations, we declare the point *in* the set.
- If $|Z_n| > 2$ for any iteration, we terminate the iterations there and declare the point *not* in the set.

We finally colour the result based upon the number of iterations we reached before making our decision.

## Complex number custom instructions

Note: These instructions are tailored to this lab, and a 'real' complex number extension may do things differently, in particular the clamping and truncation behaviour discussed below.

Our fixed point implementation of the Mandelbrot set renderer uses 16-bit numbers (12 fractional bits, 4 integer bits with 2s complement representation).
This means we can pack a complex number into a single 32-bit number.
The imaginary part will be placed in the lower bits (indexed `[15:0]` and the real part in upper bits (`[31:16]`).
So how about some custom instructions that implement complex number operations on the packed 32 bit representation?

We are going to want three new instructions:

* complex multiplication
* complex addition
* complex absolute value (squared)

These will fit into the same instruction type used by the the ALU operations (add, sub, xor etc).
All three have one destination register, multiplication and addition have two source registers whereas absolute value only has one.

### Choosing a RISC-V instruction encoding

We won't be getting into full encoding details in this lab, but just covering the ones we need (please consult the RISC-V ISA manual volume 1 if you want more details).
The bottom 7 bits of a RISC-V instruction specify the 'major opcode' (in this table, the bottom two bits are always `2'b11`, so the table only shows bits 2:6):

![](./lab5_imgs/RISC-V_base_opcode_map.png)

RISC-V reserves some major opcodes for custom instructions; we're going to use the *custom-0* opcode which has a value of `7'b0001011 = 7'h0b`.
Let's call it `OPCODE_CMPLX`.

All of our instructions will be *R-type* instructions, which have the following layout:

![](./lab5_imgs/R-type_instruction_encoding.png)

The R-type instructions provide two source registers and one destination register.
We'll use the *funct3* field to select which of our operations to execute:

- `3'b000` - Complex Multiplication
- `3'b001` - Complex Addition
- `3'b010` - Complex Absolute Value (Squared)

*funct7* will be set to 0 in all cases.
For the absolute value operation, the *rs2* source register will always be `x0` (and ignored by the instruction).

## Adding custom instructions to Ibex

Ibex has no explicit custom instruction support, however one can easily alter the RTL to add some. There are three source files in which we will need to make modifications.

- `ibex_pkg.sv`: Constants and definitions
- `ibex_decoder.sv`: Instruction decoder
- `ibex_alu.sv`: Arithmetic-logical unit (ALU)

### Modifying `ibex_pkg.sv`

Open the file `vendor/lowrisc_ibex/rtl/ibex_pkg.sv`, and make the following changes:

1. Add our new opcode (`OPCODE_CMPLX = 7'h0b`) to the opcode enum `opcode_e`.
2. Add three new operations, one for each new instruction, to the ALU operations enum `alu_op_e`.
   This is what is produced by the decoder to tell the ALU what to do.
   Name them whatever you think is best.

### Modifying `ibex_decoder.sv`

Open `vendor/lowrisc_ibex/rtl/ibex_decoder.sv` and take a look.
The first thing to note is the decoder is split into two. One part specifies things like register read and write enables, and the other part specifies ALU related signals.
The decoder also has two copies of the instruction in seperate flops, which allows us to meet the timing constraints more easily. With a single set of flops, the 'fan-out' of those flops becomes very large and requires significant buffering (drive-strength), slowing the logic down. With the duplicate flops and split, the decoder fan-out is reduced, which improves performance.
Tools can do this kind of duplication automatically, but it may not be enabled in all flows (in particular in ASIC synthesis), and tools may choose a split that doesn't work as well. Hence we explicity split the logic ourselves to guarantee the behaviour we want.

Support for our custom opcode `OPCODE_CMPLX` needs to be added to both decoders:

1. The first decoder begins with `unique case (opcode)`.
   For `OPCODE_CMPLX` we must set the following signals:
  - `rf_ren_a_o`/`rf_ren_b_o`: Register file read enables
  - `rf_we`: Register file write enable
  - `illegal_insn`: set 1 if the instruction is illegal (e.g. *funct3* isn't one of the 3 values we are using)
2. The second decoder controls the ALU operation and begins with `unique case (opcode_alu)`.
   We must set the following signals:
  - `alu_op_a_mux_sel_o`: Mux select for ALU operand A, always set to `OP_A_REG_A` as we always read our operands from registers for our new instructions.
  - `alu_op_b_mux_sel_o`: Mux select for ALU operand B, always set to `OP_B_REG_B` as we always read our operands from registers for our new instructions.
  - `alu_operator_o`: The ALU operation, set it to one of the new values you created in `ibex_pkg.sv` depending upon the *funct3* field of the instruction.

### Modifying `ibex_alu.sv`

Finally open the file `vendor/lowrisc_ibex/rtl/ibex_alu.sv` and take a look.
The ALU takes in two operands (inputs `operand_a_i` and `operand_b_i`) along with an operator (input `operator_i`) to produce a result (output `result_o`).
There are multiple parts to the ALU to handle different kinds of operations (the bit-manipulation extension for example adds significant complexity here).
At the bottom of the file you will find the result multiplexer (mux), which outputs the result from the appropriate part of the ALU based upon the operator.

We need to add new logic to implement our complex fixed point operations, but first let's look at fixed point representation and clamping/truncating behaviour.

#### Fixed point representation

Fixed point representation simply stores and operates on a scaled version of the true value. The true number is always multiplied by some constant, which is a power of two, so we can recover it by applying the reverse, a division by the constant. For the representation we're using, we choose that multiplier to be 4096 or $2^12$, so 1.0 would be represented by 4096, 2.0 by 8192, and 1.5 by 6144. This may seem slightly odd, but it allows us to make use of common ALU hardware extremely efficiently.

Manipulating fixed point numbers is as straightforward as integers:
For addition and subtraction simply perform the operations as normal:

$$xF + yF = (x + y)F$$

where $F$ is our fixed multiplier 4096.
For multiplication first multiply, then divide by the fixed constant:

$$xF yF = xyF^2$$

...so we need to divide by $F$ to get the $xyF$ result we want.

As the constant $F$ is a power of two, the division is a bitshift, or in practice is just choosing which bits out of your multiplier are fed into the final result.

#### Clamping/Truncation

Before we continue, there is an important issue with multiplication, and to a lesser extent addition.
Multiplying two 16-bit numbers can give you a 32-bit result (consider `0xffff` multiplied by `0xffff`).
We drop the bottom 12 bits of this result to give us our constant divide, but that still leaves us with 20 bits of result and only 16 bits to place it in.
There are two options:

 * Truncation: drop the top four bits;
 * Clamping: clamp the result to the maximum or minimum value as appropriate.

For complex multiplication we'll use truncation, for two main reasons:
Firstly, neither truncation nor clamping produce sensible overall results. The result of complex multiplication is the sum of multiple real multiplications, so clamping individual multiplications does not necessarily result in a sensible result overall (e.g., they could have different signs and cancel each other out). Truncation also produces a non-sensical overall result, but it's cheaper to implement and matches what normal multiplications do (e.g., in C if we do a 32-bit multiplication, we get a truncated 32-bit result).
Secondly, we can just move the responsibility for providing sane inputs to the application, by ensuring that any multiplications remain within range (which is the case for our Mandelbrot set application).

For the complex absolute value we'll use clamping, again for two main reasons:
Firstly, squaring means no negative results, so clamping is sensible. If one of the multiplications maxes out, we'll get a large result back because the other multiplication can't produce some large negative that could cancel it out.
Secondly, our Mandelbrot set application uses the absolute value to break multiplication iterations when exceeding a threshold, thereby keeping multiplications within range.
This would not be possible if we used truncation in calculating the absolute value.

Addition suffers from the same issue in that an extra bit (representing the carry-out) is added, so our 16-bit adds have 17 bits of result.
For complex additions and for the additions to implement the complex multiplication, we will use truncation, with the same reasoning as for complex multiplication.
For the addition at the end of the complex absolute value, we use the full 17-bit value of the sum and sign-extend it to 32 bits.

#### Implementing the complex operations

It's time to implement our new logic.
Here's an outline for how you may want to implement it:

```systemverilog
logic [15:0] rs1_real, rs1_imag;
logic [15:0] rs2_real, rs2_imag;

assign rs1_real = operand_a_i[31:16];
assign rs1_imag = operand_a_i[15:0];
assign rs2_real = operand_b_i[31:16];
assign rs2_imag = operand_b_i[15:0];

logic [15:0] mul1_res;

// Multipliers for complex multiplication
fp_mul#(.CLAMP(0)) mul1(.a_i(rs1_real), .b_i(rs2_real), .result_o(mul1_res));
// TODO: Add more multipliers here

logic [15:0] real_sq_res;

// Multpliers for complex absolute value
fp_mul#(.CLAMP(1)) real_sq(.a_i(rs1_real), .b_i(rs1_real), .result_o(real_sq_res));
// TODO: Add another multiplier here

logic [31:0] cmplx_result;

always_comb begin
  cmplx_result = '0;

  unique case (operator_i)
    ALU_CMPLX_MUL: begin
      // TODO Implement this write result to `cmplx_result`
    end
    ALU_CMPLX_ADD: begin
      // TODO Implement this write result to `cmplx_result`
    end
    ALU_CMPLX_ABS_SQ: begin
      // TODO Implement this write result to `cmplx_result`
    end
    default: ;
  endcase
end
```

The `fp_mul` module can be found in the supplementary material that accompanies this lab.
Copy the `fp_mul.sv` file to `vendor/lowrisc_ibex/rtl` in the demo system repository and add it to the list of Ibex RTL files in `vendor/lowrisc_ibex/ibex_core.core`.

Finally you will need to modify the ALU result mux to produce the result of the complex operation when one of the complex operands is selected.

## Testing our implementation

With our new instructions implemented, how do we test them?
We could just switch out the functions that do complex number manipulation in our Mandelbrot demo to use our new instructions, but if it doesn't work first time, it'll be hard to debug.
So instead we will use a dedicated program to test the instructions against existing software implementations.
You can find this in the `lab5_supplementary_material/cmplx_test` directory.
Follow these steps to build it:

1. Copy the `cmplx_test` directory into the `sw/demo/` directory in the demo system repository
2. Add `add_subdirectory(cmplx_test)` on its own line in `sw/demo/CMakeLists.txt`
3. Switch to the `sw/build` directory (or whatever build directory you chose to use) and run `cmake ../ -DSIM_CTRL_OUTPUT=On`
4. Build the software with `make`

Note the `-DSIM_CTRL_OUTPUT=On`.
This redirects output from the UART to the simulator control which writes them to a log file.
With this enabled you can see output from the program (in particular the test results) when running it through Verilator.
Note that you need to rerun the `cmake` command with that option set to `Off` and rebuild the software to use it on FPGA.

Run the `cmplx_test` binary you just built (path `sw/build/demo/cmplx_test/cmplx_test`).
Look at the `ibex_demo_system.log` file that will be in the same directory you ran the simulation from.
If you got your implementation correct, you will see "All tests passed".
Otherwise a failure will be reported, and you will be told what test failed and given the input and output of the software version and what your instruction implementation did. It's time to open GTKWave and start debugging!

## Implementing Mandelbrot with our custom instructions

Make a copy of the `fractal_fixed.c` file in `sw/demo/lcd_st7735`, e.g. calling it `fractal_cmplx_insn.c`.
Modify `fractal_cmplx_insn.c` to use the new custom instructions.
You can find functions that use the custom instructions in `cmplx_test/main.c`.
We suggest just copying these functions into your `fractal_cmplx_insn.c` and using those rather than taking the inline assembly within them and using that directly.
Don't forget to add `fractal_cmplx_insn.c` to the list of source files for `lcd_st7735` in `sw/demo/lcd_st7735/CMakeLists.txt`.

A few modifications will be needed to switch to using the custom instructions:

 * Remove all the fixed point and complex number functions (though you'll want to keep the macros)
 * Write a function that can take a `cmplx_fixed_t` and pack it into a 32-bit integer for use with the custom instructions.
 * Alter `mandel_iters_fixed` to use the complex number instructions.
 * Rename `mandel_iters_fixed` e.g. to `mandel_iters_cmplx_insn`
 * Update `fractal_mandelbot_fixed` similarly.
 * Call your new mandelbrot function from `main.c` as part of the fractal drawing, giving a final result that first uses the floating point implementation, then the fixed point implementation, and finally the implementation with instructions.

## Further Exercises / Questions

 1. Our new instructions add several new multipliers to Ibex, can we reduce this by sharing them?
    Can we share with the existing multipliers (it currently has 3 16x16 ones to implement the single-cycle multiplication)?
    Is there any point in doing this on an FPGA where DSPs are fixed function so our multiplier usage isn't consuming general logic?
    Would it be more worthwhile on ASIC?
 2. Our use of a major opcode for our custom instructions is simple but very wasteful of encoding space.
    Could we use an existing major opcode (e.g. `OPCODE_OP` which the bitmanip extension uses along with the base RV32I instructions)?
 3. What other kinds of clamping/truncating behaviour would make sense for our complex number implementation?
