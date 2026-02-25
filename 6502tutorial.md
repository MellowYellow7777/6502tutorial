WIP

# How to write a 6502 emulator

This is a project I keep coming back to.. Im no good at introductions so Im going to get right into it. The goal here is to implement a 6502 emulator. This is a really fun project and there are quite a few things you could learn if you havent done this sort of thing before. Now I have done this, multiple times over, maybe I didnt like something about the last time, or maybe I think I can do something better this time, but Ive gotten to the point where I can comforatably make something that at the very least, works well, so thats what Im going to show you how to do.

Im going to plan on being quite exhastive in explaining everything. There are plenty of references available online already, but Ive ran into trouble as they sometimes have conflicting information, leaving some grey area. Im going to do my best to leave as little room for interpretation as possible.

I will be using javacript mainly, but the implementation is portable to any language you like.

## Preliminary: What is the 6502?

### From the Outside

The 6502 is an 8-bit microprocessor designed in 1975 by MOS Technology. Looking at it from the outside, there are 40 pins. Heres a visual, labeled representation (< and > characters point in direction of data travel, x means the pin is bidirectional):

```
          ┌────────────────┐
    VSS   ┤1             40├ < R̅E̅S̅
    RDY > ┤2             39├ > O2
     O1 < ┤3             38├ < SO
    I̅R̅Q̅ > ┤4             37├ < O0
     NC   ┤5             36├   NC
    N̅M̅I̅ > ┤6             35├   NC
   SYNC < ┤7             34├ > R/W
    VCC   ┤8             33├ x D0
     A0 < ┤9             32├ x D1
     A1 < ┤10    6502    31├ x D2
     A2 < ┤11            30├ x D3
     A3 < ┤12            29├ x D4
     A4 < ┤13            28├ x D5
     A5 < ┤14            27├ x D6
     A6 < ┤15            26├ x D7
     A7 < ┤16            25├ > A15
     A8 < ┤17            24├ > A14
     A9 < ┤18            23├ > A13
    A10 < ┤19            22├ > A12
    A11 < ┤20            21├   VSS
          └────────────────┘
```

What is important to us:

Inputs:\
R̅E̅S̅ - reset pin\
I̅R̅Q̅ - interrupt request\
N̅M̅I̅ - nonmaskable interrupt

Outputs:\
A0-A15 - 16-bit address bus

Bidirectional:\
D0-D7 - 8-bit data bus

What we dont need to think about:\
RDY - ready pin\
SO - set overflow\
O0 - clock input\
O1-O2 - synchronize clock output (compliment pair)\
SYNC - held high during intruction-fetch cycle\
VSS - 0 to +7 volts (typically grounded)\
VCC - 0 to +7 volts (usually +5 volts)\
R/W - indicated the direction of the data bus (read/write)\
NC - no connection

What we will be doing is emulating all the stuff the chip does internally. At first we will just be assuming the chip is connected to a full 16 bit wide RAM (64kb). We will also cheat a little and simply have our 'input/output devices' manipulate and access that RAM directly, but later on we can get into memory mapped IO.

### From the Inside

#### Registers

On the chips inside, there are a handful of registers used to hold and manipulate data:

| Register | Bits | Description |
|----------|------|-------------|
| A        | 8    | Accumulator |
| X        | 8    | 'X' Index Register |
| Y        | 8    | 'Y' Index Register |
| SP       | 8    | Stack Pointer |
| PS / P   | 8    | Processor Status |
| PC       | 16   | Program Counter |

These are the main registers we will have to represent to write a working emulator. There are other registers; IR (8-bit instruction register), MAR (16-bit memory address register); components; ALU (arithmetic logic unit), ID (instruction decode); that we wont have to worry about as their puposes and behaviors will be fulfilled and arise naturally throughout writing the emulator. For example, the ALU contains circuitry to add together two binary numbers, we can just use the add ```+``` operation .

##### Accumulator

This is the main resister used for manipulating data via arithmetic and logical (bitwise) operations. For example, the instruction ```ADD 5``` would take the contents of the A register, add 5, and then store the result right back into the A register.

##### Index Registers

The X and Y registers are mainly used as memory offsets. Here's an example of how they can be used:

```asm
LDA #$FF    ; load A with value 0xFF (255)
LDX #0      ; load X with value 0
loop:
STA $40,X   ; store A at address 0x40 + X
INX         ; increment X
CPX #$20    ; compare X to value 0x20
BNE loop    ; branch if not equal
```

This program will write 0xFF to every memory address from 0x40 up to but not including 0x60. It uses 0x40 as a base address and uses X (initilized to 0, incremented up to 0x20) as an offset.

##### Stack Pointer

The stack pointer is used as a memory offset from the location 0x0100. Since the stack pointer is 8 bits, it can hold any value from 0x00 to 0xFF, making its effective range at memory addresses 0x0100 to 0x01FF. In this range, data can be pushed and pulled via stack instructions, and the stack pointer is automatically updated accordingly.

##### Processor Status

This is an 8 bit register, where each of its bits represents a flag state. See [Flags](#flags).

##### Program Counter

The program counter is actually two 8 bit registers in hardware, that together effectively store a 16 bit value. This covers the entire addressable memory range from 0x0000 to 0xFFFF. This register holds the memory address of the next instruction to execute. This is kept track of automatically, but can also be set or modified via instructions such as the `JMP` (jump) instruction.

#### Flags

Flags are stored in the [Processor Status](#processor-status) register. Several of these flags have dedicated set/clear instructions. There are also several branch instructions that conditionally jump the program depending on the state of a flag. Both of the ```PLP``` (pull processor status from stack) and ```RTI``` (return from interrupt) affect all flags as they overwrite the entire contents of the processor status.

Table:

| Bit | * | Name              | Description |
|:---:|:-:|-------------------|-------|
| 0   | C | Carry             | carry in/out of arithmetic operations |
| 1   | Z | Zero              | set if an operation result is zero |
| 2   | I | Interrupt Disable | blocks maskable interrupts |
| 3   | D | Decimal Mode      | enables decimal mode |
| 4   | B | Break             | set during a `BRK` sequence |
| 5   | U | Unused            | |
| 6   | V | Overflow          | set if an operation caused a signed overflow |
| 7   | N | Negative          | set if an operation result is negative |

\* Mnemonic

##### Carry Flag

The carry flag is affected by the following instructions:

- ```CLC```, ```SEC``` (clear/set carry)
- ```ADC```, ```SBC``` (add/subtract with carry)
- ```CMP```, ```CPX```, ```CPY``` (compare A/X/Y)
- ```ASL```, ```LSR``` (arithmetic left/logical right shift)
- ```ROL```, ```ROR``` (rotate left/right through carry)
- ```PLP``` (pull PS from stack)
- ```RTI``` (return from interrupt)

The following instructions are affected by the carry flag:

- ```ADC```, ```SBC``` (add/subtract with carry)
- ```ROL```, ```ROR``` (rotate left/right through carry)
- ```BCC```, ```BCS``` (branch if carry clear/set)

##### Zero Flag

The zero flag is affected by the following instructions:

- ```LDA```, ```LDX```, ```LDY``` (load A/X/Y)
- ```TAX```, ```TAY```, ```TXA```, ```TYA```, ```TSX``` (transfer instructions)
- ```ADC```, ```SBC``` (add/subtract with carry)
- ```CMP```, ```CPX```, ```CPY``` (compare A/X/Y)
- ```INC```, ```INX```, ```INY``` (increment memory/X/Y)
- ```DEC```, ```DEX```, ```DEY``` (increment memory/X/Y)
- ```AND```, ```ORA```, ```EOR``` (bitwise and/or/xor)
- ```BIT``` (bit test)
- ```ASL```, ```LSR``` (arithmetic left/logical right shift)
- ```ROL```, ```ROR``` (rotate left/right through carry)
- ```PLA```, ```PLP``` (pull A/PS from stack)
- ```RTI``` (return from interrupt)

The following instructions are affected by the zero flag:

- ```BNE```, ```BEQ``` (branch if zero clear/set)

##### Interrupt Disable Flag

The interrupt disable flag is affected by the following instructions:

- ```CLI```, ```SEI``` (clear/set interrupt disable)
- ```PLP``` (pull PS from stack)
- ```RTI``` (return from interrupt)

The interrupt disable flag affects only the ```IRQ``` (maskable interrupt request) routine, disabling it if set.

##### Decimal Mode Flag

The decimal mode flag is affected by the following instructions:

- ```SED```, ```CLD``` (clear/set decimal mode)
- ```PLP``` (pull PS from stack)
- ```RTI``` (return from interrupt)

The following instructions are affected by the decimal mode flag:

- ```ADC```, ```SBC``` (add/subtract with carry)

##### Break Flag

The break flag has no affect on any instruction. It may be affected by ```PLP```/```RTI```, but there is no intruction that can actually read the break flag. ```PHP``` (push PS to stack) and ```BRK``` (break) actually share much of the same circuitry on the chip, and both of these instructions push the PS to stack, but the value pushed will always have the break flag bit set, regardless of whether its set in the PS or not.

##### Unused Flag

This bit always appears high when the PS is pushed to the stack, and has no affect on any instruction.

##### Overflow Flag

The overflow flag is affected by the following instructions:

- ```CLV``` (clv overflow)
- ```ADC```, ```SBC``` (add/subtract with carry)
- ```BIT``` (bit test)
- ```PLP``` (pull PS from stack)
- ```RTI``` (return from interrupt)

The following instructions are affected by the overflow flag:

- ```BVS```, ```BVC``` (branch if overflow clear/set)

##### Negative Flag

The negative flag is affected by the following instructions:

- ```LDA```, ```LDX```, ```LDY``` (load A/X/Y)
- ```TAX```, ```TAY```, ```TXA```, ```TYA```, ```TSX``` (transfer instructions)
- ```ADC```, ```SBC``` (add/subtract with carry)
- ```CMP```, ```CPX```, ```CPY``` (compare A/X/Y)
- ```INC```, ```INX```, ```INY``` (increment memory/X/Y)
- ```DEC```, ```DEX```, ```DEY``` (increment memory/X/Y)
- ```AND```, ```ORA```, ```EOR``` (bitwise and/or/xor)
- ```BIT``` (bit test)
- ```ASL```, ```LSR``` (arithmetic left/logical right shift)
- ```ROL```, ```ROR``` (rotate left/right through carry)
- ```PLA```, ```PLP``` (pull A/PS from stack)
- ```RTI``` (return from interrupt)

The following instructions are affected by the negative flag:

- ```BPL```, ```BMI``` (branch if negative clear/set)

#### Memory

The 6502 has a 16-bit wide addressing range. Those 16 address pins are what you would connect your RAMs, ROMs, and IO devices to. As mentioned, for simplicity, we will just say we have 1 65kb RAM connected to all 16 pins.

There are some pre-designated locations/ranges in RAM:

<table>
<tr><td><code>$0000</code>-<code>$00ff</code></td><td>Zero Page</td></tr>
<tr><td><code>$0100</code>-<code>$01ff</code></td><td>Stack</td></tr>
<tr><td><code>$fffa</code>-<code>$fffb</code></td><td><code>IRQ</code> Vector</td></tr>
<tr><td><code>$fffc</code>-<code>$fffd</code></td><td>Reset Vector</td></tr>
<tr><td><code>$fffe</code>-<code>$ffff</code></td><td><code>NMI</code> Vector</td></tr>
</table>

##### Zero Page ```$0000```-```$00FF```

The 6502 has dedicated instructions that operate faster and use fewer bytes when accessing here. 

##### Stack ```$0100```-```$01FF```

Data is written to and read from this area by combining a $01 high byte with the SP as the low byte, where the SP automatically decrements/increments as data is pushed/pulled.

##### Interrupt Vectors ```$fffa```-```$ffff```

These hold 16-bit jump addresses where the processor jumps to when the corresponding interrupt is sent.





## Writing the Emulator

### Language-Specific Imports

By language, here are the imports I will be using (include these at the top of your file):

Python:
```python
import numpy as np
```

C:
```c
#include <stdio.h>
#include <stdint.h>
```

<hr>

### Initializing Registers and Memory

The most immediately obvious thing we can start doing is just assigning variables to represent those important registers we went over earlier:

JavaScript:
```javascript
var A = 0;
var X = 0;
var Y = 0;
var SP = 0;
var PS = 0;
var PC = 0;
var RAM = new Uint8Array(0x10000).fill(0);
```

We could just use a normal array for RAM, but Uint8Arrays are more memory efficient, faster, and keep their contents within range. This means we wont have to worry about masking values that get written to RAM. For registers, we just use normal variables.

Python:
```python
A = 0
X = 0
Y = 0
SP = 0
PS = 0
PC = 0
RAM = bytearray(0x10000)
```

We could just use a normal list for RAM, but bytearrays are more memory efficient and faster as the store values as actual bytes (8-bits). For registers, we just use normal variables.

C:
```c
uint8_t A = 0;
uint8_t X = 0;
uint8_t Y = 0;
uint8_t SP = 0;
uint8_t PS = 0;
uint16_t PC = 0;
uint8_t RAM[0x10000] = {0};
```

<br>

### Helper Functions (Memory Access)

The next thing to do is define a few helper functions. One thing we will want to do is be able to read a 2 byte value stored in memory. This will be stored in two separate memory locations (one after the other). The 6502 is a little endian machine, so the standard is low byte, followed by high byte. This means we will have to read the low byte at a given address, and the high byte at that address+1. And then we have to combine those two values into one 16-bit value.

JavaScript:
```javascript
// verbose:

function read2Bytes(address) {
  var lo = RAM[address+0 & 0xffff];
  var hi = RAM[address+1 & 0xffff];
  return hi << 8 | lo;
}

// simplified:

function read2Bytes(address) {
  return RAM[address]
       | RAM[address+1 & 0xffff] << 8;
}
```

Python:
```python
# verbose:

def read2Bytes(address):
  lo = RAM[address+0 & 0xffff]
  hi = RAM[address+1 & 0xffff]
  return hi << 8 | lo

# simplified:

def read2Bytes(address):
  return (RAM[address]
        | RAM[address+1 & 0xffff] << 8)
```

C:
```c
// verbose:

uint16_t read2Bytes(uint8_t address) {
  uint8_t lo = RAM[address+0 & 0xffff];
  uint8_t hi = RAM[address+1 & 0xffff];
  return hi << 8 | lo;
}

// simplified:

uint16_t read2Bytes(uint16_t address) {
  return RAM[address] | RAM[address+1 & 0xffff] << 8;
}
```

The second mask is neccessary in case the address given is ```$ffff```, meaning address+1 will have to wrap around.

Another thing we will want to do is read the next byte (or, next 2 bytes) at the address the PC is pointing to. This will be how we fetch instruction op-codes as well as arguments.

JavaScript:
```javascript
function nextByte() {
  var value = RAM[PC];
  PC = PC + 1 & 0xffff;
  return value;
}

function next2Bytes() {
  var value = RAM[PC] | RAM[PC+1 & 0xffff] << 8;
  PC = PC + 2 & 0xffff;
  return value;
}
```

Python:
```python
def nextByte():
  global PC
  value = RAM[PC]
  PC = PC + 1 & 0xffff
  return value

def next2Bytes():
  global PC
  value = RAM[PC] | RAM[PC+1 & 0xffff] << 8;
  PC = PC + 2 & 0xffff;
  return value
```

C:
```c
uint8_t nextByte() {
  uint8_t value = RAM[PC];
  PC = PC + 1 & 0xffff;
  return value;
}

uint16_t next2Bytes() {
  uint16_t value = RAM[PC] | RAM[PC+1 & 0xffff] << 8;
  PC = PC + 2 & 0xffff;
  return value;
}
```


We are going to define one more function here. Sometimes we are going to want to read 2 bytes, but do so within the zero page. The reason why will become clear later as we get into adderessing modes, but the function is going to look like this:

JavaScript:
```javascript
function read2BytesZpg(address) {
  return RAM[address & 0xff]
       | RAM[address+1 & 0xff] << 8;
}
```

Python:
```python
def read2BytesZpg(address):
  return (RAM[address & 0xff] | 
          RAM[address+1 & 0xff] << 8)
```

C:
```c
uint16_t read2BytesZpg(uint8_t address) {
  return RAM[address] | RAM[address+1 & 0xff] << 8;
}
```

Same thing as read2Bytes except we just make sure the address wraps in the zero page. Here are all the helper functions we now have:

<details>
<summary>Summary:</summary>

JavaScript:
```javascript
function nextByte() {
  var value = RAM[PC];
  PC = PC + 1 & 0xffff;
  return value;
}

function next2Bytes() {
  var value = RAM[PC] | RAM[PC+1 & 0xffff] << 8;
  PC = PC + 2 & 0xffff;
  return value;
}

function read2Bytes(address) {
  return RAM[address]
       | RAM[address+1 & 0xffff] << 8;
}

function read2BytesZpg(address) {
  return RAM[address & 0xff]
       | RAM[address+1 & 0xff] << 8;
}
```

Python:
```python
def nextByte():
  global PC
  value = RAM[PC]
  PC = PC + 1 & 0xffff
  return value

def next2Bytes():
  global PC
  value = RAM[PC] | RAM[PC+1 & 0xffff] << 8;
  PC = PC + 2 & 0xffff;
  return value

def read2Bytes(address):
  return (RAM[address] | 
          RAM[address+1 & 0xffff] << 8)

def read2BytesZpg(address):
  return (RAM[address & 0xff] | 
          RAM[address+1 & 0xff] << 8)
```

C:
```c
uint8_t nextByte() {
  uint8_t value = RAM[PC];
  PC = PC + 1 & 0xffff;
  return value;
}

uint16_t next2Bytes() {
  uint16_t value = RAM[PC] | RAM[PC+1 & 0xffff] << 8;
  PC = PC + 2 & 0xffff;
  return value;
}

uint16_t read2Bytes(uint16_t address) {
  return RAM[address] | RAM[address+1 & 0xffff] << 8;
}

uint16_t read2BytesZpg(uint8_t address) {
  return RAM[address] | RAM[address+1 & 0xff] << 8;
}
```

</details>

<hr>

### Addressing Modes (Overview)

The 6502 has several ways of retrieving values from memory to use as arguments for any given operation. These are called addressing modes. Each of the addressing modes supported are as follows. I will be putting a 3 letter abbriviation in parenthesis as well, which i will be using throughout.

Implied (imp)

  implied action, no operand.

  ex. TAX\
        transfer A to X.

Accumulator (acc)

  this mode is really the same as implied, as ther is no operand given. it is implied that the operation be done to the accumulator.

  ex. LSR\
        logical shift right (on accumulator).\
      LSR A\
        also valid syntax and does the same thing.

Immediate (imm)

  operand is the value located immediately after the opcode.

  ex. LDA #$10\
        load A with immediate value $10.

Zero Page (zpg)

  operand is retreived by using the next byte as a zero page address.

  ex. LDA $10\
        load A with the value at memory address $10.

Zero Page X (zpx)\
Zero Page Y (zpy)

  zero page address+X or +Y. resulting address wraps around at $ff.

  ex. LDA $10,X\
        load A with the value at memory address $10+X

Absolute (abs)

  operand is retreived by using the next 2 bytes as an absolute address.

  ex. LDA $1234\
        load A with the value at memory address $1234

Absolute X (abx)\
Absolute Y (aby)

  absolute address+X or +Y. resulting address wraps around at $ffff.

  ex. LDA $1234,X\
        load A with the value at memory address $1234+X

Indirect (ind)

  this is actually the same as absolute mode, the next 2 bytes are used as an absolute address to read 2 more bytes (instead of 1). this is only used in jump instructions to set PC.

  ex. jmp ($1234)\
    jump to the 16-bit value at memory addresses $1234 and $1235

Indirect X (inx)

  this is like fetching 2 bytes using zero page x mode, then doing an absolute mode at that address.

  ex. lda ($10,X)\
    load A, get value by using values at $10+X and $10+X+1 as absolute address.

Indirect Y (iny)

  this is like fetching 2 bytes using zero page mode, then doing an absolute y mode at that address.

  ex. lda ($10),Y\
    load A, get value by using values at $10 and $10+1 as absolute address to then add Y to.

Relative (rel)

  this is used for branch instructions, the next byte is used as a signed 8-bit offset to add to the PC.

  ex. bne label\
        change PC by the operand (calculated by the assembler), so that it ends up where the label is.

If any of these modes are confusing (especially indirect modes), its okay. They are confusing, but we will go over them in more detail in the next part when we implement them.





### Addressing Modes (Implementation)

Many of the addressing modes (the ones involving an actual address), I am going to define 2 variations of. The first variation will be prefixed with readRAM and the second with getIDX. Both will be followed by an abbreviation for the addressing mode. The only difference between the two, is that readRAM will essentially be returning the value at the resulting address, getIDX will just return the address.

That being said, we are actually done with a handful of these functions already. Implied mode is easy, it takes no argument, so we dont even need to define any functions for it. Same thing with the accumulator mode. And if you recall, immediate mode just uses the next byte as the operand, so we can just define it as an alias:

JavaScript:
```javascript
var readRAMimm = nextByte;
```

Python:
```python
readRAMimm = nextByte
```

C:
```c
static inline uint8_t readRAMimm(void) {
  return nextByte();
}
```

How about zero page mode? This is where we would use the next byte as the address to read from ram, so we can simply do this:

JavaScript:
```javascript
function readRAMzpg() {
  return RAM[nextByte()];
}
```

Python:
```python
def readRAMzpg():
  return RAM[nextByte()]
```

C:
```c
uint8_t readRAMzpg() {
  return RAM[nextByte()];
}
```

Zero page x and y take a zero page address and add x or y to it. We do need to be sure we stay within the zero page though, so we will have to mask with 0xff.

JavaScript:
```javascript
function readRAMzpx() {
  return RAM[nextByte() + X & 0xff];
}

function readRAMzpy() {
  return RAM[nextByte() + Y & 0xff];
}
```

Python:
```python
def readRAMzpx():
  return RAM[nextByte() + X & 0xff]

def readRAMzpy():
  return RAM[nextByte() + Y & 0xff]
```

C:
```c
uint8_t readRAMzpx(void) {
  return RAM[nextByte() + X & 0xff];
}

uint8_t readRAMzpy(void) {
  return RAM[nextByte() + Y & 0xff];
}
```

We can define absolute modes very similarly with our next2Bytes function in place of nextByte:

JavaScript:
```javascript
function readRAMabs() {
  return RAM[next2Bytes()];
}

function readRAMabx() {
  return RAM[next2Bytes() + X & 0xffff];
}

function readRAMaby() {
  return RAM[next2Bytes() + Y & 0xffff];
}
```

Python:
```python
def readRAMabs():
  return RAM[next2Bytes()]

def readRAMabx():
  return RAM[next2Bytes() + X & 0xffff]

def readRAMaby():
  return RAM[next2Bytes() + Y & 0xffff]
```

C:
```c
uint8_t readRAMabs(void) {
  return RAM[next2Bytes()];
}

uint8_t readRAMabx(void) {
  return RAM[next2Bytes() + X & 0xffff];
}

uint8_t readRAMaby(void) {
  return RAM[next2Bytes() + Y & 0xffff];
}
```

Indirect mode is technically retreiving a pointer so we will define it with the getIDX functions. For indirect x and y, lets take a closer look.

Indirect x is retrieved via:\
zp = next byte\
lo = mem[zp + x]\
hi = mem[zp + x + 1]\
pt = lo | hi << 8\
return mem[pt]

This is where we use read2BytesZpg, giving next byte + X as the argument, and then using that to read from RAM. Heres what that looks like:

JavaScript:
```javascript
function readRAMinx() {
  var zp = nextByte();
  var pt = read2BytesZpg(zp + X);
  return RAM[pt];
}
```

Python:
```python
def readRAMinx():
  zp = nextByte()
  pt = read2BytesZpg(zp + X)
  return RAM[pt]
```

C:
```c
uint8_t readRAMinx(void) {
  uint8_t zp = nextByte();
  uint16_t pt = read2BytesZpg(zp + X);
  return RAM[pt];
}
```

Indirect y is retrieved via:\
zp = next byte\
lo = mem[zp]\
hi = mem[zp + 1]\
pt = lo | hi << 8\
pt = pt + y\
return mem[pt]

Again, we can use read2BytesZpg to help. We do have to mask the final value since adding Y could exceed 16-bits.

JavaScript:
```javascript
function readRAMiny() {
  var zp = nextByte();
  var pt = read2BytesZpg(zp) + Y;
  return RAM[pt & 0xffff];
}
```

Python:
```python
def readRAMiny():
  zp = nextByte()
  pt = read2BytesZpg(zp) + Y
  return RAM[pt & 0xffff]
```

C:
```c
uint8_t readRAMiny(void) {
  uint8_t zp = nextByte();
  uint16_t pt = read2BytesZpg(zp) + Y;
  return RAM[pt & 0xffff];
}
```

Take a look at the syntax for these two modes, indirect x looks like LDA ($00,X), indirect y looks like LDA ($00),Y. You can actually see how that resembles the actual implementations by looking at these two lines:

inx:  pt = read2BytesZpg(zp + X); // ($00,X)\
iny:  pt = read2BytesZpg(zp) + Y; // ($00),Y

Now, if you want, you can simplify these into one liners:

JavaScript:
```javascript
function readRAMinx() {
  return RAM[read2BytesZpg(nextByte() + X)];
}

function readRAMiny() {
  return RAM[read2BytesZpg(nextByte()) + Y & 0xffff];
}
```

Python:
```python
def readRAMinx():
  return RAM[read2BytesZpg(nextByte() + X)]

def readRAMiny():
  return RAM[read2BytesZpg(nextByte()) + Y & 0xffff]
```

C:
```c
uint8_t readRAMinx(void) {
  return RAM[read2BytesZpg(nextByte() + X)];
}

uint8_t readRAMiny(void) {
  return RAM[read2BytesZpg(nextByte()) + Y & 0xffff];
}
```

Finally, for the relative mode, we need to turn the next byte from an unsigned 8-bit value to a signed 8-bit value:

JavaScript:
```javascript
// explicit:

function readRAMrel() {
  var n = nextByte();
  if (n > 127) {
    return n - 256;
  } else {
    return n;
  }
}

// simplified:

function readRAMrel() {
  var n = nextByte();
  return n > 127 ? n - 256 : n;
}

// bitwise implementations:

function readRAMrel() {
  return (nextByte() ^ 0x80) - 0x80;
}

function readRAMrel() {
  return (nextByte() << 24) >> 24;
}
```

Python:
```python
# explicit:

def readRAMrel():
  n = nextByte();
  if (n > 127):
    return n - 256
  else:
    return n

# simplified:

def readRAMrel():
  n = nextByte()
  return n - 256 if n > 127 else n

// bitwise implementation:

def readRAMrel():
  return (nextByte() ^ 0x80) - 0x80
```

C:
```c
static inline int8_t readRAMrel(void) {
  return nextByte();
}
```

Thats it for readRAM functions. Like I mentioned, getIDX will be the same thing except return address instead of RAM[address]. The following functions are exactly the same as the readRAM varients, just without the RAM[ ... ] around the returned value:

JavaScript:
```javascript
function getIDXzpg() {
  return nextByte();
}

function getIDXzpx() {
  return nextByte() + X & 0xff;
}

function getIDXzpy() {
  return nextByte() + Y & 0xff;
}

function getIDXabs() {
  return next2Bytes();
}

function getIDXabx() {
  return next2Bytes() + X & 0xffff;
}

function getIDXaby() {
  return next2Bytes() + Y & 0xffff;
}

function getIDXinx() {
  return read2BytesZpg(nextByte() + X);
}

function getIDXiny() {
  return read2BytesZpg(nextByte()) + Y & 0xffff;
}
```

Python:
```python
def getIDXzpg():
  return nextByte()

def getIDXzpx():
  return nextByte() + X & 0xff

def getIDXzpy():
  return nextByte() + Y & 0xff

def getIDXabs():
  return next2Bytes()

def getIDXabx():
  return next2Bytes() + X & 0xffff

def getIDXaby():
  return next2Bytes() + Y & 0xffff

def getIDXinx():
  return read2BytesZpg(nextByte() + X)

def getIDXiny():
  return read2BytesZpg(nextByte()) + Y & 0xffff
```

C:
```c
uint8_t getIDXzpg(void) {
  return nextByte();
}

uint8_t getIDXzpx(void) {
  return nextByte() + X;
}

uint8_t getIDXzpy(void) {
  return nextByte() + Y;
}

uint16_t getIDXabs(void) {
  return next2Bytes();
}

uint16_t getIDXabx(void) {
  return next2Bytes() + X;
}

uint16_t getIDXaby(void) {
  return next2Bytes() + Y;
}

uint16_t getIDXinx(void) {
  return read2BytesZpg(nextByte() + X);
}

uint16_t getIDXiny(void) {
  return read2BytesZpg(nextByte()) + Y;
}
```

Notice the redundancies in getIDXzpg and getIDXabs, we can actually define them as aliases instead of functions that just call a function:

JavaScript:
```javascript
var getIDXzpg = nextByte;

var getIDXabs = next2Bytes;
```

Python:
```python
getIDXzpg = nextByte

getIDXabs = next2Bytes
```

C:
```c
static inline uint8_t getIDXzpg(void) {
  return nextByte();
}

static inline uint16_t getIDXabs(void) {
  return next2Bytes();
}
```

And last but not least, indirect mode. It looks alot like the getIDXinx/getIDXiny functions, but we are reading the next 2 bytes, and then using that to again read 2 bytes. That looks like this:

JavaScript:
```javascript
function getIDXind() {
  return read2Bytes(next2Bytes());
}
```

Python:
```python
def getIDXind():
  return read2Bytes(next2Bytes())
```

C:
```c
uint16_t getIDXind(void) {
  return read2Bytes(next2Bytes());
}
```

Which would be it, although if you want to, there is a bug in the 6502 with the indirect mode. It happens when the low byte is $ff. If you do jmp ($50ff), you would think it would get the effective address from $50ff and $5100, but it doesnt actually increment the high byte. So what ends up happening is it uses $50ff and $5000. The low byte wraps but there is no carry over. If you want to emulate this bug, you can follow that logic, or you can do it with some bitwise ops:

JavaScript:
```javascript
function getIDXind() {
  var p = next2Bytes();
  return RAM[p] | RAM[(p & 0xff00) | ((p + 1) & 0xff)] << 8;
}
```

Python:
```python
def getIDXind():
  p = next2Bytes()
  return RAM[p] | RAM[(p & 0xff00) | ((p + 1) & 0xff)] << 8
```

C:
```c
uint16_t getIDXind(void) {
  uint16_t p = next2Bytes();
  return RAM[p] | RAM[(p & 0xff00) | ((p + 1) & 0xff)] << 8;
}
```

And now we are done with addressing modes. Heres the final set:

<details>
<summary>Summary:</summary>

JavaScript:
```javascript
var readRAMimm = nextByte;

function readRAMzpg() {
  return RAM[nextByte()];
}

function readRAMzpx() {
  return RAM[nextByte() + X & 0xff];
}

function readRAMzpy() {
  return RAM[nextByte() + Y & 0xff];
}

function readRAMabs() {
  return RAM[next2Bytes()];
}

function readRAMabx() {
  return RAM[next2Bytes() + X & 0xffff];
}

function readRAMaby() {
  return RAM[next2Bytes() + Y & 0xffff];
}

function readRAMinx() {
  return RAM[read2BytesZpg(nextByte() + X)];
}

function readRAMiny() {
  return RAM[read2BytesZpg(nextByte()) + Y & 0xffff];
}

function readRAMrel() {
  return (nextByte() << 24) >> 24;
}

var getIDXzpg = nextByte;

function getIDXzpx() {
  return nextByte() + X & 0xff;
}

function getIDXzpy() {
  return nextByte() + Y & 0xff;
}

var getIDXabs = next2Bytes;

function getIDXabx() {
  return next2Bytes() + X & 0xffff;
}

function getIDXaby() {
  return next2Bytes() + Y & 0xffff;
}

function getIDXinx() {
  return read2BytesZpg(nextByte() + X);
}

function getIDXiny() {
  return read2BytesZpg(nextByte()) + Y & 0xffff;
}

function getIDXind() {
  var p = next2Bytes();
  return RAM[p] | RAM[(p & 0xff00) | ((p + 1) & 0xff)] << 8;
}
```

Python:
```python
readRAMimm = nextByte

def readRAMzpg():
  return RAM[nextByte()]

def readRAMzpx():
  return RAM[nextByte() + X & 0xff]

def readRAMzpy():
  return RAM[nextByte() + Y & 0xff]

def readRAMabs():
  return RAM[next2Bytes()]

def readRAMabx():
  return RAM[next2Bytes() + X & 0xffff]

def readRAMaby():
  return RAM[next2Bytes() + Y & 0xffff]

def readRAMinx():
  return RAM[read2BytesZpg(nextByte() + X)]

def readRAMiny():
  return RAM[read2BytesZpg(nextByte()) + Y & 0xffff]

def readRAMrel():
  return (nextByte() ^ 0x80) - 0x80

getIDXzpg = nextByte

def getIDXzpx():
  return nextByte() + X & 0xff

def getIDXzpy():
  return nextByte() + Y & 0xff

getIDXabs = next2Bytes

def getIDXabx():
  return next2Bytes() + X & 0xffff

def getIDXaby():
  return next2Bytes() + Y & 0xffff

def getIDXinx():
  return read2BytesZpg(nextByte() + X)

def getIDXiny():
  return read2BytesZpg(nextByte()) + Y & 0xffff

def getIDXind():
  p = next2Bytes()
  return RAM[p] | RAM[(p & 0xff00) | ((p + 1) & 0xff)] << 8
```

C:
```c
static inline uint8_t readRAMimm(void) {
  return nextByte();
}

uint8_t readRAMzpg(void) {
  return RAM[nextByte()];
}

uint8_t readRAMzpx(void) {
  return RAM[nextByte() + X & 0xff];
}

uint8_t readRAMzpy(void) {
  return RAM[nextByte() + Y & 0xff];
}

uint8_t readRAMabs(void) {
  return RAM[next2Bytes()];
}

uint8_t readRAMabx(void) {
  return RAM[next2Bytes() + X & 0xffff];
}

uint8_t readRAMaby(void) {
  return RAM[next2Bytes() + Y & 0xffff];
}

uint8_t readRAMinx(void) {
  return RAM[read2BytesZpg(nextByte() + X)];
}

uint8_t readRAMiny(void) {
  return RAM[read2BytesZpg(nextByte()) + Y & 0xffff];
}

static inline int8_t readRAMrel(void) {
  return nextByte();
}

static inline uint8_t getIDXzpg(void) {
  return nextByte();
}

uint8_t getIDXzpx(void) {
  return nextByte() + X;
}

uint8_t getIDXzpy(void) {
  return nextByte() + Y;
}

static inline uint16_t getIDXabs(void) {
  return next2Bytes();
}

uint16_t getIDXabx(void) {
  return next2Bytes() + X;
}

uint16_t getIDXaby(void) {
  return next2Bytes() + Y;
}

uint16_t getIDXinx(void) {
  return read2BytesZpg(nextByte() + X);
}

uint16_t getIDXiny(void) {
  return read2BytesZpg(nextByte()) + Y;
}

uint16_t getIDXind(void) {
  uint16_t p = next2Bytes();
  return RAM[p] | RAM[(p & 0xff00) | ((p + 1) & 0xff)] << 8;
}
```

</details>

<hr>

### Some More Helpers (Flags and the stack)

Before we get into defining the instructions, we should get a couple more things together. First thing we will do is create functions for pushing and pulling values from the stack. We will increment/decrement SP accordingly. Remember that the stack is in the range $0100-$01ff, using SP as the low byte and fixing the high byte at $01. SP at any given time points to the next available location in the stack, so when we push a value, we want to use SP as is and then decrement it. When we pull, we want to first increment it and then use SP afterwards:

JavaScript:
```javascript
function pushStack(value) {
  RAM[0x100 | SP] = value;
  SP = SP - 1 & 0xff;
}

function pullStack() {
  SP = SP + 1 & 0xff;
  return RAM[0x100 | SP];
}
```

Python:
```python
def pushStack(value):
  global SP
  RAM[0x100 | SP] = value
  SP = SP - 1 & 0xff

def pullStack():
  global SP
  SP = SP + 1 & 0xff
  return RAM[0x100 | SP]
```

C:
```c
void pushStack(uint8_t value) {
  RAM[0x100 | SP--] = value;
}

uint8_t pullStack() {
  return RAM[0x100 | ++SP];
}
```

Now for the flags, each bit in PS corresponds to a flag. It will be useful to have masks for each of these bits:

JavaScript:
```javascript
var FLAG_N = 0b10000000; // negative
var FLAG_V = 0b01000000; // overflow
var FLAG_U = 0b00100000; // unused
var FLAG_B = 0b00010000; // break
var FLAG_D = 0b00001000; // decimal mode
var FLAG_I = 0b00000100; // interrupt disable
var FLAG_Z = 0b00000010; // zero
var FLAG_C = 0b00000001; // carry
```

Python:
```python
FLAG_N = 0b10000000 # negative
FLAG_V = 0b01000000 # overflow
FLAG_U = 0b00100000 # unused
FLAG_B = 0b00010000 # break
FLAG_D = 0b00001000 # decimal mode
FLAG_I = 0b00000100 # interrupt disable
FLAG_Z = 0b00000010 # zero
FLAG_C = 0b00000001 # carry
```

C:
```c
#define FLAG_N 0b10000000; // negative
#define FLAG_V 0b01000000; // overflow
#define FLAG_U 0b00100000; // unused
#define FLAG_B 0b00010000; // break
#define FLAG_D 0b00001000; // decimal mode
#define FLAG_I 0b00000100; // interrupt disable
#define FLAG_Z 0b00000010; // zero
#define FLAG_C 0b00000001; // carry
```

Then we can just do PS & FLAG_Z for example to isolate the zero flag bit. If you want to set a flag explicitly to a 1 or 0, you can use this function:

JavaScript:
```javascript
function setFlag(flag, value) {
  PS = PS & ~flag | -value & flag;
}
```

Python:
```python
def setFlag(flag, value):
  global PS
  PS = PS | flag if value else PS & ~flag
```

C:
```c
void setFlag(uint8_t flag, uint8_t value) {
  PS = PS & ~flag | -value & flag;
}
```

Another thing we will need to do for alot of the instructions, is set just the N and Z flags based on some result. These are pretty easy to set. The position of the N flag bit is conviniently in the same position as the sign bit. So by taking FLAG_N & value, we will have the sign bit already in the right position. Z flag is in the second bit position (from the right), which we want to set if the value is zero.

JavaScript:
```javascript
function setNZflags(value) {
  PS = PS & ~(FLAG_N | FLAG_Z)
       | FLAG_N & value
       | !value << 1;
}
```

Python:
```python
def setNZflags(value):
  global PS
  PS = (PS & ~(FLAG_N | FLAG_Z)
       | FLAG_N & value
       | (not value) << 1)
```

C:
```c
void setNZflags(uint8_t value) {
  PS = PS & ~(FLAG_N | FLAG_Z)
       | FLAG_N & value
       | !value << 1;
}
```

For shift instructions, we will also want to set the carry bit along with N and Z.

JavaScript:
```javascript
function setNZCflags(value, carry) {
  PS = PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | !value << 1
       | carry;
}
```

Python:
```python
def setNZCflags(value, carry):
  global PS
  PS = (PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | (not value) << 1
       | carry)
```

C:
```c
void setNZCflags(uint8_t value, uint8_t carry) {
  PS = PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | !value << 1
       | carry;
}
```

For compare instructions, we will want to set N Z and C flags based on the result of a subtraction. This will have the effect of setting N if a < b, Z if a = b, and C if a >= b.

JavaScript:
```javascript
function setCMPflags(value) {
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & value
       | !value << 1
       | FLAG_C & ~value >> 8;
}
```

Python:
```python
def setCMPflags(value, carry):
  global PS
  PS = (PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | (not value) << 1
       | FLAG_C & ~value >> 8)
```

C:
```c
void setCMPflags(uint8_t value, uint8_t carry) {
  PS = PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | !value << 1
       | carry;
}
```

Now we have everything we need to start on instructions. Heres the whole set:

<details>
<summary>Summary:</summary>

JavaScript:
```javascript
function pushStack(value) {
  RAM[0x100 | SP] = value;
  SP = SP - 1 & 0xff;
}

function pullStack() {
  SP = SP + 1 & 0xff;
  return RAM[0x100 | SP];
}

var FLAG_N = 0b10000000 // negative
var FLAG_V = 0b01000000 // overflow
var FLAG_U = 0b00100000 // unused
var FLAG_B = 0b00010000 // break
var FLAG_D = 0b00001000 // decimal mode
var FLAG_I = 0b00000100 // interrupt disable
var FLAG_Z = 0b00000010 // zero
var FLAG_C = 0b00000001 // carry

function setFlag(flag, value) {
  PS = (PS & ~flag) | (-value & flag);
}

function setNZflags(value) {
  PS = PS & ~(FLAG_N | FLAG_Z)
       | FLAG_N & value
       | FLAG_Z & value - 1 >> 7;
}

function setNZCflags(value, carry) {
  PS = PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | FLAG_Z & value - 1 >> 7
       | carry;
}

function setCMPflags(value) {
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & value
       | FLAG_Z & (value & 0xff) - 1 >> 7
       | FLAG_C & ~value >> 8;
}
```

Python:
```python
def pushStack(value):
  global SP
  RAM[0x100 | SP] = value
  SP = SP - 1 & 0xff

def pullStack():
  global SP
  SP = SP + 1 & 0xff
  return RAM[0x100 | SP]

FLAG_N = 0b10000000 # negative
FLAG_V = 0b01000000 # overflow
FLAG_U = 0b00100000 # unused
FLAG_B = 0b00010000 # break
FLAG_D = 0b00001000 # decimal mode
FLAG_I = 0b00000100 # interrupt disable
FLAG_Z = 0b00000010 # zero
FLAG_C = 0b00000001 # carry

def setFlag(flag, value):
  global PS
  PS = PS | flag if value else PS & ~flag

def setNZflags(value):
  global PS
  PS = (PS & ~(FLAG_N | FLAG_Z)
       | FLAG_N & value
       | (not value) << 1)

def setNZCflags(value, carry):
  global PS
  PS = (PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | (not value) << 1
       | carry)
```

C:
```c
#define FLAG_N 0b10000000 // negative
#define FLAG_V 0b01000000 // overflow
#define FLAG_U 0b00100000 // unused
#define FLAG_B 0b00010000 // break
#define FLAG_D 0b00001000 // decimal mode
#define FLAG_I 0b00000100 // interrupt disable
#define FLAG_Z 0b00000010 // zero
#define FLAG_C 0b00000001 // carry

void pushStack(uint8_t value) {
  RAM[0x100 | SP--] = value;
}

uint8_t pullStack() {
  return RAM[0x100 | ++SP];
}

void setFlag(uint8_t flag, uint8_t value) {
  PS = (PS & ~flag) | ((-(value != 0)) & flag);
}

void setNZflags(uint8_t value) {
  PS = PS & ~(FLAG_N | FLAG_Z)
       | FLAG_N & value
       | !value << 1;
}

void setNZCflags(uint8_t value, uint8_t carry) {
  PS = PS & ~(FLAG_N | FLAG_Z | FLAG_C)
       | FLAG_N & value
       | !value << 1
       | carry;
}
```

</details>

<hr>

### Instructions (Overview)

There are 56 distinct instructions supported by the 6502. These can be grouped into several categories:

Memory transfer

  LDA - load A\
  LDX - load X\
  LDY - load Y\
  STA - store A\
  STX - store X\
  STY - store Y\
  TAX - transfer A to X\
  TAY - transfer A to Y\
  TXA - transfer X to A\
  TYA - transfer Y to A\
  TXS - transfer X to SP\
  TSX - transfer SP to X

Stack

  PHA - push A to stack\
  PHP - push PS to stack\
  PLA - pull A from stack\
  PLP - pull PS from stack

Arithmetic

  INC - increment memory value\
  INX - increment X\
  INY - increment Y\
  DEC - decrement memory value\
  DEX - decrement X\
  DEY - decrement Y\
  ADC - add with carry\
  SBC - subtract with "carry"/borrow\
  CMP - compare A to a memory value\
  CPX - compare X to a memory value\
  CPY - compare Y to a memory value

Logic/Bitwise

  AND - bitwise and A with a memory value\
  ORA - bitwise or A with a memory value\
  EOR - bitwise xor A with a memory value\
  BIT - uses bitwise and to set zero flag\
  ASL - arithemetic shift left (on A or a memory value)\
  LSR - logical shift right (on A or a memory value)\
  ROL - rotate left through carry (on A or a memory value)\
  ROR - rotate right through carry (on A or a memory value)

Flags

  SED - set decimal mode\
  SEI - set interrupt disable\
  SEC - set carry\
  CLV - clear overflow\
  CLD - clear decimal mode\
  CLI - clear interrupt disable\
  CLC - clear carry

Control Flow

  BMI - branch if "minus" (n flag set)\
  BVS - branch if overflow set\
  BNE - branch if not equal (zero flag set)\
  BCS - branch if carry set\
  BPL - branch if "plus" (n flag clear)\
  BVC - branch if overflow clear\
  BEQ - branch if equal (zero flag clear)\
  BCC - branch if carry clear\
  JMP - unconditional jump\
  JSR - jump to subroutine\
  RTS - return from subroutine\
  BRK - software interrupt\
  RTI - return from interrupt\
  NOP - no operation

<hr>

### Instructions (Implementation)

These are going to be functions for all the general instructions (the 56 from the last section), although some instructions have multiple op-codes for different addressing modes, we will deal with that later.

We can start with load instructions. These have very easy, direct implementations:

JavaScript:
```javascript
function LDA(v) {
  A = v;
  setNZflags(A);
}

function LDX(v) {
  X = v;
  setNZflags(X);
}

function LDY(v) {
  Y = v;
  setNZflags(Y);
}
```

Python:
```python
def LDA(v):
  global A
  A = v
  setNZflags(A)

def LDX(v):
  global X
  X = v
  setNZflags(X)

def LDY(v):
  global Y
  Y = v
  setNZflags(Y)
```

C:
```c
void LDA(uint8_t v) {
  A = v;
  setNZflags(A);
}

void LDX(uint8_t v) {
  X = v;
  setNZflags(X);
}

void LDY(uint8_t v) {
  Y = v;
  setNZflags(Y);
}
```

We can condense these to one liners by using the assignemts as expressions and passing directly into the setNZflags function:

JavaScript:
```javascript
function LDA(v) { setNZflags(A = v); }
function LDX(v) { setNZflags(X = v); }
function LDY(v) { setNZflags(Y = v); }
```

Python:
```python
def LDA(v):
  global A
  setNZflags(A := v)

def LDX(v):
  global X
  setNZflags(X := v)

def LDY(v):
  global Y
  setNZflags(Y := v)
```

C:
```c
void LDA(uint8_t v) { setNZflags(A = v); }
void LDX(uint8_t v) { setNZflags(X = v); }
void LDY(uint8_t v) { setNZflags(Y = v); }
```



For store instructions we actually have to take in an address as an argument instead of a value. This is where the readRAM/getIDX distinction will come into play, for load instructions we would pass a readRAM... value, but for store instructions we pass a getIDX... value. I will be denoting these by using "v" parameters for instructions expecting readRAM arguents as "values", and "p" parameters for instructions expecting getIDX arguments as "pointers". Anyways, the store instructions are actually even simpler than the load instructions, as no flags need to be set.

JavaScript:
```javascript
function STA(p) { RAM[p] = A; }
function STX(p) { RAM[p] = X; }
function STY(p) { RAM[p] = Y; }
```

Python:
```python
def STA(p): RAM[p] = A
def STX(p): RAM[p] = X
def STY(p): RAM[p] = Y
```

C:
```c
void STA(uint16_t p) { RAM[p] = A; }
void STX(uint16_t p) { RAM[p] = X; }
void STY(uint16_t p) { RAM[p] = Y; }
```


Transfer instructions do set flags (just like the load instructions), and thus look very similar. TXS is the one exception.

JavaScript:
```javascript
function TAX() { setNZflags(X = A); }
function TAY() { setNZflags(Y = A); }
function TXA() { setNZflags(A = X); }
function TYA() { setNZflags(A = Y); }
function TSX() { setNZflags(X = SP);}
function TXS() { SP = X; }
```

Python:
```python
def TAX(v):
  global X
  setNZflags(X := A)

def TAY(v):
  global Y
  setNZflags(Y := A)

def TXA(v):
  global A
  setNZflags(A := X)

def TYA(v):
  global A
  setNZflags(A := Y)

def TSX(v):
  global X
  setNZflags(X := SP)

def TXS(v):
  global SP
  SP := X
```

C:
```c
void TAX(void) { setNZflags(X = A); }
void TAY(void) { setNZflags(Y = A); }
void TXA(void) { setNZflags(A = X); }
void TYA(void) { setNZflags(A = Y); }
void TSX(void) { setNZflags(X = SP);}
void TXS(void) { SP = X; }
```


Stack instructions involve the push/pull stack functions we defined earlier. Pushing/Pulling the accumulator is straightforward. PLA sets N and Z flags.

JavaScript:
```javascript
function PHA() { pushStack(A); }
function PLA() { setNZflags(A = pullStack()); }
```

Python:
```python
def PHA():
  pushStack(A)

def PLA():
  global A
  setNZflags(X := pullStack())
```

C:
```c
void PHA(void) { pushStack(A); }
void PLA(void) { setNZflags(A = pullStack()); }
```

With PS though, there are a couple caveats. First of all, any time PS is pushed to the stack, the U bit will be set in the value pushed. Regardless of what the actual U bit is in PS. The B flags is also ignored when pushing PS. Although the actual value in the B bit of the value pushed depends on the instruction/context. PHP actually shares logic with the BRK instruction, so in this case it will push PS with both the U and B flags set.

JavaScript:
```javascript
function PHP() { pushStack(PS | FLAG_B | FLAG_U); }
```

Python:
```python
def PHP(): pushStack(PS | FLAG_B | FLAG_U)
```

C:
```c
void PHP(void) { pushStack(PS | FLAG_B | FLAG_U); }
```

As long as we make sure to always take care of B and U flags when pushing PS, we can just pull PS as is since theres no other way for the program to interact with those bits.

JavaScript:
```javascript
function PLP() { PS = pullStack(); }
```

Python:
```python
def PLP():
  global PS
  PS = pullStack()
```

C:
```c
void PLP(void) { PS = pullStack(); }
```

Increments and decrements are fairly straightforward:

JavaScript:
```javascript
function INX() { setNZflags(X = X + 1 & 0xff); }
function INY() { setNZflags(Y = Y + 1 & 0xff); }
function DEX() { setNZflags(X = X - 1 & 0xff); }
function DEY() { setNZflags(Y = Y - 1 & 0xff); }

function INC(p) {
  RAM[p]++;
  setNZflags(RAM[p]);
}

function DEC(p) {
  RAM[p]--;
  setNZflags(RAM[p]);
}
```

Python:
```python
def INX():
  global X
  setNZflags(X := X + 1 & 0xff)

def INY():
  global Y
  setNZflags(Y := Y + 1 & 0xff)

def DEX():
  global X
  setNZflags(X := X - 1 & 0xff)

def DEY():
  global Y
  setNZflags(Y := Y - 1 & 0xff)

def INC(p):
  result = RAM[p] + 1 & 0xff
  RAM[p] = result
  setNZflags(result)

def DEC(p):
  result = RAM[p] - 1 & 0xff
  RAM[p] = result
  setNZflags(result)
```

C:
```c
void INX(void) { setNZflags(++X); }
void INY(void) { setNZflags(++Y); }
void DEX(void) { setNZflags(--X); }
void DEY(void) { setNZflags(--Y); }
void INC(uint16_t p) { setNZflags(++RAM[p]); }
void DEC(uint16_t p) { setNZflags(--RAM[p]); }
```


Now for ADC. In theory this is a very simple instruction, just like the rest. It does have a little more involvement, it take the accumulator, a memory value, a carry in, and sets some flags, but its just an addition. This is true if you ignore decimal mode. Fortunately, Ive done all the hard work for you, and worked out an actual correct implementation of ADC that matches the hardware precisely. Ive tested against every single possible set of inputs (A = 0-255, memory value = 0-255, C flag = 0-1, D flag = 0-1) with results from the visual6502 emulator. Ill paste the function I have for ADC when the D flag is set here:

JavaScript:
```javascript
function _ADC_D(v) { // D = 1 **not full implementation! decimal mode only.
  var ci = PS & 1;
  var as = A >> 7 & 1;
  var bs = v >> 7 & 1;
  var ZF = (A + v + ci) & 0xff === 0;
  var al = A & 0x0f;
  var bl = v & 0x0f;
  var rl = al + bl + ci;
  var cl = rl > 0x09;
  if (cl) rl = (rl + 0x06) & 0x0f;
  var ah = A >> 4 & 0x0f;
  var bh = v >> 4 & 0x0f;
  var rh = ah + bh + cl;
  var CF = rh > 0x09;
  var NF = rh >> 3 & 1;
  var VF = ~(as ^ bs) & (as ^ NF) & 1;
  if (CF) rh = (rh + 0x06) & 0x0f;
  var r = rh << 4 | rl;
  A = r;
  PS = PS & 0b00111100
     | NF << 7
     | VF << 6
     | ZF << 1
     | CF;
}
```

This has alot of redundant operations just so that its explicit and clear as possible. Here is a more optimized, full implementation:

JavaScript:
```javascript
function ADC(v) {
  var c = PS & FLAG_C, r;
  if (PS & FLAG_D) {
    r = (A & 0x0f) + (v & 0x0f) + c;
    if (r > 0x09) { 
      r = 0x10 | ((r + 6) & 0x0f);
    }
    r += (A & 0xf0) + (v & 0xf0);
    c = r >> 8 | (r >> 7 & (r >> 6 | r >> 5)) & FLAG_C;
    r += 0x60 * c;
  } else {
    r = A + v + c;
    c = r >> 8 & FLAG_C;
  }
  PS = PS & ~(FLAG_C | FLAG_V | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | (~(A ^ v) & (A ^ r) & FLAG_N) >> 1
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | c;
  A = r & 0xff;
}
```

Python:
```python
def ADC(v):
  global A, PS

  C = PS & FLAG_C
  br = A + v + C
  R = br & 0xff
  CF = br >> 8
  ZF = FLAG_Z if R == 0 else 0

  if PS & FLAG_D:
    rl = (A & 0x0f) + (v & 0x0f) + C
    if rl > 9: rl += 6
    rh = (A >> 4) + (v >> 4) + (rl >> 4)
    if rh > 9: 
      rh += 6
      CF = 1
    R = (rh << 4) | (rl & 0x0f)

  PS = (PS & ~(FLAG_N | FLAG_V | FLAG_Z | FLAG_C)
      | R & FLAG_N
      | (~(A ^ v) & (A ^ R) & 0x80 and FLAG_V
      | ZF
      | CF)
  A = R
```

C:
```c
void ADC(uint8_t v) {
  uint8_t C = PS & FLAG_C;
  uint16_t br = A + v + C;
  uint8_t R = br;
  uint8_t CF = br >> 8;
  uint8_t ZF = R == 0 ? FLAG_Z : 0;

  if (PS & FLAG_D) {
    uint8_t rl = (A & 0x0f) + (v & 0x0f) + C;
    uint8_t cl = (rl + 22) >> 1 & 0xf0; //rl > 9;
    rl += cl * 0x06;
    uint16_t rh = (A & 0xf0) + (v & 0xf0) + (cl << 4);
    CF = (rh + 112) >> 8; // rh > 0x90 & 1;
    rh += CF * 0x60;
    R = rh | rl & 0x0f;
  }

  PS = PS & ~(FLAG_Z | FLAG_V | FLAG_Z | FLAG_C)
     | FLAG_N & R
     | FLAG_V & (~(A ^ v) & (A ^ R)) >> 1 & FLAG_V
     | ZF
     | CF;
  A = R;
}
```

If you were to drop the decimal flag altogether (like the NES does), it becomes:

JavaScript:
```javascript
function ADC(v) {
  var C = PS & FLAG_C;
  var br = A + v + C;
  var R = br & 0xff;
  PS = PS & ~(FLAG_Z | FLAG_V | FLAG_Z | FLAG_C)
     | FLAG_N & R
     | FLAG_V & (~(A ^ v) & (A ^ R)) >> 1 & 64
     | FLAG_Z & R - 1 >> 7
     | br >> 8;
  A = R;
}
```

For SBC we are literally just going to piggy-back off of ADC:

JavaScript:
```javascript
function SBC(v) { ADC(PS & FLAG_D ? 0x99 - v : ~v & 0xff); }
```

Python:
```python
def SBC(v): ADC(~v & 0xff)
```

C:
```c
static inline void SBC(uint8_t v) { return ADC(~v); }
```

Compare instructions are really just a subtraction except only the flags are affected. No carry or decimal flags influence these instructions. These end up setting the N flag is A < v, the C flag if A > v, or the Z flag if A = v. You can implement like this:

JavaScript:
```javascript
// explicit:

function CMP(v) {
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (A < v) << 7
       | (A === v) << 1
       | (A > v);
}

function CPX(v) {
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (X < v) << 7
       | (X === v) << 1
       | (X > v);
}

function CPY(v) {
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (Y < v) << 7
       | (Y === v) << 1
       | (Y > v);
}

// bitwise implementation:

function CMP(v) {
  var r = A - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | FLAG_C & ~r >> 8;
}

function CPX(v) {
  var r = X - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | FLAG_C & ~r >> 8;
}

function CPY(v) {
  var r = Y - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | FLAG_C & ~r >> 8;
}
```

Python:
```python
def CMP(v):
  global PS
  PS = (PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (FLAG_N if A < v else 0)
       | (FLAG_Z if A == v else 0)
       | (FLAG_C if A >= v else 0))

def CPX(v):
  global PS
  PS = (PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (FLAG_N if X < v else 0)
       | (FLAG_Z if X == v else 0)
       | (FLAG_C if X >= v else 0))

def CPY(v):
  global PS
  PS = (PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (FLAG_N if Y < v else 0)
       | (FLAG_Z if Y == v else 0)
       | (FLAG_C if Y >= v else 0))
```

C:
```c
void CMP(uint8_t v) {
  uint16_t r = A - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | !(uint8_t)r << 1
       | FLAG_C & ~r >> 8;
}

void CPX(uint8_t v) {
  uint16_t r = X - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | !(uint8_t)r << 1
       | FLAG_C & ~r >> 8;
}

void CPY(uint8_t v) {
  uint16_t r = Y - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | !(uint8_t)r << 1
       | FLAG_C & ~r >> 8;
}
```

Some more easy instructions:

JavaScript:
```javascript
function AND(v) { setNZflags(A &= v); }
function ORA(v) { setNZflags(A |= v); }
function EOR(v) { setNZflags(A ^= v); }
```

Python:
```python
def AND(v):
  global A
  A &= v
  setNZflags(A)

def ORA(v):
  global A
  A |= v
  setNZflags(A)

def EOR(v):
  global A
  A ^= v
  setNZflags(A)
```

C:
```c
void AND(uint8_t v) { setNZflags(A &= v); }
void ORA(uint8_t v) { setNZflags(A |= v); }
void EOR(uint8_t v) { setNZflags(A ^= v); }
```

The BIT instruction sets the Z flag based on a bitwise and between the accumulator and the operand, but it also has a side effect where it copies bits 6 and 7 from the operand into PS. We can implement like this:

JavaScript:
```javascript
function BIT(v) {
  PS = PS & ~(FLAG_N | FLAG_V | FLAG_Z)
       | FLAG_N & v
       | FLAG_V & v
       | FLAG_Z & (A & v) - 1 >> 7
}
```

Python:
```python
def BIT(v):
  global PS
  PS = (PS & ~(FLAG_N | FLAG_V | FLAG_Z)
       | FLAG_N & v
       | FLAG_V & v
       | (not A) << 1)
```

C:
```c
void BIT(uint8_t v) {
  PS = PS & ~(FLAG_N | FLAG_V | FLAG_Z)
       | FLAG_N & v
       | FLAG_V & v
       | !(A & v) << 1;
}
```

Next, we'll take a look at the shift instructions. We need to make two versions of each, one to work on the accumulator, and one to work on memory values. For each of these, the N and Z flags get set as usual, and the C flag is also set to the bit being shifted out. Lets start with LSR:

JavaScript:
```javascript
function LSR_A() {
  var c = A & FLAG_C;
  A >>>= 1;
  setNZCflags(A,c);
}
```

Python:
```python
def LSR_A():
  global A
  c = A & FLAG_C
  A >>= 1
  setNZCflags(A,c)
```

C:
```c
void LSR_A(void) {
  uint8_t c = A & FLAG_C;
  A >>= 1;
  setNZCflags(A,c);
}
```

ASL is the same thing but in the other direction.

JavaScript:
```javascript
function ASL_A() {
  var c = A >> 7;
  A = A << 1 & 0xff;
  setNZCflags(A,c);
}
```

Python:
```python
def ASL_A():
  global A
  c = A >> 7
  A = A << 1 & 0xff
  setNZCflags(A,c)
```

C:
```c
void ASL_A(void) {
  uint8_t c = A >> 7;
  A <<= 1;
  setNZCflags(A,c);
}
```

ROR and ROL are similar to LSR and ASL, the only difference is instead of shifting in a 0 at either side, we shift in the C flag.

JavaScript:
```javascript
function ROR_A() {
  var c = A & FLAG_C;
  A = A >>> 1 | (PS & FLAG_C) << 7;
  setNZCflags(A,c);
}

function ROL_A() {
  var c = A >> 7;
  A = (A << 1 | PS & FLAG_C) & 0xff;
  setNZCflags(A,c);
}
```

Python:
```python
def ROR_A():
  global A
  c = A & FLAG_C
  A = A >> 1 | (PS & FLAG_C) << 7
  setNZCflags(A,c)

def ROL_A():
  global A
  c = A >> 7
  A = (A << 1 | PS & FLAG_C) & 0xff
  setNZCflags(A,c)
```

C:
```c
void ROR_A(void) {
  uint8_t c = A & FLAG_C;
  A = A >> 1 | (PS & FLAG_C) << 7;
  setNZCflags(A,c);
}

void ROL_A(void) {
  uint8_t c = A >> 7;
  A = A << 1 | PS & FLAG_C;
  setNZCflags(A,c);
}
```

_M versions will work in the same way, they just take in a memory pointer value. We're going to use a temporary variable so we dont have to repeatedly access RAM.

JavaScript:
```javascript
function LSR_M(p) {
  var v = RAM[p];
  var c = v & FLAG_C;
  RAM[p] = v >>>= 1;
  setNZCflags(v,c);
}

function ASL_M(p) {
  var v = RAM[p];
  var c = v >> 7;
  RAM[p] = v = v << 1 & 0xff;
  setNZCflags(v,c);
}

function ROR_M(p) {
  var v = RAM[p];
  var c = v & FLAG_C;
  RAM[p] = v = v >>> 1 | (PS & FLAG_C) << 7;
  setNZCflags(v,c);
}

function ROL_M(p) {
  var v = RAM[p];
  var c = v >> 7;
  RAM[p] = v = (v << 1 | PS & FLAG_C) & 0xff;
  setNZCflags(v,c);
}
```

Python:
```python
def LSR_M(p):
  v = RAM[p]
  c = v & FLAG_C
  RAM[p] = (v := v >> 1)
  setNZCflags(v,c)

def ASL_M(p):
  v = RAM[p]
  c = v >> 7
  RAM[p] = (v := v << 1 & 0xff)
  setNZCflags(v,c)

def ROR_M(p):
  v = RAM[p]
  c = v & FLAG_C
  RAM[p] = (v := v >> 1 | (PS & FLAG_C) << 7)
  setNZCflags(v,c)

def ROL_M(p):
  v = RAM[p]
  c = v >> 7
  RAM[p] = (v := (v << 1 | PS & FLAG_C) & 0xff)
  setNZCflags(v,c)
```

C:
```c
void LSR_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = v & FLAG_C;
  RAM[p] = v >>= 1;
  setNZCflags(v,c);
}

void ASL_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = v >> 7;
  RAM[p] = v <<= 1;
  setNZCflags(v,c);
}

void ROR_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = v & FLAG_C;
  RAM[p] = v = v >> 1 | (PS & FLAG_C) << 7;
  setNZCflags(v,c);
}

void ROL_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = A >> 7;
  RAM[p] = v = v << 1 | PS & FLAG_C;
  setNZCflags(v,c);
}
```

At this point, we're going to knock out some more one-liners. Here are the instructions for setting/clearing flags:

JavaScript:
```javascript
function SEC() { PS |= FLAG_C; }
function SEI() { PS |= FLAG_I; }
function SED() { PS |= FLAG_D; }
function CLC() { PS &= ~FLAG_C; }
function CLI() { PS &= ~FLAG_I; }
function CLD() { PS &= ~FLAG_D; }
function CLV() { PS &= ~FLAG_V; }
```

Python:
```python
def SEC():
  global PS
  PS = PS | FLAG_C

def SEC():
  global PS
  PS = PS | FLAG_C

def SEI():
  global PS
  PS = PS | FLAG_I

def SED():
  global PS
  PS = PS | FLAG_D

def CLC():
  global PS
  PS = PS & ~FLAG_C

def CLI():
  global PS
  PS = PS & ~FLAG_I

def CLD():
  global PS
  PS = PS & ~FLAG_D

def CLV():
  global PS
  PS = PS & ~FLAG_V
```

C:
```c
void SEC(void) { PS |= FLAG_C; }
void SEI(void) { PS |= FLAG_I; }
void SED(void) { PS |= FLAG_D; }
void CLC(void) { PS &= ~FLAG_C; }
void CLI(void) { PS &= ~FLAG_I; }
void CLD(void) { PS &= ~FLAG_D; }
void CLV(void) { PS &= ~FLAG_V; }
```

Branch instructions take in a relative, signed 8-bit value, and they depend on certain flags either being clear or set to determine whether or not to add the offset to PC.

JavaScript:
```javascript
function BCS(v) { if (PS & FLAG_C) PC = PC + v & 0xffff; }
function BEQ(v) { if (PS & FLAG_Z) PC = PC + v & 0xffff; }
function BVS(v) { if (PS & FLAG_V) PC = PC + v & 0xffff; }
function BMI(v) { if (PS & FLAG_N) PC = PC + v & 0xffff; }
function BCC(v) { if (~PS & FLAG_C) PC = PC + v & 0xffff; }
function BNE(v) { if (~PS & FLAG_Z) PC = PC + v & 0xffff; }
function BVC(v) { if (~PS & FLAG_V) PC = PC + v & 0xffff; }
function BPL(v) { if (~PS & FLAG_N) PC = PC + v & 0xffff; }
```

Python:
```python
def BCS(v):
  global PC
  if PS & FLAG_C: PC = PC + v & 0xffff

def BEQ(v):
  global PC
  if PS & FLAG_Z: PC = PC + v & 0xffff

def BVS(v):
  global PC
  if PS & FLAG_V: PC = PC + v & 0xffff

def BMI(v):
  global PC
  if PS & FLAG_N: PC = PC + v & 0xffff

def BCC(v):
  global PC
  if ~PS & FLAG_C: PC = PC + v & 0xffff

def BNE(v):
  global PC
  if ~PS & FLAG_Z: PC = PC + v & 0xffff

def BVC(v):
  global PC
  if ~PS & FLAG_V: PC = PC + v & 0xffff

def BPL(v):
  global PC
  if ~PS & FLAG_N: PC = PC + v & 0xffff
```

C:
```c
void BCS(int8_t v) { if (PS & FLAG_C) PC += v; }
void BEQ(int8_t v) { if (PS & FLAG_Z) PC += v; }
void BVS(int8_t v) { if (PS & FLAG_V) PC += v; }
void BMI(int8_t v) { if (PS & FLAG_N) PC += v; }
void BCC(int8_t v) { if (~PS & FLAG_C) PC += v; }
void BNE(int8_t v) { if (~PS & FLAG_Z) PC += v; }
void BVC(int8_t v) { if (~PS & FLAG_V) PC += v; }
void BPL(int8_t v) { if (~PS & FLAG_N) PC += v; }
```

Jump is dead simple.

JavaScript:
```javascript
function JMP(p) { PC = p; }
```

Python:
```python
def JMP(p):
  global PC
  PC = p
```

C:
```c
void JMP(uint16_t p) { PC = p; }
```

And we are almost there. Just a few more to go. Lets look at JSR. For JSR instructions, two things happen. First, the PC (minus 1) is pushed onto the stack, high byte first, and then low byte. Second, PC jumps to the given operand.

JavaScript:
```javascript
function JSR(p) {
  PC--;
  pushStack(PC >>> 8);
  pushStack(PC);
  PC = p;
}
```

Python:
```python
def JSR(p):
  global PC
  pc = PC - 1 & 0xfff
  pushStack(pc >> 8)
  pushStack(pc & 0xff)
  PC = p
```

C:
```c
void JSR(uint16_t p) {
  PC--;
  pushStack(PC >> 8);
  pushStack(PC);
  PC = p;
}
```

Notice how PC is pushed to the stack as-is in the second push function. This ends up storing the low byte since RAM is a Uint8Array, so it just stores what fits (the low byte).

RTI does the opposite. It needs to pull PC from the stack, low byte first, and then the high byte. It also needs to un-subtract 1 from the return pointer.

JavaScript:
```javascript
function RTS() {
  PC = ((pullStack() | pullStack() << 8) + 1) & 0xffff;
}
```

Python:
```python
def RTS():
  global PC
  PC = ((pullStack() | pullStack() << 8) + 1) & 0xffff
```

C:
```c
void RTS(void) {
  PC = (pullStack() | pullStack() << 8) + 1;
}
```

BRK is somewhat similar to JSR, but it has a few more steps. Specifically, BRK does:

1. increment PC (to the correct return address)
2. push the PC high byte onto the stack
3. push the PC low byte onto the stack
4. push PS onto the stack (with B and U flags set)
5. set the I flag
6. jump to the value store at the irq vector ($fffe-$ffff)

That ends up looking like this:

JavaScript:
```javascript
function BRK() {
  PC++;
  pushStack(PC >>> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}
```

Python:
```python
def BRK():
  global PS, PC
  pc = PC + 1 & 0xffff
  pushStack(PC >> 8)
  pushStack(PC & 0xff)
  pushStack(PS | FLAG_U | FLAG_B)
  PS = PS | FLAG_I
  PC = read2Bytes(0xfffe)
```

C:
```c
void BRK(void) {
  PC++;
  pushStack(PC >> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}
```

RTI just pulls those values in the opposite order:

JavaScript:
```javascript
function RTI() {
  PS = pullStack();
  PC = pullStack() | pullStack() << 8;
}
```

Python:
```python
def RTI():
  global PS, PC
  PS = pullStack()
  PC = pullStack() | pullStack() << 8
```

C:
```c
void RTI(void) {
  PS = pullStack();
  PC = pullStack() | pullStack() << 8;
}
```

One more instruction to go:

JavaScript:
```javascript
function NOP() {}
```

Python:
```python
def NOP(): pass
```

C:
```c
void NOP(void) {}
```

And thats it. Here's the full set:

<details>
<summary>Summary:</summary>

JavaScript:
```javascript
function LDA(v) { setNZflags(A = v); }
function LDX(v) { setNZflags(X = v); }
function LDY(v) { setNZflags(Y = v); }

function STA(p) { RAM[p] = A; }
function STX(p) { RAM[p] = X; }
function STY(p) { RAM[p] = Y; }

function TAX() { setNZflags(X = A); }
function TAY() { setNZflags(Y = A); }
function TXA() { setNZflags(A = X); }
function TYA() { setNZflags(A = Y); }
function TSX() { setNZflags(X = SP);}
function TXS() { SP = X; }

function PHA() { pushStack(A); }
function PHP() { pushStack(PS | FLAG_B | FLAG_U); }
function PLA() { setNZflags(A = pullStack()); }
function PLP() { PS = pullStack(); }

function INX() { setNZflags(X = X + 1 & 0xff); }
function INY() { setNZflags(Y = Y + 1 & 0xff); }
function DEX() { setNZflags(X = X - 1 & 0xff); }
function DEY() { setNZflags(Y = Y - 1 & 0xff); }

function INC(p) {
  RAM[p]++;
  setNZflags(RAM[p]);
}

function DEC(p) {
  RAM[p]--;
  setNZflags(RAM[p]);
}

function ADC(v) {
  var c = PS & FLAG_C, r;
  if (PS & FLAG_D) {
    r = (A & 0x0f) + (v & 0x0f) + c;
    if (r > 0x09) { 
      r = 0x10 | ((r + 6) & 0x0f);
    }
    r += (A & 0xf0) + (v & 0xf0);
    c = r >> 8 | (r >> 7 & (r >> 6 | r >> 5)) & FLAG_C;
    r += 0x60 * c;
  } else {
    r = A + v + c;
    c = r >> 8 & FLAG_C;
  }
  PS = PS & ~(FLAG_C | FLAG_V | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | (~(A ^ v) & (A ^ r) & FLAG_N) >> 1
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | c;
  A = r & 0xff;
}

function SBC(v) { ADC(PS & FLAG_D ? 0x99 - v : ~v & 0xff); }

function CMP(v) {
  var r = A - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | FLAG_C & ~r >> 8;
}

function CPX(v) {
  var r = X - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | FLAG_C & ~r >> 8;
}

function CPY(v) {
  var r = Y - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | FLAG_Z & (r & 0xff) - 1 >> 7
       | FLAG_C & ~r >> 8;
}

function AND(v) { setNZflags(A &= v); }
function ORA(v) { setNZflags(A |= v); }
function EOR(v) { setNZflags(A ^= v); }

function BIT(v) {
  PS = PS & ~(FLAG_N | FLAG_V | FLAG_Z)
       | FLAG_N & v
       | FLAG_V & v
       | FLAG_Z & (A & v) - 1 >> 7
}

function LSR_A() {
  var c = A & FLAG_C;
  A >>>= 1;
  setNZCflags(A,c);
}

function ASL_A() {
  var c = A >> 7;
  A = A << 1 & 0xff;
  setNZCflags(A,c);
}

function ROR_A() {
  var c = A & FLAG_C;
  A = A >>> 1 | (PS & FLAG_C) << 7;
  setNZCflags(A,c);
}

function ROL_A() {
  var c = A >> 7;
  A = (A << 1 | PS & FLAG_C) & 0xff;
  setNZCflags(A,c);
}

function LSR_M(p) {
  var v = RAM[p];
  var c = v & FLAG_C;
  RAM[p] = v >>>= 1;
  setNZCflags(v,c);
}

function ASL_M(p) {
  var v = RAM[p];
  var c = v >> 7;
  RAM[p] = v = v << 1 & 0xff;
  setNZCflags(v,c);
}

function ROR_M(p) {
  var v = RAM[p];
  var c = v & FLAG_C;
  RAM[p] = v = v >>> 1 | (PS & FLAG_C) << 7;
  setNZCflags(v,c);
}

function ROL_M(p) {
  var v = RAM[p];
  var c = v >> 7;
  RAM[p] = v = (v << 1 | PS & FLAG_C) & 0xff;
  setNZCflags(v,c);
}

function SEC() { PS |= FLAG_C; }
function SEI() { PS |= FLAG_I; }
function SED() { PS |= FLAG_D; }
function CLC() { PS &= ~FLAG_C; }
function CLI() { PS &= ~FLAG_I; }
function CLD() { PS &= ~FLAG_D; }
function CLV() { PS &= ~FLAG_V; }

function BCS(v) { if (PS & FLAG_C) PC = PC + v & 0xffff; }
function BEQ(v) { if (PS & FLAG_Z) PC = PC + v & 0xffff; }
function BVS(v) { if (PS & FLAG_V) PC = PC + v & 0xffff; }
function BMI(v) { if (PS & FLAG_N) PC = PC + v & 0xffff; }
function BCC(v) { if (~PS & FLAG_C) PC = PC + v & 0xffff; }
function BNE(v) { if (~PS & FLAG_Z) PC = PC + v & 0xffff; }
function BVC(v) { if (~PS & FLAG_V) PC = PC + v & 0xffff; }
function BPL(v) { if (~PS & FLAG_N) PC = PC + v & 0xffff; }

function JMP(p) { PC = p; }

function JSR(p) {
  PC--;
  pushStack(PC >>> 8);
  pushStack(PC);
  PC = p;
}

function RTS() {
  PC = ((pullStack() | pullStack() << 8) + 1) & 0xffff;
}

function BRK() {
  PC++;
  pushStack(PC >>> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}

function RTI() {
  PS = pullStack();
  PC = pullStack() | pullStack() << 8;
}

function NOP() {}
```

Python:
```python
def LDA(v):
  global A
  setNZflags(A := v)

def LDX(v):
  global X
  setNZflags(X := v)

def LDY(v):
  global Y
  setNZflags(Y := v)

def STA(p): RAM[p] = A
def STX(p): RAM[p] = X
def STY(p): RAM[p] = Y

def TAX(v):
  global X
  setNZflags(X := A)

def TAY(v):
  global Y
  setNZflags(Y := A)

def TXA(v):
  global A
  setNZflags(A := X)

def TYA(v):
  global A
  setNZflags(A := Y)

def TSX(v):
  global X
  setNZflags(X := SP)

def TXS(v):
  global SP
  SP = X

def PHA(): pushStack(A)
def PHP(): pushStack(PS | FLAG_B | FLAG_U)

def PLA():
  global A
  setNZflags(X := pullStack())

def PLP():
  global PS
  PS = pullStack()

def INX():
  global X
  setNZflags(X := X + 1 & 0xff)

def INY():
  global Y
  setNZflags(Y := Y + 1 & 0xff)

def DEX():
  global X
  setNZflags(X := X - 1 & 0xff)

def DEY():
  global Y
  setNZflags(Y := Y - 1 & 0xff)

def INC(p):
  result = RAM[p] + 1 & 0xff
  RAM[p] = result
  setNZflags(result)

def DEC(p):
  result = RAM[p] - 1 & 0xff
  RAM[p] = result
  setNZflags(result)

def ADC(v):
  global A, PS

  C = PS & FLAG_C
  br = A + v + C
  R = br & 0xff
  CF = br >> 8
  ZF = FLAG_Z if R == 0 else 0

  if PS & FLAG_D:
    rl = (A & 0x0f) + (v & 0x0f) + C
    if rl > 9: rl += 6
    rh = (A >> 4) + (v >> 4) + (rl >> 4)
    if rh > 9: 
      rh += 6
      CF = 1
    R = (rh << 4) | (rl & 0x0f)

  PS = (PS & ~(FLAG_N | FLAG_V | FLAG_Z | FLAG_C)
      | R & FLAG_N
      | (~(A ^ v) & (A ^ R) & FLAG_N) and FLAG_V
      | ZF
      | CF)
  A = R

def SBC(v): ADC(~v & 0xff)

def CMP(v):
  global PS
  PS = (PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (FLAG_N if A < v else 0)
       | (FLAG_Z if A == v else 0)
       | (FLAG_C if A >= v else 0))

def CPX(v):
  global PS
  PS = (PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (FLAG_N if X < v else 0)
       | (FLAG_Z if X == v else 0)
       | (FLAG_C if X >= v else 0))

def CPY(v):
  global PS
  PS = (PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | (FLAG_N if Y < v else 0)
       | (FLAG_Z if Y == v else 0)
       | (FLAG_C if Y >= v else 0))

def AND(v):
  global A
  A &= v
  setNZflags(A)

def ORA(v):
  global A
  A |= v
  setNZflags(A)

def EOR(v):
  global A
  A ^= v
  setNZflags(A)

def BIT(v):
  global PS
  PS = (PS & ~(FLAG_N | FLAG_V | FLAG_Z)
       | FLAG_N & v
       | FLAG_V & v
       | (not A) << 1)

def LSR_A():
  global A
  c = A & FLAG_C
  A >>= 1
  setNZCflags(A,c)

def ASL_A():
  global A
  c = A >> 7
  A = A << 1 & 0xff
  setNZCflags(A,c)

def ROR_A():
  global A
  c = A & FLAG_C
  A = A >> 1 | (PS & FLAG_C) << 7
  setNZCflags(A,c)

def ROL_A():
  global A
  c = A >> 7
  A = (A << 1 | PS & FLAG_C) & 0xff
  setNZCflags(A,c)

def LSR_M(p):
  v = RAM[p]
  c = v & FLAG_C
  RAM[p] = (v := v >> 1)
  setNZCflags(v,c)

def ASL_M(p):
  v = RAM[p]
  c = v >> 7
  RAM[p] = (v := v << 1 & 0xff)
  setNZCflags(v,c)

def ROR_M(p):
  v = RAM[p]
  c = v & FLAG_C
  RAM[p] = (v := v >> 1 | (PS & FLAG_C) << 7)
  setNZCflags(v,c)

def ROL_M(p):
  v = RAM[p]
  c = v >> 7
  RAM[p] = (v := (v << 1 | PS & FLAG_C) & 0xff)
  setNZCflags(v,c)

def SEC():
  global PS
  PS = PS | FLAG_C

def SEC():
  global PS
  PS = PS | FLAG_C

def SEI():
  global PS
  PS = PS | FLAG_I

def SED():
  global PS
  PS = PS | FLAG_D

def CLC():
  global PS
  PS = PS & ~FLAG_C

def CLI():
  global PS
  PS = PS & ~FLAG_I

def CLD():
  global PS
  PS = PS & ~FLAG_D

def CLV():
  global PS
  PS = PS & ~FLAG_V

def BCS(v):
  global PC
  if PS & FLAG_C: PC = PC + v & 0xffff

def BEQ(v):
  global PC
  if PS & FLAG_Z: PC = PC + v & 0xffff

def BVS(v):
  global PC
  if PS & FLAG_V: PC = PC + v & 0xffff

def BMI(v):
  global PC
  if PS & FLAG_N: PC = PC + v & 0xffff

def BCC(v):
  global PC
  if ~PS & FLAG_C: PC = PC + v & 0xffff

def BNE(v):
  global PC
  if ~PS & FLAG_Z: PC = PC + v & 0xffff

def BVC(v):
  global PC
  if ~PS & FLAG_V: PC = PC + v & 0xffff

def BPL(v):
  global PC
  if ~PS & FLAG_N: PC = PC + v & 0xffff

def JMP(p):
  global PC
  PC = p

def JSR(p):
  global PC
  pc = PC - 1 & 0xfff
  pushStack(pc >> 8)
  pushStack(pc & 0xff)
  PC = p

def RTS():
  global PC
  PC = ((pullStack() | pullStack() << 8) + 1) & 0xffff

def BRK():
  global PS, PC
  pc = PC + 1 & 0xffff
  pushStack(PC >> 8)
  pushStack(PC & 0xff)
  pushStack(PS | FLAG_U | FLAG_B)
  PS = PS | FLAG_I
  PC = read2Bytes(0xfffe)

def RTI():
  global PS, PC
  PS = pullStack()
  PC = pullStack() | pullStack() << 8

def NOP(): pass
```

C:
```c
void LDA(uint8_t v) { setNZflags(A = v); }
void LDX(uint8_t v) { setNZflags(X = v); }
void LDY(uint8_t v) { setNZflags(Y = v); }

void STA(uint16_t p) { RAM[p] = A; }
void STX(uint16_t p) { RAM[p] = X; }
void STY(uint16_t p) { RAM[p] = Y; }

void TAX(void) { setNZflags(X = A); }
void TAY(void) { setNZflags(Y = A); }
void TXA(void) { setNZflags(A = X); }
void TYA(void) { setNZflags(A = Y); }
void TSX(void) { setNZflags(X = SP);}
void TXS(void) { SP = X; }

void PHA(void) { pushStack(A); }
void PHP(void) { pushStack(PS | FLAG_B | FLAG_U); }
void PLA(void) { setNZflags(A = pullStack()); }
void PLP(void) { PS = pullStack(); }

void INX(void) { setNZflags(++X); }
void INY(void) { setNZflags(++Y); }
void DEX(void) { setNZflags(--X); }
void DEY(void) { setNZflags(--Y); }
void INC(uint16_t p) { setNZflags(++RAM[p]); }
void DEC(uint16_t p) { setNZflags(--RAM[p]); }

void ADC(uint8_t v) {
  uint8_t C = PS & FLAG_C;
  uint16_t br = A + v + C;
  uint8_t R = br;
  uint8_t CF = br >> 8;
  uint8_t ZF = R == 0 ? FLAG_Z : 0;

  if (PS & FLAG_D) {
    uint8_t rl = (A & 0x0f) + (v & 0x0f) + C;
    uint8_t cl = (rl + 22) >> 1 & 0xf0; //rl > 9;
    rl += cl * 0x06;
    uint16_t rh = (A & 0xf0) + (v & 0xf0) + (cl << 4);
    CF = (rh + 112) >> 8; // rh > 0x90 & 1;
    rh += CF * 0x60;
    R = rh | rl & 0x0f;
  }

  PS = PS & ~(FLAG_Z | FLAG_V | FLAG_Z | FLAG_C)
     | FLAG_N & R
     | FLAG_V & (~(A ^ v) & (A ^ R)) >> 1 & FLAG_V
     | ZF
     | CF;
  A = R;
}

static inline void SBC(uint8_t v) { return ADC(~v); }

void CMP(uint8_t v) {
  uint16_t r = A - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | !(uint8_t)r << 1
       | FLAG_C & ~r >> 8;
}

void CPX(uint8_t v) {
  uint16_t r = X - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | !(uint8_t)r << 1
       | FLAG_C & ~r >> 8;
}

void CPY(uint8_t v) {
  uint16_t r = Y - v;
  PS = PS & ~(FLAG_C | FLAG_N | FLAG_Z)
       | FLAG_N & r
       | !(uint8_t)r << 1
       | FLAG_C & ~r >> 8;
}

void AND(uint8_t v) { setNZflags(A &= v); }
void ORA(uint8_t v) { setNZflags(A |= v); }
void EOR(uint8_t v) { setNZflags(A ^= v); }

void BIT(uint8_t v) {
  PS = PS & ~(FLAG_N | FLAG_V | FLAG_Z)
       | FLAG_N & v
       | FLAG_V & v
       | !(A & v) << 1;
}

void LSR_A(void) {
  uint8_t c = A & FLAG_C;
  A >>= 1;
  setNZCflags(A,c);
}

void ASL_A(void) {
  uint8_t c = A >> 7;
  A <<= 1;
  setNZCflags(A,c);
}

void ROR_A(void) {
  uint8_t c = A & FLAG_C;
  A = A >> 1 | (PS & FLAG_C) << 7;
  setNZCflags(A,c);
}

void ROL_A(void) {
  uint8_t c = A >> 7;
  A = A << 1 | PS & FLAG_C;
  setNZCflags(A,c);
}

void LSR_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = v & FLAG_C;
  RAM[p] = v >>= 1;
  setNZCflags(v,c);
}

void ASL_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = v >> 7;
  RAM[p] = v <<= 1;
  setNZCflags(v,c);
}

void ROR_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = v & FLAG_C;
  RAM[p] = v = v >> 1 | (PS & FLAG_C) << 7;
  setNZCflags(v,c);
}

void ROL_M(uint16_t p) {
  uint8_t v = RAM[p];
  uint8_t c = A >> 7;
  RAM[p] = v = v << 1 | PS & FLAG_C;
  setNZCflags(v,c);
}

void SEC(void) { PS |= FLAG_C; }
void SEI(void) { PS |= FLAG_I; }
void SED(void) { PS |= FLAG_D; }
void CLC(void) { PS &= ~FLAG_C; }
void CLI(void) { PS &= ~FLAG_I; }
void CLD(void) { PS &= ~FLAG_D; }
void CLV(void) { PS &= ~FLAG_V; }

void BCS(int8_t v) { if (PS & FLAG_C) PC += v; }
void BEQ(int8_t v) { if (PS & FLAG_Z) PC += v; }
void BVS(int8_t v) { if (PS & FLAG_V) PC += v; }
void BMI(int8_t v) { if (PS & FLAG_N) PC += v; }
void BCC(int8_t v) { if (~PS & FLAG_C) PC += v; }
void BNE(int8_t v) { if (~PS & FLAG_Z) PC += v; }
void BVC(int8_t v) { if (~PS & FLAG_V) PC += v; }
void BPL(int8_t v) { if (~PS & FLAG_N) PC += v; }

void JMP(uint16_t p) { PC = p; }

void JSR(uint16_t p) {
  PC--;
  pushStack(PC >> 8);
  pushStack(PC);
  PC = p;
}

void RTS(void) {
  PC = (pullStack() | pullStack() << 8) + 1;
}

void BRK(void) {
  PC++;
  pushStack(PC >> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}

void RTI(void) {
  PS = pullStack();
  PC = pullStack() | pullStack() << 8;
}

void NOP(void) {}
```

</details>

### Interrupts/Reset routines

There are 3 more functions that we are going to treat kind of like instructions, except these dont get executed like a normal instruction. These are triggered by the hardware instead of the software. They include, reset, NMI, and IRQ.

Starting with reset, this is the first thing that should be executed when the chip "turns on". We should also be able to trigger it by "sending a reset signal". Both of those scenerios we will handle a little bit later, but as for the implementation, there is really only three things that happen in the reset sequence:

1. a couple flags get set/cleared (D flag cleared, I flag set)
2. SP is initialized to $fd
3. PC jumps to the address stored at the reset vector ($fffc-$fffd)

SP ends up at $fd because the 6502 shares some of this logic with the BRK instruction. It actually initializes SP to 0, and then does a couple of ghost pushes to the stack. Although no values are actually put onto the stack at this time because this is done as a read operation instead of a write. Heres what we end up with for reset:

JavaScript:
```javascript
function RES() {
  PS = PS & ~FLAG_D | FLAG_I;
  SP = 0xfd;
  PC = read2Bytes(0xfffc);
}
```

Python:
```python
def RES():
  global PS, SP, PC
  PS = PS & ~FLAG_D | FLAG_I
  SP = 0xfd
  PC = read2Bytes(0xfffc)
```

C:
```c
void RES(void) {
  PS = PS & ~FLAG_D | FLAG_I;
  SP = 0xfd;
  PC = read2Bytes(0xfffc);
}
```

IRQ does the same thing as the BRK instruction, but in this case, the interrupt disable flag can block it.

JavaScript:
```javascript
function IRQ() {
  if (PS & FLAG_I) return;
  pushStack(PC >>> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}
```

Python:
```python
def IRQ():
  global PS, PC
  if PS & FLAG_I: return
  pushStack(PC >> 8)
  pushStack(PC & 0xff)
  pushStack(PS | FLAG_U | FLAG_B)
  PS = PS | FLAG_I
  PC = read2Bytes(0xfffe)
```

C:
```c
void IRQ(void) {
  if (PS & FLAG_I) return;
  pushStack(PC >> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}
```

NMI on the other hand, is a non-maskable interrupt. It will run no matter what if triggered. It also has a different vector where it jumps to than the others.

JavaScript:
```javascript
function NMI() {
  pushStack(PC >>> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffa);
}
```

Python:
```python
def NMI():
  global PS, PC
  pushStack(PC >> 8)
  pushStack(PC & 0xff)
  pushStack(PS | FLAG_U | FLAG_B)
  PS = PS | FLAG_I
  PC = read2Bytes(0xfffa)
```

C:
```c
void NMI(void) {
  pushStack(PC >> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffa);
}
```

Finally, it will be useful to be able to 'trigger' these interrupts and synchronize them with execution by having them run on the next step:

JavaScript:
```javascript
function sendReset() {
  var _step = step;
  step = function() {
    RES();
    step = _step;
  }
}

function sendIRQ() {
  var _step = step;
  step = function() {
    IRQ();
    step = _step;
  }
}

function sendNMI() {
  var _step = step;
  step = function() {
    NMI();
    step = _step;
  }
}
```

Python:
```python
def sendReset():
  global step
  _step = step
  def new_step():
    global step
    nonlocal _step
    RES()
    step = _step
  step = new_step

def sendIRQ():
  global step
  _step = step
  def new_step():
    global step
    nonlocal _step
    IRQ()
    step = _step
  step = new_step

def sendNMI():
  global step
  _step = step
  def new_step():
    global step
    nonlocal _step
    NMI()
    step = _step
  step = new_step

All together:

JavaScript:
```javascript
function RES() {
  PS = PS & ~FLAG_D | FLAG_I;
  SP = 0xfd;
  PC = read2Bytes(0xfffc);
}

function IRQ() {
  if (PS & FLAG_I) return;
  pushStack(PC >>> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}

function NMI() {
  pushStack(PC >>> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffa);
}
```

Python:
```python
def RES():
  global PS, SP, PC
  PS = PS & ~FLAG_D | FLAG_I
  SP = 0xfd
  PC = read2Bytes(0xfffc)

def IRQ():
  global PS, PC
  if PS & FLAG_I: return
  pushStack(PC >> 8)
  pushStack(PC & 0xff)
  pushStack(PS | FLAG_U | FLAG_B)
  PS = PS | FLAG_I
  PC = read2Bytes(0xfffe)

def NMI():
  global PS, PC
  pushStack(PC >> 8)
  pushStack(PC & 0xff)
  pushStack(PS | FLAG_U | FLAG_B)
  PS = PS | FLAG_I
  PC = read2Bytes(0xfffa)
```

C:
```c
void RES(void) {
  PS = PS & ~FLAG_D | FLAG_I;
  SP = 0xfd;
  PC = read2Bytes(0xfffc);
}

void IRQ(void) {
  if (PS & FLAG_I) return;
  pushStack(PC >> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffe);
}

void NMI(void) {
  pushStack(PC >> 8);
  pushStack(PC);
  pushStack(PS | FLAG_U | FLAG_B);
  PS |= FLAG_I;
  PC = read2Bytes(0xfffa);
}
```








### Op-Codes (Overview)

There are a total of 151 op-codes (documented, more on that later), each one being some combination of one of the general instructions we just defined and one of the addressing modes we looked at earlier. We can lay these all out in a table:

|     | $00          | $01          | $02 | $03 | $04          | $05          | $06 | $07          | $08          | $09 | $0A | $0B | $0C          | $0D          | $0E | $0F          |
|-----|--------------|--------------|-----|-----|--------------|--------------|-----|--------------|--------------|-----|-----|-----|--------------|--------------|-----|--------------|
| $00 | BRK<br>imp   | ORA<br>inx   |     |     | ORA<br>zpg   | ASL<br>zpg   |     | PHP<br>imp   | ORA<br>imm   | ASL<br>acc |     |     | ORA<br>abs   | ASL<br>abs   |     |              |
| $10 | BPL<br>rel   | ORA<br>iny   |     |     | ORA<br>zpx   | ASL<br>zpx   |     | CLC<br>imp   | ORA<br>aby   |     |     |     | ORA<br>abx   | ASL<br>abx   |     |              |
| $20 | JSR<br>abs   | AND<br>inx   |     |     | BIT<br>zpg   | AND<br>zpg   | ROL<br>zpg   | PLP<br>imp   | AND<br>imm   | ROL<br>acc |     |     | BIT<br>abs   | AND<br>abs   | ROL<br>abs |     |
| $30 | BMI<br>rel   | AND<br>iny   |     |     | AND<br>zpx   | ROL<br>zpx   |     | SEC<br>imp   | AND<br>aby   |     |     |     | AND<br>abx   | ROL<br>abx   |     |              |
| $40 | RTI<br>imp   | EOR<br>inx   |     |     | EOR<br>zpg   | LSR<br>zpg   |     | PHA<br>imp   | EOR<br>imm   | LSR<br>acc | JMP<br>abs | EOR<br>abs | LSR<br>abs   |              |     |              |
| $50 | BVC<br>rel   | EOR<br>iny   |     |     | EOR<br>zpx   | LSR<br>zpx   |     | CLI<br>imp   | EOR<br>aby   |     |     |     | EOR<br>abx   | LSR<br>abx   |     |              |
| $60 | RTS<br>imp   | ADC<br>inx   |     |     | ADC<br>zpg   | ROR<br>zpg   |     | PLA<br>imp   | ADC<br>imm   | ROR<br>acc | JMP<br>ind | ADC<br>abs | ROR<br>abs   |              |     |              |
| $70 | BVS<br>rel   | ADC<br>iny   |     |     | ADC<br>zpx   | ROR<br>zpx   |     | SEI<br>imp   | ADC<br>aby   |     |     |     | ADC<br>abx   | ROR<br>abx   |     |              |
| $80 |              | STA<br>inx   |     | STY<br>zpg | STA<br>zpg   | STX<br>zpg   |     | DEY<br>imp   | TXA<br>imp   |     | STY<br>abs | STA<br>abs | STX<br>abs   |              |     |              |
| $90 | BCC<br>rel   | STA<br>iny   |     | STY<br>zpx | STA<br>zpx   | STX<br>zpy   |     | TYA<br>imp   | STA<br>aby   | TXS<br>imp |     |     | STA<br>abx   |              |     |              |
| $A0 | LDY<br>imm   | LDA<br>inx   | LDX<br>imm |     | LDY<br>zpg   | LDA<br>zpg   | LDX<br>zpg | TAY<br>imp   | LDA<br>imm   | TAX<br>imp |     | LDY<br>abs | LDA<br>abs | LDX<br>abs   |     |              |
| $B0 | BCS<br>rel   | LDA<br>iny   |     | LDY<br>zpx   | LDA<br>zpx   | LDX<br>zpy   |     | CLV<br>imp   | LDA<br>aby   | TSX<br>imp |     | LDY<br>abx | LDA<br>abx | LDX<br>aby   |     |              |
| $C0 | CPY<br>imm   | CMP<br>inx   |     |     | CPY<br>zpg   | CMP<br>zpg   | DEC<br>zpg | INY<br>imp   | CMP<br>imm   | DEX<br>imp |     | CPY<br>abs | CMP<br>abs | DEC<br>abs   |     |              |
| $D0 | BNE<br>rel   | CMP<br>iny   |     |     | CMP<br>zpx   | DEC<br>zpx   |     | CLD<br>imp   | CMP<br>aby   |     |     |     | CMP<br>abx   | DEC<br>abx   |     |              |
| $E0 | CPX<br>imm   | SBC<br>inx   |     |     | CPX<br>zpg   | SBC<br>zpg   | INC<br>zpg | INX<br>imp   | SBC<br>imm   | NOP<br>imp |     | CPX<br>abs | SBC<br>abs | INC<br>abs   |     |              |
| $F0 | BEQ<br>rel   | SBC<br>iny   |     |     | SBC<br>zpx   | INC<br>zpx   |     | SED<br>imp   | SBC<br>aby   |     |     |     | SBC<br>abx   | INC<br>abx   |     |              |

Alt. Table:

```
    $00 $01 $02 $03 $04 $05 $06 $07 $08 $09 $0A $0B $0C $0D $0E $0F
$00 BRK ORA             ORA ASL     PHP ORA ASL         ORA ASL    
    imp inx             zpg zpg     imp imm acc         abs abs    
$10 BPL ORA             ORA ASL     CLC ORA             ORA ASL    
    rel iny             zpx zpx     imp aby             abx abx    
$20 JSR AND         BIT AND ROL     PLP AND ROL     BIT AND ROL    
    abs inx         zpg zpg zpg     imp imm acc     abs abs abs    
$30 BMI AND             AND ROL     SEC AND             AND ROL    
    rel iny             zpx zpx     imp aby             abx abx    
$40 RTI EOR             EOR LSR     PHA EOR LSR     JMP EOR LSR    
    imp inx             zpg zpg     imp imm acc     abs abs abs    
$50 BVC EOR             EOR LSR     CLI EOR             EOR LSR    
    rel iny             zpx zpx     imp aby             abx abx    
$60 RTS ADC             ADC ROR     PLA ADC ROR     JMP ADC ROR    
    imp inx             zpg zpg     imp imm acc     ind abs abs    
$70 BVS ADC             ADC ROR     SEI ADC             ADC ROR    
    rel iny             zpx zpx     imp aby             abx abx    
$80     STA         STY STA STX     DEY     TXA     STY STA STX    
        inx         zpg zpg zpg     imp     imp     abs abs abs    
$90 BCC STA         STY STA STX     TYA STA TXS         STA        
    rel iny         zpx zpx zpy     imp aby imp         abx        
$A0 LDY LDA LDX     LDY LDA LDX     TAY LDA TAX     LDY LDA LDX    
    imm inx imm     zpg zpg zpg     imp imm imp     abs abs abs    
$B0 BCS LDA         LDY LDA LDX     CLV LDA TSX     LDY LDA LDX    
    rel iny         zpx zpx zpy     imp aby imp     abx abx aby    
$C0 CPY CMP         CPY CMP DEC     INY CMP DEX     CPY CMP DEC    
    imm inx         zpg zpg zpg     imp imm imp     abs abs abs    
$D0 BNE CMP             CMP DEC     CLD CMP             CMP DEC    
    rel iny             zpx zpx     imp aby             abx abx    
$E0 CPX SBC         CPX SBC INC     INX SBC NOP     CPX SBC INC    
    imm inx         zpg zpg zpg     imp imm imp     abs abs abs    
$F0 BEQ SBC             SBC INC     SED SBC             SBC INC    
    rel iny             zpx zpx     imp aby             abx abx    
```

Or, if you like, by instruction:

todo: these need fixed (formatting, inconsitant spacing)

  	inx	zpg	imm	abs	iny	zpx	aby	abx\
ORA	$01	$05	$09	$0D	$11	$15	$19	$1D\
AND	$21	$25	$29	$2D	$31	$35	$39	$3D\
EOR	$41	$45	$49	$4D	$51	$55	$59	$5D\
ADC	$61	$65	$69	$6D	$71	$75	$79	$7D\
STA	$81	$85		  $8D	$91	$95	$99	$9D\
LDA	$A1	$A5	$A9	$AD	$B1	$B5	$B9	$BD\
CMP	$C1	$C5	$C9	$CD	$D1	$D5	$D9	$DD\
SBC	$E1	$E5	$E9	$ED	$F1	$F5	$F9	$FD


	imm	zpg	abs	zpx	abx\
BIT		$24	$2C\
JMP			$4C\
JMP			$6C*\
STX		$84	$8C	$94\
LDY	$A0	$A4	$AC	$B4	$BC\
CPY	$C0	$C4	$CC\
CPX	$E0	$E4	$EC\
* ind


	imm	zpg	acc	abs	zpx	abx\
ASL		$06	$0A	$0E	$16	$1E\
ROL		$26	$2A	$2E	$36	$3E\
LSR		$46	$4A	$4E	$56	$5E\
ROR		$66	$6A	$6E	$76	$7E\
STX		$86		$8E	$96*	\
LDX	$A2	$A6		$AE	$B6*	$BE**\
DEC		$C6		$CE	$D6	$DE\
INC		$E6		$EE	$F6	$FE\
*  zpy mode\
** aby mode

implied\
PHP $08\
PLP $28\
PHA $48\
PLA $68\
TXA $8A\
TYA $98\
TXS $9A\
TAX $AA\
TAY $A8\
TSX $BA\
DEX $CA\
DEY $88\
INY $C8\
INX $E8

CLC $18\
SEC $38\
CLI $58\
SEI $78\
CLV $B8\
CLD $D8\
SED $F8

BPL $10\
BMI $30\
BVC $50\
BVS $70\
BCC $90\
BCS $B0\
BNE $D0\
BEQ $F0

BRK $00\
JSR $20\
RTI $40\
RTS $60

NOP $EA

<hr>

### Op-Codes (Implementation)

Basically we want to create a function called step, which reads the next byte, and uses it as an op-code to pick the appropriate instruction to run. For this, we are going to use a giant switch case block, and map each of the op-codes to the corresponding instruction/addressing mode.

JavaScript:
```javascript
function step() {
  var opcode = nextByte();
  switch (opcode) {
    case 0x00: BRK(); break;              // BRK imp
    case 0x01: ORA(readRAMinx()); break;  // ORA inx
    case 0x02: HLT(readRAMimm()); break;  // HLT imm*
    case 0x04: NOP(readRAMzpg()); break;  // NOP zpg*
    case 0x05: ORA(readRAMzpg()); break;  // ORA zpg
    case 0x06: ASL_M(getIDXzpg()); break; // ASL zpg
    case 0x08: PHP(); break;              // PHP imp
    case 0x09: ORA(readRAMimm()); break;  // ORA imm
    case 0x0A: ASL_A(); break;            // ASL acc
    case 0x0C: NOP(readRAMabs()); break;  // NOP abs*
    case 0x0D: ORA(readRAMabs()); break;  // ORA abs
    case 0x0E: ASL_M(getIDXabs()); break; // ASL abs
    case 0x10: BPL(readRAMrel()); break;  // BPL rel
    case 0x11: ORA(readRAMiny()); break;  // ORA iny
    case 0x12: HLT(readRAMimm()); break;  // HLT imm*
    case 0x14: NOP(readRAMzpg()); break;  // NOP zpg*
    case 0x15: ORA(readRAMzpx()); break;  // ORA zpx
    case 0x16: ASL_M(getIDXzpx()); break; // ASL zpx
    case 0x18: CLC(); break;              // CLC imp
    case 0x19: ORA(readRAMaby()); break;  // ORA aby
    case 0x1A: NOP(); break;              // NOP imp*
    case 0x1C: NOP(readRAMabx()); break;  // NOP abx*
    case 0x1D: ORA(readRAMabx()); break;  // ORA abx
    case 0x1E: ASL_M(getIDXabx()); break; // ASL abx
    case 0x20: JSR(getIDXabs()); break;   // JSR abs
    case 0x21: AND(readRAMinx()); break;  // AND inx
    case 0x22: HLT(readRAMimm()); break;  // HLT imm*
    case 0x24: BIT(readRAMzpg()); break;  // BIT zpg
    case 0x25: AND(readRAMzpg()); break;  // AND zpg
    case 0x26: ROL_M(getIDXzpg()); break; // ROL zpg
    case 0x28: PLP(); break;              // PLP imp
    case 0x29: AND(readRAMimm()); break;  // AND imm
    case 0x2A: ROL_A(); break;            // ROL acc
    case 0x2C: BIT(readRAMabs()); break;  // BIT abs
    case 0x2D: AND(readRAMabs()); break;  // AND abs
    case 0x2E: ROL_M(getIDXabs()); break; // ROL abs
    case 0x30: BMI(readRAMrel()); break;  // BMI rel
    case 0x31: AND(readRAMiny()); break;  // AND iny
    case 0x32: HLT(readRAMimm()); break;  // HLT imm*
    case 0x34: NOP(readRAMzpx()); break;  // NOP zpx*
    case 0x35: AND(readRAMzpx()); break;  // AND zpx
    case 0x36: ROL_M(getIDXzpx()); break; // ROL zpx
    case 0x38: SEC(); break;              // SEC imp
    case 0x39: AND(readRAMaby()); break;  // AND aby
    case 0x3A: NOP(); break;              // NOP imp*
    case 0x3D: AND(readRAMabx()); break;  // AND abx
    case 0x3E: ROL_M(getIDXabx()); break; // ROL abx
    case 0x40: RTI(); break;              // RTI imp
    case 0x41: EOR(readRAMinx()); break;  // EOR inx
    case 0x42: HLT(readRAMimm()); break;  // HLT imm*
    case 0x44: NOP(readRAMzpg()); break;  // NOP zpg*
    case 0x45: EOR(readRAMzpg()); break;  // EOR zpg
    case 0x46: LSR_M(getIDXzpg()); break; // LSR zpg
    case 0x48: PHA(); break;              // PHA imp
    case 0x49: EOR(readRAMimm()); break;  // EOR imm
    case 0x4A: LSR_A(); break;            // LSR acc
    case 0x4C: JMP(getIDXabs()); break;   // JMP abs
    case 0x4D: EOR(readRAMabs()); break;  // EOR abs
    case 0x4E: LSR_M(getIDXabs()); break; // LSR abs
    case 0x50: BVC(readRAMrel()); break;  // BVC rel
    case 0x51: EOR(readRAMiny()); break;  // EOR iny
    case 0x52: HLT(readRAMimm()); break;  // HLT imm*
    case 0x54: NOP(readRAMzpx()); break;  // NOP zpx*
    case 0x55: EOR(readRAMzpx()); break;  // EOR zpx
    case 0x56: LSR_M(getIDXzpx()); break; // LSR zpx
    case 0x58: CLI(); break;              // CLI imp
    case 0x59: EOR(readRAMaby()); break;  // EOR aby
    case 0x5A: NOP(); break;              // NOP imp*
    case 0x5D: EOR(readRAMabx()); break;  // EOR abx
    case 0x5E: LSR_M(getIDXabx()); break; // LSR abx
    case 0x60: RTS(); break;              // RTS imp
    case 0x61: ADC(readRAMinx()); break;  // ADC inx
    case 0x62: HLT(readRAMimm()); break;  // HLT imm*
    case 0x64: NOP(readRAMzpg()); break;  // NOP zpg*
    case 0x65: ADC(readRAMzpg()); break;  // ADC zpg
    case 0x66: ROR_M(getIDXzpg()); break; // ROR zpg
    case 0x68: PLA(); break;              // PLA imp
    case 0x69: ADC(readRAMimm()); break;  // ADC imm
    case 0x6A: ROR_A(); break;            // ROR acc
    case 0x6C: JMP(getIDXind()); break;   // JMP ind
    case 0x6D: ADC(readRAMabs()); break;  // ADC abs
    case 0x6E: ROR_M(getIDXabs()); break; // ROR abs
    case 0x70: BVS(readRAMrel()); break;  // BVS rel
    case 0x71: ADC(readRAMiny()); break;  // ADC iny
    case 0x72: HLT(readRAMimm()); break;  // HLT imm*
    case 0x74: NOP(readRAMzpg()); break;  // NOP zpg*
    case 0x75: ADC(readRAMzpx()); break;  // ADC zpx
    case 0x76: ROR_M(getIDXzpx()); break; // ROR zpx
    case 0x78: SEI(); break;              // SEI imp
    case 0x79: ADC(readRAMaby()); break;  // ADC aby
    case 0x7A: NOP(); break;              // NOP imp*
    case 0x7D: ADC(readRAMabx()); break;  // ADC abx
    case 0x7E: ROR_M(getIDXabx()); break; // ROR abx
    case 0x80: NOP(readRAMimm()); break;  // NOP imm*
    case 0x81: STA(getIDXinx()); break;   // STA inx
    case 0x82: NOP(readRAMimm()); break;  // NOP imm*
    case 0x84: STY(getIDXzpg()); break;   // STY zpg
    case 0x85: STA(getIDXzpg()); break;   // STA zpg
    case 0x86: STX(getIDXzpg()); break;   // STX zpg
    case 0x88: DEY(); break;              // DEY imp
    case 0x89: NOP(readRAMimm()); break;  // NOP imm*
    case 0x8A: TXA(); break;              // TXA imp
    case 0x8C: STY(getIDXabs()); break;   // STY abs
    case 0x8D: STA(getIDXabs()); break;   // STA abs
    case 0x8E: STX(getIDXabs()); break;   // STX abs
    case 0x90: BCC(readRAMrel()); break;  // BCC rel
    case 0x91: STA(getIDXiny()); break;   // STA iny
    case 0x92: HLT(readRAMimm()); break;  // HLT imm*
    case 0x94: STY(getIDXzpx()); break;   // STY zpx
    case 0x95: STA(getIDXzpx()); break;   // STA zpx
    case 0x96: STX(getIDXzpy()); break;   // STX zpy
    case 0x98: TYA(); break;              // TYA imp
    case 0x99: STA(getIDXaby()); break;   // STA aby
    case 0x9A: TXS(); break;              // TXS imp
    case 0x9D: STA(getIDXabx()); break;   // STA abx
    case 0xA0: LDY(readRAMimm()); break;  // LDY imm
    case 0xA1: LDA(readRAMinx()); break;  // LDA inx
    case 0xA2: LDX(readRAMimm()); break;  // LDX imm
    case 0xA4: LDY(readRAMzpg()); break;  // LDY zpg
    case 0xA5: LDA(readRAMzpg()); break;  // LDA zpg
    case 0xA6: LDX(readRAMzpg()); break;  // LDX zpg
    case 0xA8: TAY(); break;              // TAY imp
    case 0xA9: LDA(readRAMimm()); break;  // LDA imm
    case 0xAA: TAX(); break;              // TAX imp
    case 0xAC: LDY(readRAMabs()); break;  // LDY abs
    case 0xAD: LDA(readRAMabs()); break;  // LDA abs
    case 0xAE: LDX(readRAMabs()); break;  // LDX abs
    case 0xB0: BCS(readRAMrel()); break;  // BCS rel
    case 0xB1: LDA(readRAMiny()); break;  // LDA iny
    case 0xB2: HLT(readRAMimm()); break;  // HLT imm*
    case 0xB4: LDY(readRAMzpx()); break;  // LDY zpx
    case 0xB5: LDA(readRAMzpx()); break;  // LDA zpx
    case 0xB6: LDX(readRAMzpy()); break;  // LDX zpy
    case 0xB8: CLV(); break;              // CLV imp
    case 0xB9: LDA(readRAMaby()); break;  // LDA aby
    case 0xBA: TSX(); break;              // TSX imp
    case 0xBC: LDY(readRAMabx()); break;  // LDY abx
    case 0xBD: LDA(readRAMabx()); break;  // LDA abx
    case 0xBE: LDX(readRAMaby()); break;  // LDX aby
    case 0xC0: CPY(readRAMimm()); break;  // CPY imm
    case 0xC1: CMP(readRAMinx()); break;  // CMP inx
    case 0xC2: NOP(readRAMimm()); break;  // NOP imm*
    case 0xC4: CPY(readRAMzpg()); break;  // CPY zpg
    case 0xC5: CMP(readRAMzpg()); break;  // CMP zpg
    case 0xC6: DEC(getIDXzpg()); break;   // DEC zpg
    case 0xC8: INY(); break;              // INY imp
    case 0xC9: CMP(readRAMimm()); break;  // CMP imm
    case 0xCA: DEX(); break;              // DEX imp
    case 0xCC: CPY(readRAMabs()); break;  // CPY abs
    case 0xCD: CMP(readRAMabs()); break;  // CMP abs
    case 0xCE: DEC(getIDXabs()); break;   // DEC abs
    case 0xD0: BNE(readRAMrel()); break;  // BNE rel
    case 0xD1: CMP(readRAMiny()); break;  // CMP iny
    case 0xD2: HLT(readRAMimm()); break;  // HLT imm*
    case 0xD4: NOP(readRAMzpx()); break;  // NOP zpx*
    case 0xD5: CMP(readRAMzpx()); break;  // CMP zpx
    case 0xD6: DEC(getIDXzpx()); break;   // DEC zpx
    case 0xD8: CLD(); break;              // CLD imp
    case 0xD9: CMP(readRAMaby()); break;  // CMP aby
    case 0xDA: NOP(); break;              // NOP imp*
    case 0xDD: CMP(readRAMabx()); break;  // CMP abx
    case 0xDE: DEC(getIDXabx()); break;   // DEC abx
    case 0xE0: CPX(readRAMimm()); break;  // CPX imm
    case 0xE1: SBC(readRAMinx()); break;  // SBC inx
    case 0xE2: NOP(readRAMimm()); break;  // NOP imm*
    case 0xE4: CPX(readRAMzpg()); break;  // CPX zpg
    case 0xE5: SBC(readRAMzpg()); break;  // SBC zpg
    case 0xE6: INC(getIDXzpg()); break;   // INC zpg
    case 0xE8: INX(); break;              // INX imp
    case 0xE9: SBC(readRAMimm()); break;  // SBC imm
    case 0xEA: NOP(); break;              // NOP imp
    case 0xEC: CPX(readRAMabs()); break;  // CPX abs
    case 0xED: SBC(readRAMabs()); break;  // SBC abs
    case 0xEE: INC(getIDXabs()); break;   // INC abs
    case 0xF0: BEQ(readRAMrel()); break;  // BEQ rel
    case 0xF1: SBC(readRAMiny()); break;  // SBC iny
    case 0xF2: HLT(readRAMimm()); break;  // HLT imm*
    case 0xF4: NOP(readRAMzpx()); break;  // NOP zpx*
    case 0xF5: SBC(readRAMzpx()); break;  // SBC zpx
    case 0xF6: INC(getIDXzpx()); break;   // INC zpx
    case 0xF8: SED(); break;              // SED imp
    case 0xF9: SBC(readRAMaby()); break;  // SBC aby
    case 0xFA: NOP(); break;              // NOP imp*
    case 0xFD: SBC(readRAMabx()); break;  // SBC abx
    case 0xFE: INC(getIDXabx()); break;   // INC abx
    default: NOP(); break;                // default to nop
  }
}
```

Python:
```python
instructions = {
  0x00: lambda: BRK(),              # BRK imp
  0x01: lambda: ORA(readRAMinx()),  # ORA inx
  0x02: lambda: HLT(readRAMimm()),  # HLT imm*
  0x04: lambda: NOP(readRAMzpg()),  # NOP zpg*
  0x05: lambda: ORA(readRAMzpg()),  # ORA zpg
  0x06: lambda: ASL_M(getIDXzpg()), # ASL zpg
  0x08: lambda: PHP(),              # PHP imp
  0x09: lambda: ORA(readRAMimm()),  # ORA imm
  0x0A: lambda: ASL_A(),            # ASL acc
  0x0C: lambda: NOP(readRAMabs()),  # NOP abs*
  0x0D: lambda: ORA(readRAMabs()),  # ORA abs
  0x0E: lambda: ASL_M(getIDXabs()), # ASL abs
  0x10: lambda: BPL(readRAMrel()),  # BPL rel
  0x11: lambda: ORA(readRAMiny()),  # ORA iny
  0x12: lambda: HLT(readRAMimm()),  # HLT imm*
  0x14: lambda: NOP(readRAMzpg()),  # NOP zpg*
  0x15: lambda: ORA(readRAMzpx()),  # ORA zpx
  0x16: lambda: ASL_M(getIDXzpx()), # ASL zpx
  0x18: lambda: CLC(),              # CLC imp
  0x19: lambda: ORA(readRAMaby()),  # ORA aby
  0x1A: lambda: NOP(),              # NOP imp*
  0x1C: lambda: NOP(readRAMabx()),  # NOP abx*
  0x1D: lambda: ORA(readRAMabx()),  # ORA abx
  0x1E: lambda: ASL_M(getIDXabx()), # ASL abx
  0x20: lambda: JSR(getIDXabs()),   # JSR abs
  0x21: lambda: AND(readRAMinx()),  # AND inx
  0x22: lambda: HLT(readRAMimm()),  # HLT imm*
  0x24: lambda: BIT(readRAMzpg()),  # BIT zpg
  0x25: lambda: AND(readRAMzpg()),  # AND zpg
  0x26: lambda: ROL_M(getIDXzpg()), # ROL zpg
  0x28: lambda: PLP(),              # PLP imp
  0x29: lambda: AND(readRAMimm()),  # AND imm
  0x2A: lambda: ROL_A(),            # ROL acc
  0x2C: lambda: BIT(readRAMabs()),  # BIT abs
  0x2D: lambda: AND(readRAMabs()),  # AND abs
  0x2E: lambda: ROL_M(getIDXabs()), # ROL abs
  0x30: lambda: BMI(readRAMrel()),  # BMI rel
  0x31: lambda: AND(readRAMiny()),  # AND iny
  0x32: lambda: HLT(readRAMimm()),  # HLT imm*
  0x34: lambda: NOP(readRAMzpx()),  # NOP zpx*
  0x35: lambda: AND(readRAMzpx()),  # AND zpx
  0x36: lambda: ROL_M(getIDXzpx()), # ROL zpx
  0x38: lambda: SEC(),              # SEC imp
  0x39: lambda: AND(readRAMaby()),  # AND aby
  0x3A: lambda: NOP(),              # NOP imp*
  0x3D: lambda: AND(readRAMabx()),  # AND abx
  0x3E: lambda: ROL_M(getIDXabx()), # ROL abx
  0x40: lambda: RTI(),              # RTI imp
  0x41: lambda: EOR(readRAMinx()),  # EOR inx
  0x42: lambda: HLT(readRAMimm()),  # HLT imm*
  0x44: lambda: NOP(readRAMzpg()),  # NOP zpg*
  0x45: lambda: EOR(readRAMzpg()),  # EOR zpg
  0x46: lambda: LSR_M(getIDXzpg()), # LSR zpg
  0x48: lambda: PHA(),              # PHA imp
  0x49: lambda: EOR(readRAMimm()),  # EOR imm
  0x4A: lambda: LSR_A(),            # LSR acc
  0x4C: lambda: JMP(getIDXabs()),   # JMP abs
  0x4D: lambda: EOR(readRAMabs()),  # EOR abs
  0x4E: lambda: LSR_M(getIDXabs()), # LSR abs
  0x50: lambda: BVC(readRAMrel()),  # BVC rel
  0x51: lambda: EOR(readRAMiny()),  # EOR iny
  0x52: lambda: HLT(readRAMimm()),  # HLT imm*
  0x54: lambda: NOP(readRAMzpx()),  # NOP zpx*
  0x55: lambda: EOR(readRAMzpx()),  # EOR zpx
  0x56: lambda: LSR_M(getIDXzpx()), # LSR zpx
  0x58: lambda: CLI(),              # CLI imp
  0x59: lambda: EOR(readRAMaby()),  # EOR aby
  0x5A: lambda: NOP(),              # NOP imp*
  0x5D: lambda: EOR(readRAMabx()),  # EOR abx
  0x5E: lambda: LSR_M(getIDXabx()), # LSR abx
  0x60: lambda: RTS(),              # RTS imp
  0x61: lambda: ADC(readRAMinx()),  # ADC inx
  0x62: lambda: HLT(readRAMimm()),  # HLT imm*
  0x64: lambda: NOP(readRAMzpg()),  # NOP zpg*
  0x65: lambda: ADC(readRAMzpg()),  # ADC zpg
  0x66: lambda: ROR_M(getIDXzpg()), # ROR zpg
  0x68: lambda: PLA(),              # PLA imp
  0x69: lambda: ADC(readRAMimm()),  # ADC imm
  0x6A: lambda: ROR_A(),            # ROR acc
  0x6C: lambda: JMP(getIDXind()),   # JMP ind
  0x6D: lambda: ADC(readRAMabs()),  # ADC abs
  0x6E: lambda: ROR_M(getIDXabs()), # ROR abs
  0x70: lambda: BVS(readRAMrel()),  # BVS rel
  0x71: lambda: ADC(readRAMiny()),  # ADC iny
  0x72: lambda: HLT(readRAMimm()),  # HLT imm*
  0x74: lambda: NOP(readRAMzpg()),  # NOP zpg*
  0x75: lambda: ADC(readRAMzpx()),  # ADC zpx
  0x76: lambda: ROR_M(getIDXzpx()), # ROR zpx
  0x78: lambda: SEI(),              # SEI imp
  0x79: lambda: ADC(readRAMaby()),  # ADC aby
  0x7A: lambda: NOP(),              # NOP imp*
  0x7D: lambda: ADC(readRAMabx()),  # ADC abx
  0x7E: lambda: ROR_M(getIDXabx()), # ROR abx
  0x80: lambda: NOP(readRAMimm()),  # NOP imm*
  0x81: lambda: STA(getIDXinx()),   # STA inx
  0x82: lambda: NOP(readRAMimm()),  # NOP imm*
  0x84: lambda: STY(getIDXzpg()),   # STY zpg
  0x85: lambda: STA(getIDXzpg()),   # STA zpg
  0x86: lambda: STX(getIDXzpg()),   # STX zpg
  0x88: lambda: DEY(),              # DEY imp
  0x89: lambda: NOP(readRAMimm()),  # NOP imm*
  0x8A: lambda: TXA(),              # TXA imp
  0x8C: lambda: STY(getIDXabs()),   # STY abs
  0x8D: lambda: STA(getIDXabs()),   # STA abs
  0x8E: lambda: STX(getIDXabs()),   # STX abs
  0x90: lambda: BCC(readRAMrel()),  # BCC rel
  0x91: lambda: STA(getIDXiny()),   # STA iny
  0x92: lambda: HLT(readRAMimm()),  # HLT imm*
  0x94: lambda: STY(getIDXzpx()),   # STY zpx
  0x95: lambda: STA(getIDXzpx()),   # STA zpx
  0x96: lambda: STX(getIDXzpy()),   # STX zpy
  0x98: lambda: TYA(),              # TYA imp
  0x99: lambda: STA(getIDXaby()),   # STA aby
  0x9A: lambda: TXS(),              # TXS imp
  0x9D: lambda: STA(getIDXabx()),   # STA abx
  0xA0: lambda: LDY(readRAMimm()),  # LDY imm
  0xA1: lambda: LDA(readRAMinx()),  # LDA inx
  0xA2: lambda: LDX(readRAMimm()),  # LDX imm
  0xA4: lambda: LDY(readRAMzpg()),  # LDY zpg
  0xA5: lambda: LDA(readRAMzpg()),  # LDA zpg
  0xA6: lambda: LDX(readRAMzpg()),  # LDX zpg
  0xA8: lambda: TAY(),              # TAY imp
  0xA9: lambda: LDA(readRAMimm()),  # LDA imm
  0xAA: lambda: TAX(),              # TAX imp
  0xAC: lambda: LDY(readRAMabs()),  # LDY abs
  0xAD: lambda: LDA(readRAMabs()),  # LDA abs
  0xAE: lambda: LDX(readRAMabs()),  # LDX abs
  0xB0: lambda: BCS(readRAMrel()),  # BCS rel
  0xB1: lambda: LDA(readRAMiny()),  # LDA iny
  0xB2: lambda: HLT(readRAMimm()),  # HLT imm*
  0xB4: lambda: LDY(readRAMzpx()),  # LDY zpx
  0xB5: lambda: LDA(readRAMzpx()),  # LDA zpx
  0xB6: lambda: LDX(readRAMzpy()),  # LDX zpy
  0xB8: lambda: CLV(),              # CLV imp
  0xB9: lambda: LDA(readRAMaby()),  # LDA aby
  0xBA: lambda: TSX(),              # TSX imp
  0xBC: lambda: LDY(readRAMabx()),  # LDY abx
  0xBD: lambda: LDA(readRAMabx()),  # LDA abx
  0xBE: lambda: LDX(readRAMaby()),  # LDX aby
  0xC0: lambda: CPY(readRAMimm()),  # CPY imm
  0xC1: lambda: CMP(readRAMinx()),  # CMP inx
  0xC2: lambda: NOP(readRAMimm()),  # NOP imm*
  0xC4: lambda: CPY(readRAMzpg()),  # CPY zpg
  0xC5: lambda: CMP(readRAMzpg()),  # CMP zpg
  0xC6: lambda: DEC(getIDXzpg()),   # DEC zpg
  0xC8: lambda: INY(),              # INY imp
  0xC9: lambda: CMP(readRAMimm()),  # CMP imm
  0xCA: lambda: DEX(),              # DEX imp
  0xCC: lambda: CPY(readRAMabs()),  # CPY abs
  0xCD: lambda: CMP(readRAMabs()),  # CMP abs
  0xCE: lambda: DEC(getIDXabs()),   # DEC abs
  0xD0: lambda: BNE(readRAMrel()),  # BNE rel
  0xD1: lambda: CMP(readRAMiny()),  # CMP iny
  0xD2: lambda: HLT(readRAMimm()),  # HLT imm*
  0xD4: lambda: NOP(readRAMzpx()),  # NOP zpx*
  0xD5: lambda: CMP(readRAMzpx()),  # CMP zpx
  0xD6: lambda: DEC(getIDXzpx()),   # DEC zpx
  0xD8: lambda: CLD(),              # CLD imp
  0xD9: lambda: CMP(readRAMaby()),  # CMP aby
  0xDA: lambda: NOP(),              # NOP imp*
  0xDD: lambda: CMP(readRAMabx()),  # CMP abx
  0xDE: lambda: DEC(getIDXabx()),   # DEC abx
  0xE0: lambda: CPX(readRAMimm()),  # CPX imm
  0xE1: lambda: SBC(readRAMinx()),  # SBC inx
  0xE2: lambda: NOP(readRAMimm()),  # NOP imm*
  0xE4: lambda: CPX(readRAMzpg()),  # CPX zpg
  0xE5: lambda: SBC(readRAMzpg()),  # SBC zpg
  0xE6: lambda: INC(getIDXzpg()),   # INC zpg
  0xE8: lambda: INX(),              # INX imp
  0xE9: lambda: SBC(readRAMimm()),  # SBC imm
  0xEA: lambda: NOP(),              # NOP imp
  0xEC: lambda: CPX(readRAMabs()),  # CPX abs
  0xED: lambda: SBC(readRAMabs()),  # SBC abs
  0xEE: lambda: INC(getIDXabs()),   # INC abs
  0xF0: lambda: BEQ(readRAMrel()),  # BEQ rel
  0xF1: lambda: SBC(readRAMiny()),  # SBC iny
  0xF2: lambda: HLT(readRAMimm()),  # HLT imm*
  0xF4: lambda: NOP(readRAMzpx()),  # NOP zpx*
  0xF5: lambda: SBC(readRAMzpx()),  # SBC zpx
  0xF6: lambda: INC(getIDXzpx()),   # INC zpx
  0xF8: lambda: SED(),              # SED imp
  0xF9: lambda: SBC(readRAMaby()),  # SBC aby
  0xFA: lambda: NOP(),              # NOP imp*
  0xFD: lambda: SBC(readRAMabx()),  # SBC abx
  0xFE: lambda: INC(getIDXabx()),   # INC abx
}

def step():
  opcode = nextByte()
  if opcode in instructions:
    instructions[opcode]()
  else:
    NOP() # default to nop
```

C:
```c
void DOP(uint8_t v) {} // needed for 2-byte nops

void TOP(uint16_t v) {} // needed for 3-byte nops

void HLT(uint8_t v) {} // will be implemented later, needed to compile

void step(void) {
  uint8_t opcode = nextByte();
  switch (opcode) {
    case 0x00: BRK(); break;              // BRK imp
    case 0x01: ORA(readRAMinx()); break;  // ORA inx
    case 0x02: HLT(readRAMimm()); break;  // HLT imm*
    case 0x04: DOP(readRAMzpg()); break;  // NOP zpg*
    case 0x05: ORA(readRAMzpg()); break;  // ORA zpg
    case 0x06: ASL_M(getIDXzpg()); break; // ASL zpg
    case 0x08: PHP(); break;              // PHP imp
    case 0x09: ORA(readRAMimm()); break;  // ORA imm
    case 0x0A: ASL_A(); break;            // ASL acc
    case 0x0C: TOP(readRAMabs()); break;  // NOP abs*
    case 0x0D: ORA(readRAMabs()); break;  // ORA abs
    case 0x0E: ASL_M(getIDXabs()); break; // ASL abs
    case 0x10: BPL(readRAMrel()); break;  // BPL rel
    case 0x11: ORA(readRAMiny()); break;  // ORA iny
    case 0x12: HLT(readRAMimm()); break;  // HLT imm*
    case 0x14: DOP(readRAMzpg()); break;  // NOP zpg*
    case 0x15: ORA(readRAMzpx()); break;  // ORA zpx
    case 0x16: ASL_M(getIDXzpx()); break; // ASL zpx
    case 0x18: CLC(); break;              // CLC imp
    case 0x19: ORA(readRAMaby()); break;  // ORA aby
    case 0x1A: NOP(); break;              // NOP imp*
    case 0x1C: TOP(readRAMabx()); break;  // NOP abx*
    case 0x1D: ORA(readRAMabx()); break;  // ORA abx
    case 0x1E: ASL_M(getIDXabx()); break; // ASL abx
    case 0x20: JSR(getIDXabs()); break;   // JSR abs
    case 0x21: AND(readRAMinx()); break;  // AND inx
    case 0x22: HLT(readRAMimm()); break;  // HLT imm*
    case 0x24: BIT(readRAMzpg()); break;  // BIT zpg
    case 0x25: AND(readRAMzpg()); break;  // AND zpg
    case 0x26: ROL_M(getIDXzpg()); break; // ROL zpg
    case 0x28: PLP(); break;              // PLP imp
    case 0x29: AND(readRAMimm()); break;  // AND imm
    case 0x2A: ROL_A(); break;            // ROL acc
    case 0x2C: BIT(readRAMabs()); break;  // BIT abs
    case 0x2D: AND(readRAMabs()); break;  // AND abs
    case 0x2E: ROL_M(getIDXabs()); break; // ROL abs
    case 0x30: BMI(readRAMrel()); break;  // BMI rel
    case 0x31: AND(readRAMiny()); break;  // AND iny
    case 0x32: HLT(readRAMimm()); break;  // HLT imm*
    case 0x34: DOP(readRAMzpx()); break;  // NOP zpx*
    case 0x35: AND(readRAMzpx()); break;  // AND zpx
    case 0x36: ROL_M(getIDXzpx()); break; // ROL zpx
    case 0x38: SEC(); break;              // SEC imp
    case 0x39: AND(readRAMaby()); break;  // AND aby
    case 0x3A: NOP(); break;              // NOP imp*
    case 0x3D: AND(readRAMabx()); break;  // AND abx
    case 0x3E: ROL_M(getIDXabx()); break; // ROL abx
    case 0x40: RTI(); break;              // RTI imp
    case 0x41: EOR(readRAMinx()); break;  // EOR inx
    case 0x42: HLT(readRAMimm()); break;  // HLT imm*
    case 0x44: DOP(readRAMzpg()); break;  // NOP zpg*
    case 0x45: EOR(readRAMzpg()); break;  // EOR zpg
    case 0x46: LSR_M(getIDXzpg()); break; // LSR zpg
    case 0x48: PHA(); break;              // PHA imp
    case 0x49: EOR(readRAMimm()); break;  // EOR imm
    case 0x4A: LSR_A(); break;            // LSR acc
    case 0x4C: JMP(getIDXabs()); break;   // JMP abs
    case 0x4D: EOR(readRAMabs()); break;  // EOR abs
    case 0x4E: LSR_M(getIDXabs()); break; // LSR abs
    case 0x50: BVC(readRAMrel()); break;  // BVC rel
    case 0x51: EOR(readRAMiny()); break;  // EOR iny
    case 0x52: HLT(readRAMimm()); break;  // HLT imm*
    case 0x54: DOP(readRAMzpx()); break;  // NOP zpx*
    case 0x55: EOR(readRAMzpx()); break;  // EOR zpx
    case 0x56: LSR_M(getIDXzpx()); break; // LSR zpx
    case 0x58: CLI(); break;              // CLI imp
    case 0x59: EOR(readRAMaby()); break;  // EOR aby
    case 0x5A: NOP(); break;              // NOP imp*
    case 0x5D: EOR(readRAMabx()); break;  // EOR abx
    case 0x5E: LSR_M(getIDXabx()); break; // LSR abx
    case 0x60: RTS(); break;              // RTS imp
    case 0x61: ADC(readRAMinx()); break;  // ADC inx
    case 0x62: HLT(readRAMimm()); break;  // HLT imm*
    case 0x64: DOP(readRAMzpg()); break;  // NOP zpg*
    case 0x65: ADC(readRAMzpg()); break;  // ADC zpg
    case 0x66: ROR_M(getIDXzpg()); break; // ROR zpg
    case 0x68: PLA(); break;              // PLA imp
    case 0x69: ADC(readRAMimm()); break;  // ADC imm
    case 0x6A: ROR_A(); break;            // ROR acc
    case 0x6C: JMP(getIDXind()); break;   // JMP ind
    case 0x6D: ADC(readRAMabs()); break;  // ADC abs
    case 0x6E: ROR_M(getIDXabs()); break; // ROR abs
    case 0x70: BVS(readRAMrel()); break;  // BVS rel
    case 0x71: ADC(readRAMiny()); break;  // ADC iny
    case 0x72: HLT(readRAMimm()); break;  // HLT imm*
    case 0x74: DOP(readRAMzpg()); break;  // NOP zpg*
    case 0x75: ADC(readRAMzpx()); break;  // ADC zpx
    case 0x76: ROR_M(getIDXzpx()); break; // ROR zpx
    case 0x78: SEI(); break;              // SEI imp
    case 0x79: ADC(readRAMaby()); break;  // ADC aby
    case 0x7A: NOP(); break;              // NOP imp*
    case 0x7D: ADC(readRAMabx()); break;  // ADC abx
    case 0x7E: ROR_M(getIDXabx()); break; // ROR abx
    case 0x80: DOP(readRAMimm()); break;  // NOP imm*
    case 0x81: STA(getIDXinx()); break;   // STA inx
    case 0x82: DOP(readRAMimm()); break;  // NOP imm*
    case 0x84: STY(getIDXzpg()); break;   // STY zpg
    case 0x85: STA(getIDXzpg()); break;   // STA zpg
    case 0x86: STX(getIDXzpg()); break;   // STX zpg
    case 0x88: DEY(); break;              // DEY imp
    case 0x89: DOP(readRAMimm()); break;  // NOP imm*
    case 0x8A: TXA(); break;              // TXA imp
    case 0x8C: STY(getIDXabs()); break;   // STY abs
    case 0x8D: STA(getIDXabs()); break;   // STA abs
    case 0x8E: STX(getIDXabs()); break;   // STX abs
    case 0x90: BCC(readRAMrel()); break;  // BCC rel
    case 0x91: STA(getIDXiny()); break;   // STA iny
    case 0x92: HLT(readRAMimm()); break;  // HLT imm*
    case 0x94: STY(getIDXzpx()); break;   // STY zpx
    case 0x95: STA(getIDXzpx()); break;   // STA zpx
    case 0x96: STX(getIDXzpy()); break;   // STX zpy
    case 0x98: TYA(); break;              // TYA imp
    case 0x99: STA(getIDXaby()); break;   // STA aby
    case 0x9A: TXS(); break;              // TXS imp
    case 0x9D: STA(getIDXabx()); break;   // STA abx
    case 0xA0: LDY(readRAMimm()); break;  // LDY imm
    case 0xA1: LDA(readRAMinx()); break;  // LDA inx
    case 0xA2: LDX(readRAMimm()); break;  // LDX imm
    case 0xA4: LDY(readRAMzpg()); break;  // LDY zpg
    case 0xA5: LDA(readRAMzpg()); break;  // LDA zpg
    case 0xA6: LDX(readRAMzpg()); break;  // LDX zpg
    case 0xA8: TAY(); break;              // TAY imp
    case 0xA9: LDA(readRAMimm()); break;  // LDA imm
    case 0xAA: TAX(); break;              // TAX imp
    case 0xAC: LDY(readRAMabs()); break;  // LDY abs
    case 0xAD: LDA(readRAMabs()); break;  // LDA abs
    case 0xAE: LDX(readRAMabs()); break;  // LDX abs
    case 0xB0: BCS(readRAMrel()); break;  // BCS rel
    case 0xB1: LDA(readRAMiny()); break;  // LDA iny
    case 0xB2: HLT(readRAMimm()); break;  // HLT imm*
    case 0xB4: LDY(readRAMzpx()); break;  // LDY zpx
    case 0xB5: LDA(readRAMzpx()); break;  // LDA zpx
    case 0xB6: LDX(readRAMzpy()); break;  // LDX zpy
    case 0xB8: CLV(); break;              // CLV imp
    case 0xB9: LDA(readRAMaby()); break;  // LDA aby
    case 0xBA: TSX(); break;              // TSX imp
    case 0xBC: LDY(readRAMabx()); break;  // LDY abx
    case 0xBD: LDA(readRAMabx()); break;  // LDA abx
    case 0xBE: LDX(readRAMaby()); break;  // LDX aby
    case 0xC0: CPY(readRAMimm()); break;  // CPY imm
    case 0xC1: CMP(readRAMinx()); break;  // CMP inx
    case 0xC2: DOP(readRAMimm()); break;  // NOP imm*
    case 0xC4: CPY(readRAMzpg()); break;  // CPY zpg
    case 0xC5: CMP(readRAMzpg()); break;  // CMP zpg
    case 0xC6: DEC(getIDXzpg()); break;   // DEC zpg
    case 0xC8: INY(); break;              // INY imp
    case 0xC9: CMP(readRAMimm()); break;  // CMP imm
    case 0xCA: DEX(); break;              // DEX imp
    case 0xCC: CPY(readRAMabs()); break;  // CPY abs
    case 0xCD: CMP(readRAMabs()); break;  // CMP abs
    case 0xCE: DEC(getIDXabs()); break;   // DEC abs
    case 0xD0: BNE(readRAMrel()); break;  // BNE rel
    case 0xD1: CMP(readRAMiny()); break;  // CMP iny
    case 0xD2: HLT(readRAMimm()); break;  // HLT imm*
    case 0xD4: DOP(readRAMzpx()); break;  // NOP zpx*
    case 0xD5: CMP(readRAMzpx()); break;  // CMP zpx
    case 0xD6: DEC(getIDXzpx()); break;   // DEC zpx
    case 0xD8: CLD(); break;              // CLD imp
    case 0xD9: CMP(readRAMaby()); break;  // CMP aby
    case 0xDA: NOP(); break;              // NOP imp*
    case 0xDD: CMP(readRAMabx()); break;  // CMP abx
    case 0xDE: DEC(getIDXabx()); break;   // DEC abx
    case 0xE0: CPX(readRAMimm()); break;  // CPX imm
    case 0xE1: SBC(readRAMinx()); break;  // SBC inx
    case 0xE2: DOP(readRAMimm()); break;  // NOP imm*
    case 0xE4: CPX(readRAMzpg()); break;  // CPX zpg
    case 0xE5: SBC(readRAMzpg()); break;  // SBC zpg
    case 0xE6: INC(getIDXzpg()); break;   // INC zpg
    case 0xE8: INX(); break;              // INX imp
    case 0xE9: SBC(readRAMimm()); break;  // SBC imm
    case 0xEA: NOP(); break;              // NOP imp
    case 0xEC: CPX(readRAMabs()); break;  // CPX abs
    case 0xED: SBC(readRAMabs()); break;  // SBC abs
    case 0xEE: INC(getIDXabs()); break;   // INC abs
    case 0xF0: BEQ(readRAMrel()); break;  // BEQ rel
    case 0xF1: SBC(readRAMiny()); break;  // SBC iny
    case 0xF2: HLT(readRAMimm()); break;  // HLT imm*
    case 0xF4: DOP(readRAMzpx()); break;  // NOP zpx*
    case 0xF5: SBC(readRAMzpx()); break;  // SBC zpx
    case 0xF6: INC(getIDXzpx()); break;   // INC zpx
    case 0xF8: SED(); break;              // SED imp
    case 0xF9: SBC(readRAMaby()); break;  // SBC aby
    case 0xFA: NOP(); break;              // NOP imp*
    case 0xFD: SBC(readRAMabx()); break;  // SBC abx
    case 0xFE: INC(getIDXabx()); break;   // INC abx
    default: NOP(); break;                // default to nop
  }
}
```

Youll notice I added some extra NOPs with an asterisk, as well as some HLT instructions which we havent defined. These are some of the undocumented instructions, and these ones do have some utility in our case. HLT is just going to stop the simulation, which we can use as actual break points in a program. The extra NOPs just up different numbers of bytes depending on the addressing mode. None of these are necessary to the emulator, but i do recommend at least including a HLT instruction as it is very useful when programming.

<hr>

### Execution Loop and Emulation Control

At this point we are actually just about done with the emulator. There are just a couple more things to do to get this up and running. First of all, lets define a couple globals to keep track of the state.

JavaScript:
```javascript
var running = false;
var ips = 100;
var lastTime = 0;
var instAcc = 0;
```

The running variable is exactly what it says. It says whether or not the emulator is actually running. The ips variable stands for "instructions per second", and will be the target number of instructions we want the emulator to run in a second. The instAcc variable is just a counter for the number of instructions ran, and will be used to throttle the execution loop to try and match the ips. And lastTime will help track the time delta within the execution loop. Next:

JavaScript:
```javascript
function run() {
  lastTime = performance.now()/1000;
  running = true;
  loop();
}

function stop() {
  running = false;
}

var HLT = stop;
```

These are pretty self explanitory, run the emulator, stop the emulator, and also create a HLT alias to stop so that we can trigger within a program. And finally, the execution loop:

JavaScript:
```javascript
function loop() {
  if (!running) return;

  var time = performance.now()/1000;
  var delta = time - lastTime;
  lastTime = time;

  instAcc += delta * ips;
  var i = instAcc | 0;
  instAcc -= i;
  while (i--) step();

  setTimeout(loop,0);
}
```

or if you are in node, you can use these instead:

JavaScript:
```javascript
function run() {
  lastTime = Number(process.hrtime.bigint()) / 1e9;
  running = true;
  loop();
}

function loop() {
  if (!running) return;

  var time = Number(process.hrtime.bigint()) / 1e9;
  var delta = time - lastTime;
  lastTime = time;

  instAcc += delta * ips;
  var i = instAcc | 0;
  instAcc -= i;
  while (i--) step();

  setImmediate(loop);
}
```

All together:

<details>
<summary>Summary:</summary>

JavaScript:
```javascript
var running = false;
var ips = 100;
var lastTime = 0;
var instAcc = 0;

function run() {
  lastTime = performance.now()/1000;
  running = true;
  loop();
}

function stop() {
  running = false;
}

function loop() {
  if (!running) return;
  var time = performance.now()/1000;
  var delta = time - lastTime;
  lastTime = time;
  instAcc += delta * ips;
  var i = instAcc | 0;
  instAcc -= i;
  while (i--) step();
  setTimeout(loop,0);
}
```

<hr>

### Illegal Instructions (Overview)

You may have noticed, there are a lot of unfilled spaces in the instruction set we went over. I filled in some of those spaces (the ones with the astrisk), but what happens if the cpu reads an opcode thats in one of those spaces? It turns out that there is some interesting behavior, behavior that we can emulatate (for the most part).

HLT (aka JAM or KIL)

These instructions cause the processor to get stuck and would require a reset on normal hardware. They do use the immediate addressing mode (taking up 2 bytes), although nothing ever gets done with the data.

HLT imm - $02 $12 $22 $32 $42 $52 $62 $72 $92 $B2 $D2 $F2

NOPs

There are several instructions that effectively do nothing. Some of them technically read from memory and thus take up multiple bytes. They are sometimes called DOP (double NOP, 2 bytes) and TOP (triple NOP, 3 bytes).

NOP imp - $1A $3A $5A $7A $DA $EA $FA\
NOP imm (DOP) - $80 $82 $89 $C2 $E2\
NOP zpg (DOP) - $04 $44 $64\
NOP zpx (DOP) - $14 $34 $54 $74 $D4 $F4\
NOP abs (TOP) - $0C\
NOP abx (TOP) - $1C $3C $5C $7C $DC $FC


Some of these instructions end up as the result of the cpu trying to execute two instructions at once. Here are those:

SLO = ASL/ORA\
RLA = ROL/AND\
SRE = LSR/EOR\
RRA = ROR/ADC\
SAX = STA/STX\
LAX = LDA/LDX\
DCP = DEC/CMP\
ISC = INC/SBC

```
      inx   zpg   imm   abs   iny   zpx   aby   abx
SLO   $03   $07         $0F   $13   $17   $1B   $1F
RLA   $23   $27         $2F   $33   $37   $3B   $3F
SRE   $43   $47         $4F   $53   $57   $5B   $5F
RRA   $63   $67         $6F   $73   $77   $7B   $7F
SAX   $83   $87         $8F         $97*
LAX   $A3   $A7   $AB   $AF   $B3   $B7*        $BF**
DCP   $C3   $C7         $CF   $D3   $D7   $DB   $DF
ISC   $E3   $E7         $EF   $F3   $F7   $FB   $FF
* = zpy
** = aby
```

There are still missing spaces, some of which are already covered by the NOPs from earlier. The others though, have more complicated/unpredictable behavior. The first group are the immediate versions of these instructions (besides LAX, which behaves the same as it does for the other addressing modes):

ANC imm - $0B $2B\
ALR imm - $4B\
ARR imm - $6B\
XAA imm - $8B\
AXS imm - $CB\
SBC imm - $EB

SBC is here because $EB actually ends up doing the same thing as $E9 (the normal SBC opcode). ANC has two opcodes because they also do the same thing. Heres what they do in more detail:

ANC imm ('AND with carry')

This is like AND imm followed by ROL acc, except the ROL only sets the flags. Which ends up setting A = A & value, and setting the N Z and C flags as if an ROL was done after.

ALR imm

This is like AND imm followed by LSR acc. Which ends up setting A = A & value, and setting the N Z and C flags as if an LSR was done after.

ARR imm

This is like AND imm followed by ROR acc. Which ends up setting A = A & value, and setting the N Z and C flags as if an ROR was done after. It also sets the V flag to the xor of bit 5 and 6 of the result.

XAA imm

This one is unpredictable, it basically does A = A & X & value, but it also reads garbage data leftover on the data bus, resulting in A = (A | ?) & X & value.

AXS imm (aka SBX)

This sets X = (A & X) - value. It sets flags like CPX.


Finally we have the rest of the instructions in the $90 and $B0 rows. These are various variations of store/transfer instructions.

AHX iny - $93\
TAX aby - $9B\
SHY abx - $9C\
SHX aby - $9E\
AHX aby - $9F\
LAS aby - $BB

We will go into more detail in the implementation.

<hr>

### Illegal Instructions (Implementation)

Starting off easy, we can piggyback alot of these off of already implemented instructions.

JavaScript:
```javascript
function SLO(p) {
  ASL_M(p);
  ORA(RAM[p]);
}

function RLA(p) {
  ROL_M(p);
  AND(RAM[p]);
}

function SRE(p) {
  LSR_M(p);
  EOR(RAM[p]);
}

function RRA(p) {
  ROR_M(p);
  ADC(RAM[p]);
}

function SAX(p) {
  RAM[p] = A & X;
}

function LAX(v) {
  setNZflags(A = X = v);
}

function DCP(p) {
  DEC(p);
  CMP(RAM[p]);
}

function ISC(p) {
  INC(p);
  SBC(RAM[p]);
}
```

Python:
```python
def SLO(p):
  ASL_M(p)
  ORA(RAM[p])

def RLA(p):
  ROL_M(p)
  AND(RAM[p])

def SRE(p):
  LSR_M(p)
  EOR(RAM[p])

def RRA(p):
  ROR_M(p)
  ADC(RAM[p])

def SAX(p):
  RAM[p] = A & X

def LAX(v):
  global A, X
  setNZflags(A := (X := v))

def DCP(p):
  DEC(p)
  CMP(RAM[p])

def ISC(p):
  INC(p)
  SBC(RAM[p])
```

C:
```c
void SLO(uint16_t p) {
  ASL_M(p);
  ORA(RAM[p]);
}

void RLA(uint16_t p) {
  ROL_M(p);
  AND(RAM[p]);
}

void SRE(uint16_t p) {
  LSR_M(p);
  EOR(RAM[p]);
}

void RRA(uint16_t p) {
  ROR_M(p);
  ADC(RAM[p]);
}

void SAX(uint16_t p) {
  RAM[p] = A & X;
}

void LAX(uint8_t v) {
  setNZflags(A = X = v);
}

void DCP(uint16_t p) {
  DEC(p);
  CMP(RAM[p]);
}

void ISC(uint16_t p) {
  INC(p);
  SBC(RAM[p]);
}
```
