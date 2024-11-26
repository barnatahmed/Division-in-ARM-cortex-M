# How buffer sizes should be chosen 

## 1. Introduction
A ring buffer, or circular buffer, is a data structure used to emulate a wrap-around in memory. Ring buffers are contiguous blocks of memory that act as if they are circular, with no beginning or end, instead of 1D. These buffers are used with ADC values reading, streaming, gateways and lot more... However choosing the right size of the buffer is crucial and more nuanced than developpers sometimes think. This topic is mysteriously intertwined with the fundamental of division.
The ARM Cortex-M series of microcontrollers presents unique challenges in division operations, reflecting the complex trade-offs between computational efficiency and hardware constraints in embedded systems. Some cortex M does not support a "div" instruction as there is no hardware support inside. so the questions arises, What are the software solution that these processors rely as a decent alternave? How This would affect the choice of a circular buffer size?

## 2. No Div instruction
### 2.1 Div instruction support
Until 2004, there was no hardware support for division instruction for ARM processors. With the introduction of the cortex brand series, The pair of instructions "SDIV/UDIV" are finally introduced to do the job. Focusing only on ARMv6-M and ARMv7-M series, This is the list of processors that support these instructions in the thumb instruction set. 
| Processor     | Division in Thumb instruction|
| ---      | ---       |
| Cortex M7 |     Yes     |
| Cortex M4 |     Yes     |
| Cortex M3 |     Yes    |
| Cortex M1 |     No     |
| Cortex M0 |     No     |
| Cortex M0+ |    No      |

### 2.2 How these instructions work
This is the general form of the instruction:

SDIV{cond} {Rd,} Rn, Rm  ( Rd = Rn/Rm )

UDIV{cond} {Rd,} Rn, Rm

SDIV performs a signed integer division of the value in Rn by the value in Rm. Whereas, UDIV performs an unsigned integer division of the value in Rn by the value in Rm. The result in both shall be stored in Rd ( in simple words: Rd = Rn/Rm )
For both instructions, if the value in Rn is not divisible by the value in Rm, the result is rounded towards zero.

Dividing by zero will cause a UsageFault/hardfault and the UFSR.DIVBYZERO bit indicating the reason for the fault.

One drawback about it is that it might consume between 2-12 clock cycles depending on the number of leading ones and zeroes in the input operands. In fact, ARM cortex could possibly find a way to optimize the computations depending on the input values. That is why some arm compilers won't generate this instruction in some cases under certain conditions.

The most important question is, How division is calculated in Cortex with DIV hardware support?

## 3. Fixed point multiplication
Divsion must never be ommitted, if there is no division hardware support, the compiler would in this case generate "a list of instructions" as a software solution tackling this challenge. The idea was to find a way to transform the division operation into a multiplication indroducing a clever solution that might be refrenced as "fixed point multiplication". ( At first it might be obvious, dividing by 5 is the same as multiplying with 0.2)

### 3.1 Fixed point representation
A “binary point” can be created using our binary representation and the same decimal point concept. A binary point, like in the decimal system (just the same binary representation we are used to), represents the coefficient of the expression 2^0 = 1. The weight of each digit (or bit) to the left of the binary point is 2^0, 2^1, 2^2, and so forth. The binary point’s rightmost digits (or bits) have weights of 2^-1, 2^-2, 2^-3, and so on.

![image](https://github.com/user-attachments/assets/beed4090-297f-4a2c-9e48-7802a974c694)

let us tackle this concept with several examples
````
26.5
= 16 + 8 + 2 + 0.5
= 1 * 2^4 + 1 * 2^3 + 0 * 2^2 + 1 * 2^1 + 0* 2^0 + 1 * 2^-1
=11010.1
````
````
7.625
= 4 + 2 + 1 + 0.5 + 0.625
= 1 * 2^2 + 1 * 2^1 + 1* 2^0 + 1 * 2^-1 + 1 * 2^-3
= 111.101
````
### 3.2 Drawnack of fixed point representation
As much as as this represntation seems easy and intuitive, it has some limitations. Some fractional numbers can't be represented without loosing precision. for example try solving the number 0.1 (decimal). The binary fractional part tend to infinity. Therefore in numerical systems, The CPU would truncate some bits so the number will fit in memory and that is what loosing precision is.
````
0,1 (decimal) = 0.000110011001100110011... (repeating) | in ARM: it would be stored as 0.000110 whilch is approximately 0,09375
```` 

### 3.3 Fixed point multiplication (ARM example)
Let us take a code example in arm cortex M0 compiled with arm-none-eabi-gcc. We added the "naked" attribute to eliminate the prologue and epilogue sequences and focus only on our useful part.
````
__attribute__((naked)) unsigned int div_in_arm(unsigned int num) 
{
    return num / 9U;
}
````
This is the assembly code generated with no optimization for cortex M4 and M0 (similar output)
````
div_in_arm:
        mov     r3, r0
        ldr     r2, .L3
        umull   r1, r3, r2, r3
        lsr     r3, r3, #1
        mov     r0, r3
.L3:
        .word   954437177
````
Breaking this mythical calculation, we can clearly see the UMULL instruction suggesting the operation of division to transform into a multiplication.

Dividing with 9 is the same as multiplying with 1/9.
1/9 is the real number : 0,111111 (infinite ones)
The fixed point representation of 1/9 after fitting in ARM arch is :
````
1/9 = 0.000111 ( real number is 0.109375 )
````
ARM would try to perform this multiplication: num * 0.000111 in binary. However this is cannot be done with floats. The MUL instructions require integers and that the solution is all about.
Instead of multiplying with 0.000111, we will multiply 0.000111 shifted left by 33 times, the result is the shifted right 33. Interting zeros right just so the number would fit to be integer. 
First step: generating the magic number:
The compiler will generate the magic number added by 1 to componsate the value we lost due to the imprecision.
````
round(1/9  * 2^33) +1 ~ round( 0.000111111 << 33 ) +1 =  954437177
````
Second step: Load the input number and magic number in CPU registers
````
r3 = num
r2 = 954437177
````
Third step: multiply the input value "num" wit our magic number:
UMULL is unsigned multiplication with 32*32 bits to 64 bits
````
umull   r1, r3, r2, r3
// r1 = lower 32 bits of r2*r3
// r3 = upper 32 bits of r2*r3
````
Fourth step: reshifting by 33to gain the actual result oh the division
ARM has otptimzed this part to by taking the upper 32 bits of the result and simply shifting it by 1.
````
lsr     r3, r3, #1
mov     r0, r3
// resutlt >> 33 same as Upper 32 bits >> 1
// put final result to r0 
````

### 3.3 Division by a multiple of two (ARM)
Dividing with a power of two is relief in the processor as it would simply shift it right: 
````
__attribute__((naked)) unsigned int div_in_arm_pow_2(unsigned int num) 
{
    return num / 1024U;
}
````
````
mov     r3, r0
lsr     r3, r3, #10
mov     r0, r3
// 1024 = 2^10 
````
## 3.4 Consequences:
After testing different examples, we noticed:
1. magic number's multiplier ( 2^33 ) may vary according the input's type.
2. The higher the magic number, the better precision in the result for higher input values. 

## 4. Buffer sizes must be multiple of 2:
The matter we have been presenting would directly affect the usage of ring buffers or circular buffers. In fact using circular buffers is related with division.
![image](https://github.com/user-attachments/assets/e7f80d52-f8cb-4935-a6a0-9744dfbaf65f)

If the index exceeds the size of the buffer, we would find the new index by taking the remainder of the division of the index/buffer_size
### 4.1 Circular buffer index
The mod operation (%) is related to the division operation. ARM compiler solves the mod operation following this formula:
````
a % b = a - (a/b)*b. a/b is calculated via the fixed point multiplication explained earlier. 
````
Let us retake the example with a mod operation.
````
#define BUFFER_SIZE     15U
__attribute__((naked))  unsigned int update_index(unsigned int index) {
    return index % BUFFER_SIZE; }
````
````
        mov     r1, r0
        ldr     r3, .L9
        umull   r2, r3, r1, r3
        lsr     r2, r3, #3
        mov     r3, r2
        lsl     r3, r3, #4
        sub     r3, r3, r2
        sub     r2, r1, r3
        mov     r3, r2
        mov     r0, r3
.L9:
        .word   -2004318071
````
First, it calculates "index / BUFFER_SIZE" using the fixed point multiplication method. Then, it makes the substraction "index - index/BUFFER_SIZE" to extract the remainder
### 4.2 buffer size must be mulitiple of two:
If we choose a buffer size a multiple of two, the result is much simpler and faster:
````
#define BUFFER_SIZE     16U
update_index:
        mov     r3, r0
        and     r3, r3, #15
        mov     r0, r3
````
What happened is that ARM has optimized this code in way that we don't even need multiplications or divisions. the mod opration is transformed to this statement:
````
__attribute__((naked))  unsigned int index_wrap(unsigned int num) {
    return num & (BUFFER_SIZE - 1);
}
````
The power of 2 size requirement for circular buffers enables efficient wrapping using bitwise AND instead of mod (%) operation. Here's why:

Binary representation:

````
BUFFER_SIZE     = 16  = 0b10000
BUFFER_SIZE - 1 = 15  = 0b01111  (This becomes our mask)

Let's wrap 20 to valid index:
20        = 0b10100
15        = 0b01111 (mask)
15 & 20   = 0b00100 = 4  -> Wrapped to index 4!   ( 20 % 16 = 4 )

````
# 5. conclusion:
In ARM processsors division is quite challenging and tricky and it must dealt with caution especially when it comes to circular buffers, choosing their size a power of two is an explicit solution to avoid the division instruction or its alternatives. 

