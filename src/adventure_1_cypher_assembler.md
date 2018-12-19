# Cypher Assembler

In my spare time i created a virtual machine -as always- but this time i decieded to create an assembler for it since it's much easier than creating a full programming language and here's my report.

## Virtual Machine

I've witnessed many programming being scared by the term virtual machine but in reality a virtual machine is a piece of code where it can read instructions in some form and execute some code in response to each instruction so a minimal virtual machine would look like this

```C++
//this a listing of all the instructions a vm can do
enum ISA: uint8_t
{
	STOP,
	PRINT_HELLO_WORLD,
	ADD,
	...
};

//a program is just a byte buffer loaded in memory
using Program = Buf<uint8_t>;

//this struct holds the VM state
struct VM { size_t program_counter; }

//opcode decode function
//which is reponsible of decoding a single/multiple bytes from the program buffer into an ISA instruction
ISA
vm_opcode(VM& vm, Program& program)
{
	ISA opcode = ISA(program[vm.program_counter++]);
	//add all the checks for validation here
	return opcode;
}

//function which decodes and executes a single instruction and returns whether we should continue
bool
vm_execute_instruction(VM& vm, Program& program)
{
	switch(vm_opcode(vm, program))
	{
		case ISA::STOP: return false;
		case ISA::PRINT_HELLO_WORLD: printfmt("Hello, World!\n"); return true;
		case ISA::ADD: ...
		...
		default:
			assert(false && "ILLEGAL INSTRUCTION");
			return false;
	}
}

//function which runs the entire program
void
vm_run(VM& vm, Program& program)
{
	//while we should continue we will execute instructions
	while(vm_execute_instruction(vm, program));
}
```

That's it, all you need to do now is just design your instruction set and make sure the vm executes them efficiently so as you can see there's nothing to be scared of here and it's quite fun to do virtual machines.

## Sage VM

Sage VM is a general purpose register-based virtual machine which has the following registers:

- rpc: program counter
- rflg: various flags for compare, ... etc.
- rem: remainder register where the remainder of a div instruction lives
- rstk: stack pointer
- r0 ... r15: general purpose registers

All of these registers are 64-bit wide, and by default Sage VM gives each program a 1MB of stack memory.

Sage VM instructions are 32-bit wide/aligned which means they have 4 forms

1. [opcode(8-bit)]
2. [opcode(8-bit), operand(24-bit)]
3. [opcode(8-bit), operand(8-bit), operand(16-bit)]
4. [opcode(8-bit), operand(8-bit), operand(8-bit), operand(8-bit)]

## Cypher Assembler

As we have stated above our programs are just binary blobs that we decode and execute. It's inconvenient to write binary code by hand so the logical step is to write instructions in text format and create a program which converts this texture form of instructions into a binary format the VM can handle a.k.a. and Assembler.

### Simplist Program
```x86
;comments start with a semicolon
;all the code should be enclosed into procedures

;you must have at least a main proc in which the program starts
proc main
	hlt ;this stops the execution of the program
end
```
The above program creates a simple main procedure and `hlt` stops the program execution immediately.

### Mov Program
```x86
proc main
	i32 r0; declares the usage of register r0 as a signed 32-bit integer register
	mov r0 5 ;r0 = 5
	i32 r1
	mov r1 r0 ; r1 = r0
	hlt
end
```

Cypher language is a strongly typed assembler language so you have to specify the type of the register before you use it there's eight type of the registers right now

- i8: integer 8-bit
- i16: integer 16-bit
- i32: integer 32-bit
- i64: integer 64-bit
- u8: unsigned integer 8-bit
- u16: unsigned integer 16-bit
- u32: unsigned integer 32-bit
- u64: unsigned integer 64-bit

Some assembler languages bind the types of the data to the instructions for example x86 assembler language has a `div` instruction for the unsigned division and `sdiv` for signed division. But in Cypher language we bind the type of the data to the registers and the assembler will figure out the correct instruction based on the type annotations you provide like `i32 r0` which means that register `r0` is a signed 32-bit integer.

Those type annotations has no effect on the VM which means that VM registers has no encoded types at all. Type annotations are only used by the assembler to generate helpful error messages and correctly choose the right instruction.

### Add Program
```x86
proc main
	i32 r0
	i32 r1
	i32 r2
	mov r0 1
	mov r1 2
	add r0 r1 r2; r2 = r0 + r1
	add r2 r2 r2; r2 = r2 + r2
	div r2 r1 r0; r0 = r2 / r1
	i32 rem; declare the type of the remainder before using it
	i32 r3
	mov r3 rem; move the remainder to r3 register
	hlt
end
```

The Arithmetic instructions take three registers which are all registers `add operand1 operand2 destenation`

There are the four basic arithmetic instructions: `add`, `sub`, `mul`, `div`

### Sum Program
```x86
; sum function which sums the numbers from 0 to whatever the value of r0 is
proc sum
	u64 r0; first and only arg, and used as accumulator later
	u64 r1; loop limit
	u64 r2; loop counter

	mov r1 r0; set the loop limit to the r0 arg
	mov r0 0; sets the accumulator to 0
	mov r2 0; sets the loop counter to 0

; this is a label which gives the next instruction a human readable name instead of using the address
LOOP:
	ltq r2 r1; less than or equal (r2 <= r1)
	jf LOOP_EXIT; jump if the last comparison is false to exit the loop
	add r0 r2 r0; loop body (r0 = r2 + r0)
	inc r2; inc loop counter (r2++)
	jmp LOOP; jump back to the start of the loop

LOOP_EXIT:
	ret; return from the function
end

proc main
	u64 r0
	mov r0 1000000
	call sum ;simple call instruction
	hlt	;stop the program
end
```

As you can see this is a more complex program so i will provide the equavilant C program to help us understand how things map from C to the Cypher language.

```C
uint64_t sum(uint64_t n)
{
	uint64_t accumulator = 0;
	for(uint64_t i = 0; i <= n; ++i)
		accumulator += i;
	return accumulator;
}

int main()
{
	sum(1000000);
	return 0;
}
```

### Cypher Instructions Listing
- `mov`: `mov target source` moves a value from source to target, targets can only be registers and sources can be registers or immediate values
- `add`: `add operand1 operand2 destination` adds two operand registers together and puts the result into the destination register
- `sub`: `sub operand1 operand2 destination` subtracts two operand registers together and puts the result into the destination register
- `mul`: `mul operand1 operand2 destination` multiplies two operand registers together and puts the result into the destination register
- `div`: `div operand1 operand2 destination` divides two operand registers together and puts the result into the destination register
- `inc`: `inc register` increments the value in register
- `dec`: `dec register` decrements the value in register
- `eq`: `eq register1 register2` compares `1 == 2` two registers and puts the result in `rflg`
- `neq`: `neq register1 register2` compares `1 != 2` two registers and puts the result in `rflg`
- `lt`: `lt register1 register2` compares `1 < 2` two registers and puts the result in `rflg`
- `gt`: `gt register1 register2` compares `1 > 2` two registers and puts the result in `rflg`
- `ltq`: `ltq register1 register2` compares `1 <= 2` two registers and puts the result in `rflg`
- `gtq`: `gtq register1 register2` compares `1 >= 2` two registers and puts the result in `rflg`
- `gtq`: `gtq register1 register2` compares `1 >= 2` two registers and puts the result in `rflg`
- `jmp`: `jmp label/register` jumps unconditionally to the speicifed address
- `jt`: `jt label/register` jumps if `rflg` is true to the speicifed address
- `jf`: `jf label/register` jumps if `rflg` is false to the speicifed address
- `store`: `store address-register value-register` stores the value register at the address specified by the address register for now the only type of memory available is stack memory (the width/size is specified by the type of value-register)
- `load`: `load address-register target-register` loads a value from the address specified by the address register into the target register (the width/size is specified by the type of value-register)
- `push`: `push register` pushes a register onto the stack and moves the stack pointer `rstk` (the width/size is specified by the type of value-register)
- `pop`: `pop register` pops a register off the stack and moves the stack pointer `rstk` (the width/size is specified by the type of value-register)
- `call`: `call procedure-name` performs a call to the speicified procedure
- `ret`: `ret` returns from the current procedure
- `i8`: `i8 register` sets the type of this register to be signed 8-bit integer
- `i16`: `i16 register` sets the type of this register to be signed 16-bit integer
- `i32`: `i32 register` sets the type of this register to be signed 32-bit integer
- `i64`: `i64 register` sets the type of this register to be signed 64-bit integer
- `u8`: `u8 register` sets the type of this register to be unsigned 8-bit integer
- `u16`: `u16 register` sets the type of this register to be unsigned 16-bit integer
- `u32`: `u32 register` sets the type of this register to be unsigned 32-bit integer
- `u64`: `u64 register` sets the type of this register to be unsigned 64-bit integer

### Recursive Fibonacci

Just for laughs i decided to write a recursive fibonacci function in Cypher Assembler.

I was trying to encode this logic
```C
uint64_t fib(uint64_t x)
{
	if(x == 0)
		return 0;
	if(x == 1)
		return 1;
	return fib(x - 1) + fib(x - 2);
}
```

And here's the result (it works)
```x86
proc fib
	u64 r0; argument and return value

	; if(r0 == 0) return 0;
	u64 r1
	mov r1 0
	eq r0 r1; r0 == 0
	jt EARLY_EXIT_0; jump to early exit if r0 == 0

	; if(r0 == 1) return 1;
	mov r1 1
	eq r0 r1; r0 == 1
	jt EARLY_EXIT_1; jump to early exit if r0 == 1

	u64 r2
	mov r2 r0; move the argument to r2
	push r2; save the argument on the stack

	;fib(r0 - 1);
	sub r2 r1 r0; r0 = r2 - r1(1) which means r0 = r2 - 1
	call fib ;call fib(r0 - 1)
	pop r2; pop the original argument from the stack
	push r0; push the result of fib(r0 - 1) on the stack

	;fib(r0 - 2)
	mov r1 2; set r1 to 2
	sub r2 r1 r0; r0 = r2 - r1(2) which means r0 = r2 - 2
	call fib; call fib(r0 - 2)

	pop r2; pop the result of fib(r0 - 1) from the stack

	;set r0 = fib(r0 - 1) + fib(r0 - 2)
	add r0 r2 r0; r0 = r0(fib(r0-2)) + r2(fib(r0-1))
	ret;

;early exit when argument is 0
EARLY_EXIT_0:
	mov r0 0; since the fib(0) is 0
	ret

;early exit when argument is 1
EARLY_EXIT_1:
	mov r0 1; since the fib(1) is 1
	ret
end
```

### Download
Here you can download the x64 build of the cypher assembler

- [Windows](https://github.com/MoustaphaSaad/MoustaphaSaad.github.io/releases/download/v0/adventure_1_cypher_assembler_win64.zip)
- [Linux](https://github.com/MoustaphaSaad/MoustaphaSaad.github.io/releases/download/v0/adventure_1_cypher_assembler_linux64.zip)

Of Course you can invoke it with the following command `cypher run file.asm` if you want some time measurements you can add the `--verbose` flag to be `cypher run file.asm --verbose`

### Conculsion

Although being just a hobbiest work Sage VM was about 6x faster than python in the sum program N=100000 and 500 microseconds behind JS on chrome.

Virtual Machines, Compilers, Assemblers, and Programming Languages are fun to fiddle with if you are interested in this work -for example if you want to implement Sage VM in other programming languages than C++ to see the performance difference- let me know through email i can post the spec of the Sage VM and make the assembler->binary available for you.