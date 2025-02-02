---
template: page.html
---

Computers don't execute JavaScript directly.
Instead,
each processor has its own <span g="instruction_set" i="instruction set">instruction set</span>,
and a compiler translates high-level languages into those instructions.
Compilers often use an intermediate representation called <span g="assembly_code" i="assembly code">assembly code</span>
that gives instructions human-readable names instead of numbers.
To understand more about how JavaScript actually runs
we will simulate a very simple processor with a little bit of memory.
If you want to dive deeper,
have a look at <span i="Nystrom, Bob">[Bob Nystrom's][nystrom-bob]</span> *[Crafting Interpreters][crafting-interpreters]*.
You may also enjoy <span i="Human Resource Machine">[Human Resource Machine][human-resource-machine]</span>,
which asks you to solve puzzles of increasing difficulty
using a processor almost as simple as ours.

## What is the architecture of our virtual machine? {#virtual-machine-arch}

Our <span g="virtual_machine" i="virtual machine">virtual machine</span> has three parts,
which are shown in <a figure="virtual-machine-architecture"/>
for a program made up of 110 instructions:

1.  An <span g="instruction_pointer" i="instruction pointer">instruction pointer</span> (IP)
    that holds the memory address of the next instruction to execute.
    It is automatically initialized to point at address 0,
    which is where every program must start.
    This rule is part of the <span g="abi" i="Application Binary Interface">Application Binary Interface</span> (ABI)
    for our virtual machine.

1.  Four <span g="register" i="register (in computer)">registers</span> named R0 to R4 that instructions can access directly.
    There are no memory-to-memory operations in our VM:
    everything  happens in or through registers.

1.  256 <span g="word_memory">words</span> of memory, each of which can store a single value.
    Both the program and its data live in this single block of memory;
    we chose the size 256 so that each address will fit in a single byte.

<figure id="virtual-machine-architecture">
  <img src="figures/architecture.svg" alt="Virtual machine architecture" />
  <figcaption>Architecture of the virtual machine.</figcaption>
</figure>

The instructions for our VM are 3 bytes long.
The <span g="op_code" i="op code; virtual machine!op code">op code</span> fits into one byte,
and each instruction may optionally include one or two single-byte operands.
Each operand is a register identifier,
a constant,
or an address
(which is just a constant that identifies a location in memory);
since constants have to fit in one byte,
the largest number we can represent directly is 256.
<a table="virtual-machine-op-codes"/> uses the letters `r`, `c`, and `a`
to indicate instruction format,
where `r` indicates a register identifier,
`c` indicates a constant,
and `a` indicates an address.

<div class="table" id="virtual-machine-op-codes" cap="Virtual machine op codes.">
| Instruction | Code | Format | Action              | Example      | Equivalent                |
| ----------- | ---- | ------ | ------------------- | ------------ | ------------------------- |
|  `hlt`      |    1 | `--`   | Halt program        | `hlt`        | `process.exit(0)`         |
|  `ldc`      |    2 | `rc`   | Load immediate      | `ldc R0 123` | `R0 := 123`               |
|  `ldr`      |    3 | `rr`   | Load register       | `ldr R0 R1`  | `R0 := RAM[R1]`           |
|  `cpy`      |    4 | `rr`   | Copy register       | `cpy R0 R1`  | `R0 := R1`                |
|  `str`      |    5 | `rr`   | Store register      | `str R0 R1`  | `RAM[R1] := R0`           |
|  `add`      |    6 | `rr`   | Add                 | `add R0 R1`  | `R0 := R0 + R1`           |
|  `sub`      |    7 | `rr`   | Subtract            | `sub R0 R1`  | `R0 := R0 - R1`           |
|  `beq`      |    8 | `ra`   | Branch if equal     | `beq R0 123` | `if (R0 === 0) PC := 123` |
|  `bne`      |    9 | `ra`   | Branch if not equal | `bne R0 123` | `if (R0 !== 0) PC := 123` |
|  `prr`      |   10 | `r-`   | Print register      | `prr R0`     | `console.log(R0)`         |
|  `prm`      |   11 | `r-`   | Print memory        | `prm R0`     | `console.log(RAM[R0])`    |
</div>

We put our VM's architectural details in a file
that can be shared by other components:

<div class="include" file="architecture.js" />

<!-- continue -->
While there isn't a name for this design pattern,
putting all the constants that define a system in one file
instead of scattering them across multiple files
makes them easier to find as well as ensuring consistency.

## How can we execute these instructions? {#virtual-machine-execute}

As in previous chapters,
we will split a class that would normally be written in one piece into several parts for exposition.
We start by defining a class with an instruction pointer, some registers, and some memory
along with a prompt for output:

<div class="include" file="vm-base.js" omit="skip" />

A program is just an array of numbers representing instructions.
To load one,
we copy those numbers into memory and reset the instruction pointer and registers:

<div class="include" file="vm-base.js" keep="initialize" />

In order to handle the next instruction,
the VM gets the value in memory that the instruction pointer currently refers to
and moves the instruction pointer on by one address.
It then uses <span g="bitwise_operation" i="bitwise operation">bitwise operations</span>
to extract the op code and operands from the instruction
(<a figure="virtual-machine-unpacking"/>):

<div class="include" file="vm-base.js" keep="fetch" />

<figure id="virtual-machine-unpacking">
  <img src="figures/unpacking.svg" alt="Unpacking instructions" />
  <figcaption>Using bitwise operations to unpack instructions.</figcaption>
</figure>

> ### Semi-realistic
>
> We always unpack two operands regardless of whether the instructions has them or not,
> since this is what a hardware implementation would be.
> We have also included assertions in our VM
> to simulate the way that real hardware includes logic
> to detect illegal instructions and out-of-bound memory addresses.

The next step is to extend our base class with one that has a `run` method.
As its name suggests,
this runs the program by fetching instructions and executing them until told to stop:

<div class="include" file="vm.js" omit="skip" />

Some instructions are very similar to others,
so we will only look at three here.
The first stores the value of one register in the address held by another register:

<div class="include" file="vm.js" keep="op_str" />

<!-- continue -->
The first three lines check that the operation is legal;
the fourth one uses the value in one register as an address,
which is why it has nested array indexing.

Adding the value in one register to the value in another register is simpler:

<div class="include" file="vm.js" keep="op_add" />

<!-- continue -->
as is jumping to a fixed address if the value in a register is zero:

<div class="include" file="vm.js" keep="op_beq" />

## What do assembly programs look like? {#virtual-machine-assembly}

We could figure out numerical op codes by hand,
and in fact that's what [the first programmers][eniac-programmers] did.
However,
it is much easier to use an <span g="assembler" i="assembler">assembler</span>,
which is just a small compiler for a language that very closely represents actual machine instructions.

Each command in our assembly languages matches an instruction in the VM.
Here's an assembly language program to print the value stored in R1 and then halt:

<div class="include" file="print-r1.as" />

<!-- continue -->
Its numeric representation is:

<div class="include" file="print-r1.mx" />

One thing the assembly language has that the instruction set doesn't
is <span g="label_address" i="label (on address)">labels on addresses</span>.
The label `loop` doesn't take up any space;
instead,
it tells the assembler to give the address of the next instruction a name
so that we can refer to that address as `@loop` in jump instructions.
For example,
this program prints the numbers from 0 to 2
(<a figure="virtual-machine-count-up"/>):

<div class="include" pat="count-up.*" fill="as mx" />

<figure id="virtual-machine-count-up">
  <img src="figures/count-up.svg" alt="Counting from 0 to 2" />
  <figcaption>Flowchart of assembly language program to count up from 0 to 2.</figcaption>
</figure>

Let's trace this program's execution
(<a figure="virtual-machine-trace-counter"/>):

1.  R0 holds the current loop index.
1.  R1 holds the loop's upper bound (in this case 3).
1.  The loop prints the value of R0 (one instruction).
1.  The program adds 1 to R0.
    This takes two instructions because we can only add register-to-register.
1.  It checks to see if we should loop again,
    which takes three instructions.
1.  If the program *doesn't* jump back, it halts.

<figure id="virtual-machine-trace-counter">
  <img src="figures/trace-counter.svg" alt="Trace counting program" />
  <figcaption>Tracing registers and memory values for a simple counting program.</figcaption>
</figure>

The implementation of the assembler mirrors the simplicity of assembly language.
The main method gets interesting lines,
finds the addresses of labels,
and turns each remaining line into an instruction:

<div class="include" file="assembler.js" keep="assemble" />

To find labels,
we go through the lines one by one
and either save the label *or* increment the current address
(because labels don't take up space):

<div class="include" file="assembler.js" keep="find-labels" />

To compile a single instruction we break the line into tokens,
look up the format for the operands,
and pack them into a single value:

<div class="include" file="assembler.js" keep="compile" />

Combining op codes and operands into a single value
is the reverse of the unpacking done by the virtual machine:

<div class="include" file="assembler.js" keep="combine" />

Finally, we need few utility functions:

<div class="include" file="assembler.js" keep="utilities" />

Let's try assembling a program and display its output,
the registers,
and the interesting contents of memory.
As a test,
this program counts up to three:

<div class="include" file="count-up.as" />
<div class="include" file="count-up-out.out" />

## How can we store data? {#virtual-machine-data}

It is tedious to write interesting programs when each value needs a unique name.
We can do a lot more once we have collections like <span i="array!implementation of">arrays</span>,
so let's add those to our assembler.
We don't have to make any changes to the virtual machine,
which doesn't care if we think of a bunch of numbers as individuals or elements of an array,
but we do need a way to create arrays and refer to them.

We will allocate storage for arrays at the end of the program
by using `.data` on a line of its own to mark the start of the data section
and then `label: number` to give a region a name and allocate some storage space
(<a figure="virtual-machine-storage-allocation"/>).

<figure id="virtual-machine-storage-allocation">
  <img src="figures/storage-allocation.svg" alt="Storage allocation" />
  <figcaption>Allocating storage for arrays in the virtual machine.</figcaption>
</figure>

This enhancement only requires a few changes to the assembler.
First,
we need to split the lines into instructions and data allocations:

<div class="include" file="allocate-data.js" keep="assemble" />

<div class="include" file="allocate-data.js" keep="split-allocations" />

Second,
we need to figure out where each allocation lies and create a label accordingly:

<div class="include" file="allocate-data.js" keep="add-allocations" />

And that's it:
no other changes are needed to either compilation or execution.
To test it,
let's fill an array with the numbers from 0 to 3:

<div class="include" file="fill-array.as" />
<div class="include" file="fill-array-out.out" />

> ### How does it actually work?
>
> Our VM is just another program.
> If you'd like to know what happens when instructions finally meet hardware,
> and how electrical circuits are able to do arithmetic,
> make decisions,
> and talk to the world,
> <cite>Patterson2017</cite> has everything you want to know and more.

## Exercises {#virtual-machine-exercises}

### Swapping values {.exercise}

Write an assembly language program that swaps the values in R1 and R2
without affecting the values in other registers.

### Reversing an array {.exercise}

Write an assembly language program that starts with:

-   the base address of an array in one word
-   the length of the array N in the next word
-   N values immediately thereafter

<!-- continue -->
and reverses the array in place.

### Increment and decrement {.exercise}

1.  Add instructions `inc` and `dec` that add one to the value of a register
    and subtract one from the value of a register respectively.

2.  Rewrite the examples to use these instructions.
    How much shorter do they make the programs?
    How much easier to read?

### Using long addresses {.exercise}

1.  Modify the virtual machine so that the `ldr` and `str` instructions
    contain 16-bit addresses rather than 8-bit addresses
    and increase the virtual machine's memory to 64K words to match.

2.  How does this complicate instruction interpretation?

### Operating on strings {.exercise}

The C programming language stored character strings as non-zero bytes terminated by a byte containing zero.

1.  Write a program that starts with the base address of a string in R1
    and finishes with the length of the string (not including the terminator) in the same register.

2.  Write a program that starts with the base address of a string in R1
    and the base address of some other block of memory in R2
    and copies the string to that new location (including the terminator).

3.  What happens in each case if the terminator is missing?

### Call and return {.exercise}

1.  Add another register to the virtual machine called SP (for "stack pointer")
    that is automatically initialized to the *last* address in memory.

2.  Add an instruction `psh` (short for "push") that copies a value from a register
    to the address stored in SP and then subtracts one from SP.

3.  Add an instruction `pop` (short for "pop") that adds one to SP
    and then copies a value from that address into a register.

4.  Using these instructions,
    write a subroutine that evaluates `2x+1` for every value in an array.

### Disassembling instructions {.exercise}

A <span g="disassembler">disassembler</span> turns machine instructions into assembly code.
Write a disassembler for the instruction set used by our virtual machine.
(Since the labels for addresses are not stored in machine instructions,
disassemblers typically generate labels like `@L001` and `@L002`.)

### Linking multiple files {.exercise}

1.  Modify the assembler to handle `.include filename` directives.

2.  What does your modified assembler do about duplicate label names?
    How does it prevent infinite includes
    (i.e., `A.as` includes `B.as` which includes `A.as` again)?

### Providing system calls {.exercise}

Modify the virtual machine so that developers can add "system calls" to it.

1.  On startup,
    the virtual machine loads an array of functions defined in a file called `syscalls.js`.

2.  The `sys` instruction takes a one-byte constant argument.
    It looks up the corresponding function and calls it with the values of R0-R3 as parameters
    and places the result in R0.

### Unit testing {.exercise}

1.  Write unit tests for the assembler.

2.  Once they are working,
    write unit tests for the virtual machine.
