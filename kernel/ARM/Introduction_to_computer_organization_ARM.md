Introduce the concepts of how computer hardware works from a program's point of view.
A programmer's job is to design a sequence of instructions that will cause the hardware
to perform operations to solve a problem.

* Understanding how the hardware works helps you to become a better programmer.
* Learning is easier if it builds upon concepts your already know.
* "Real world" hardware and software make a more interesting platform for learning theoretical concepts.
* The tools used for teaching should be inexpensive and readily avaiable.

* Assembly language is more efficient. This does not always hold.
* Assembly language is required in some situations.

* How the interger stored in memory?
* How the computer instructed to implement an if-else construct.
* What happens when one function calls another function.
* How does computer know how to return to the statement following the function call statement.


BUS:
A program consists of a sequence of instructions that is stored in memory. When the CPU is
ready to execute the next instruction in the progarm, the location of that instruction in
memory is placed on the address bus. The CPU also places a "read" signal on the control bus.
The memory subsystem responds by replacing the instruction on the data bus, where the CPU
can then read it.

If the CPU is instructed to store data in memory, it places the data on the data bus, places
the location in memory where the data is to be stored on the address bus, and places a "write"
signal on the control bus. The memory subsystem responds by copying the data on the data bus
into the specified memory location.

## Condition Codes
EQ	Equal							Z == 1
NE	Not Equal						Z == 0
CS	Carry Set	Greater than, equal, or unordered	C == 1
CC	Carry Clear	Less than				C == 0
MI	Negative	Less than				N == 1
...
LS	Unsgined lower or same	Less than or equal		C == 0 OR Z == 0
GE	Signed greater or equal	Greater than or equal		N == V
LT	Signed less than					N != V
GT	Signed greater than					Z == 0 AND N == V
LE	Signed less or equal					Z == 1 or N != V


## Shift Options

LSL #<n>	LSL <Rs>	Logical shift left n bits. 1 =< n <= 31.
LSR #<n>	LSR <Rs>	Logical shift right n bits. 
ASR #<n>	ASR <Rs>	Arithmetic shift right n bits. 
ROR #<n>	ROR <Rs>	Rotate right n bits. 
RRX

Logical shift: Zeroes are written into the vacated bit positions.
Arithmetic shift right: The value in the high-order bit, 0 or 1, is copied into each of the
vacated bit positions, thus preserving the sign of the value being shifted.
Rotate right: As the low-order bits spill off the right side (low-order positions), they are
rotated around to flow into the high-order positions as the value is rotated.

I.E. 

mov     r0, 12
mov     r1, 60
add     r2, r0, r1, lsl 2		# r2 = r0 + r1 << 2

BX: Branches to another location in the program. The address of that location is in a register.
LDR: Loads a word from memory into a register.
	LDR<c>  <Rt>, <label>                  % Label
	LDR<c>  <Rt>, [<Rn>{, #+/-<imm>}]      % Offset
	LDR<c>  <Rt>, [<Rn>, #+/-<imm>]!       % Pre-indexed
	LDR<c>  <Rt>, [<Rn>], #+/-<imm>        % Post-indexed
	* <Rt> is the destination register, and <Rn> is the base register.
	* <label> is a labeled memory address.
	* <imm> is a signed integer in the range −2048…+2047.

	* In the Pre-indexed form, the signed integer is added to the value in the base register, <Rn>,
	  the base register is updated to the new address, and then the value at this new address is loaded into <Rt>.
	* In the Post-indexed form, the value in the base register, <Rn>, is used as an address, and the
	  value at that address is loaded into <Rt>. Then the signed integer is added to the value in the base register.

STR:	Stores a word from a register into memory.
	STR<c>  <Rt>, <label>                  % Label
	STR<c>  <Rt>, [<Rn>{, #+/-<imm>}]      % Offset
	STR<c>  <Rt>, [<Rn>, #+/-<imm>]!       % Pre-indexed
	STR<c>  <Rt>, [<Rn>], #+/-<imm>        % Post-indexed

I.E.
str     fp, [sp, -4]!
This instruction first determines a memory address by subtracting 4 from the address in the sp register
and updating the sp register to this new address. It then stores the address in the fp register in memory at this new address.

ldr     fp, [sp], 4
Load the address in the sp to the fp, and then adding 4 to the sp.


r0		N	argument/results
r1		N	argument/results
r2		N	argument/results
r3		N	argument/results
r4		Y	local variable
r5		Y	local variable
r6		Y	local variable
r7		Y	local variable
r8		Y	local variable
r9		Y	depends on platform standard
r10		Y	local variable
r11	fp	Y	frame pointer/local variable
r12	ip	N	intra-procedure-call scratch
r13	sp	Y	stack pointer
r14	lr	N	link register
r15	pc	N	program counter


ARM 32:	push and pop are not included in the ARM64.
push    {fp, lr}	#push lr, fp into the stack.
pop     {fp, pc}	#pop fp, pc from the stack.

ARM64:
LDMDB	<Rn>{!}, <registers>
STMDB	<Rn>{!}, <registers>

BL <label>	#The address of the instruction immediately following this bl instruction is
loaded into the lr register, and the address of <label> is moved to the pc, thus causing program 
execution to branch to that location.


The protocol specification, Procedure Call Standard for the ARM Architecture:
* The stack pointer must always be aligned to a word (4 bytes) boundary.
* The stack pointer must be double-word (8 bytes) aligned at a “public interface.”

* The stack pointer must always be aligned on a 16-byte boundary in AARCH64 mode.

main:
        stmfd   sp!, {fp, lr}   @ save caller's info
        add     fp, sp, 4       @ set up our frame pointer
        add     sp, sp, locals  @ allocate memory for local var

![](Introduction_to_computer_organization_ARM.assets/1.svg)

	add     sp, sp, local   @ deallocate local var
	ldmfd   sp!, {fp, lr}   @ restore caller's info
	bx      lr              @ return

B <label>	#The address corresponding to the label is moved to the pc, thus causing program execution to branch to that location.
bne	else

BL <label>	#update the lr register.


CMP		#Does an arithmetic comparison of two values and sets condition flags according to the result.
TST		#Does a logical comparison of two values and sets condition flags according to the result.


**In ARM state, the value of the PC is the address of the current instruction plus 8 bytes.**

In Thumb state:

**For B, BL, CBNZ, and CBZ instructions, the value of the PC is the address of the current instruction plus 4 bytes.**

**For all other instructions that use labels, the value of the PC is the address of the current instruction plus
4 bytes, with bit[1] of the result cleared to 0 to make it word-aligned.**
