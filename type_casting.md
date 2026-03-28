# Background

Recently, I have been working on the practice problems in the famous book "Computer System: A Programmer's Perspective". One of the problems caught my attention and I decided to write a deep dive about it.

The question itself is really simple:
```
// Given x to be any 32 bit signed integer,
// Will it always return true? If not, give an example of x such that it returns false.
x == (int)(float)x
```

That's it.

## Initial thoughts

Well, we know that both an integer and a float in C has 32 bits. And if you are familiar with the IEEE754 floating point number, you know that a floating point number only has 23 bits in the fraction part. Which means, when I cast x to a float, there will be some precision loss. So the answer is clearly FALSE.

Well, then it would be easy to construct a value of x to make the evaluation false. I simply need to find an x with more than 23 significant bits, maybe, the maximum representable value of a 32 bits signed integer. 

## Dry run

```
int x = 0x7fffffff;

// x = 0b 0111 1111 1111 1111 1111 1111 1111 1111

// The next step is to convert it to a floating point number

float f = (float)x;

// x = 1.11 1111 1111 1111 1111 1111 1111 1111 x 2^30
// x = 1.11 1111 1111 1111 1111 1111 1111 1111 x 2^(157-127)
// exp part = 157 = 0b10011101
// frac part = 11 1111 1111 1111 1111 1111 1 | 111 1111  (truncated)
// f = 0b 0 | 10011101 | 1111 1111 1111 1111 1111 111
// f = 0b 0100 1110 1111 1111 1111 1111 1111 1111
// f = 0x 4effffff (This is how I predict f would be)
```

However, when I tested it out using `lldb`, the actual value of `f` is `0x4f000000`.

Can you see my mistake?

## Check out the assembly instruction

Actually at the moment of working on this problem, I am not quite knowing how inside the CPU computes these kind of conversion like magic. Like, what does that single line compiles to? What assembly instructions were used? Does it require multiple steps of bit shifting, and, or, etc.? 

Well, take a look then. I quickly pull out:

```
gcc trytry.c -S trytry.S
```

I was running this on a ubuntu docker container inside a Mac with an ARM chip. And therefore the compiler spits out ARM assembly in which I was not quite understand.

Part of the ARM assembly code
```
main:
.LFB0:
	.cfi_startproc                     ; some unknown flag 
	sub	sp, sp, #16                 ; probably handling a new stack frame
	.cfi_def_cfa_offset 16             ; unknown flag
	mov	w0, 2147483647              ; move the 0x7fffffff into w0
	str	w0, [sp, 4]                 ; str should be store, now the value is stored inside memory
	ldr	s0, [sp, 4]                 ; ldr should be load register? it load the value into s0.
	scvtf	s0, s0                      ; scvtf, should be signed int convert to float!! the key instruction I wanna know
	str	s0, [sp, 8]                 ; store the floating point to memory
	ldr	s0, [sp, 8]                 ; load that back
	fcvtzs	s0, s0                      ; fcvtzs should be float convert  signed int, not sure what the 'z' means. 
	str	s0, [sp, 12]                ; store the result back to memory
	mov	w0, 0                       ; prepare for return 0 at the end
	add	sp, sp, 16
	.cfi_def_cfa_offset 0
	ret
	.cfi_endproc
```

Well, then I check out the ARM manual (https://developer.arm.com/documentation/100076/0100/A64-Instruction-Set-Reference/A64-Floating-point-Instructions/SCVTF--scalar--fixed-point-?lang=en):

> Signed fixed-point Convert to Floating-point (scalar). This instruction converts the signed value in the 32-bit or 64-bit general-purpose source register to a floating-point value using the rounding mode that is specified by the FPCR, and writes the result to the SIMD and FP destination register.

Okay... It mentioned about rounding mode. Apart from truncation is there any other rounding mode??? What the heck is FPCR? Let me check that out as well...

FPCR (https://developer.arm.com/documentation/ddi0595/2020-12/AArch64-Registers/FPCR--Floating-point-Control-Register)

Okay... So it is a 64 bits config for all different kinds of floating point number operations, e.g. how to handle Nan in computation? And the 22-23 bits is what we are interested in:

RMode, bits [23:22]
Rounding Mode control field.
| RMode | Meaning                                 |
| ----- | --------------------------------------- |
| 0b00  | Round to Nearest (RN) mode.             |
| 0b01  | Round towards Plus Infinity (RP) mode.  |
| 0b10  | Round towards Minus Infinity (RM) mode. |
| 0b11  | Round towards Zero (RZ) mode.           |
       
Oh.. so there are 4 different rounding modes. Which one I am using while executing the test problem? I am gonna use a debugger to check that:

```
gcc trytry.c -g -o trytry
lldb trytry
breakpoint set --file trytry.c --line 4. # first line in main()
run

register read fpcr
```

And it shows this:
```
(lldb) register read fpcr
    fpcr = 0x00000000
         = (AHP = 0, DN = 0, FZ = 0, RMode = 0, FZ16 = 0, IDE = 0, IXE = 0, UFE = 0, OFE = 0, DZE = 0, IOE = 0, NEP = 0, AH = 0, FIZ = 0)
```

Okay! So RMode = 0, which means we are using the RN mode.

How does RN mode handle signed integer to float? Let me read the manual again...
(https://developer.arm.com/documentation/ddi0487/maa/-Part-A-Arm-Architecture-Introduction-and-Overview/-Chapter-A1-Introduction-to-the-Arm-Architecture/-A1-5-Floating-point-support/-A1-5-8-Rounding?lang=en#babjdcahg4)

#### Round to Nearest (RN) mode
Round to Nearest rounding mode rounds the exact result of a floating-point operation to a value that is representable in the destination format as follows: 

> If the value before rounding has an absolute value that is too large to represent in the output format, the rounded value is an Infinity. The sign of the rounded value is the same as the sign of the value before rounding.

This is not our case, the representable range of a float is much larger than the signed int.

> If the value before rounding has an absolute value that is not too large to represent in the output format, the result is calculated as follows:

This is our case!

> If the two nearest floating-point numbers bracketing the value before rounding are equally near, the result is the number with an even least significant digit.

This is the Round-to-even mentioned in CSAPP!

> If the two nearest floating-point numbers bracketing the value before rounding are not equally near, the result is the floating-point number nearest to the value before rounding.

Oh, I see, so let me do the math again!

```
x = 1.11 1111 1111 1111 1111 1111 1111 1111 x 2^30    // exact value
x = 1.11 1111 1111 1111 1111 1111 1 | 111 1111 x 2^30  

Find the closest:
x = 10.00 0000 0000 0000 0000 0000 0 | 000 0000 x 2^30  (upper bound, closer to actual value)
x =  1.11 1111 1111 1111 1111 1111 1 | 111 1111 x 2^30  
x =  1.11 1111 1111 1111 1111 1111 1 | 000 0000 x 2^30  (lower bound)

Therefore, x is rounded up to 10 x 2^30 = 1 x 2^31 = 2^(158-127)!
// exp part = 0b 10011110
// frac part = 0

// f = 0 | 1001 1110 | 000 0000 0000 0000 0000 0000
// f = 0100 1111 0000 0000 0000 0000 0000 0000
// f = 0x4f000000
```

This is exactly the value in my test!

## The second surprise

Then the remaining part of the problem is to convert this float back to a signed integer.

But... x was rounded up by 1, and that means x has exceeded the largest representable integer in 32 bit two's complement!

```
f = 2^31 // exact value
f = 0x80000000 (it does not have a representation in 32 bit signed integer!)
```

Then what happens? Let me try to run it in ARM first...

```
int x = (int)f;
// x = 0x7fffffff
```

Hmm, the result becomes the largest representable signed integer in 32 bit... Why? Is this by accident? Let me try another floating point value that is much larger:

```
// x = 0x7fffffff
```

Still the same! It seems the ARM CPU did some value clamping when overflow happens. Remember the `fcvtzs` instruction? Let's check out the manual for ARM...

FCVTZS stands for Floating-point Convert to Signed integer, rounding toward Zero (scalar).

> Round towards Zero rounding mode rounds the exact result of a floating-point operation to a value that is representable in the destination format. The result is the floating-point number in the output format that is closest to and not greater in absolute value than the value before rounding.

Oh, so that's why it gives me the largest representable value in 32 bit signed int, since 0x7fffffff is the closest value in 32 bit signed int that is not greater than 0x80000000!

That's interesting, I wonder if this is also the same in x86_64 machine. Therefore, I installed a cross compiler and checkout the x86_64 version assembly:

```
x86_64-linux-gnu-gcc trytry.c -S -masm=intel trytry_x86_64.s
```

This is part of the x86_64 assembly code
```
main:
.LFB0:
	.cfi_startproc                                   ; unknown
	endbr64                                          ; unknown
	push	rbp                                       ; stack stuff
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	mov	rbp, rsp                                  
	.cfi_def_cfa_register 6     
	mov	DWORD PTR -12[rbp], 2147483647            ; load x
	pxor	xmm0, xmm0                                ; clear the xmm0 register... but this one looks strange, what is it?
	cvtsi2ss	xmm0, DWORD PTR -12[rbp]           ; cvt = convert, si = signed integer, 2 = to, ss = scalar single-precision (float)
	movss	DWORD PTR -8[rbp], xmm0                   ; load the float into memory
	movss	xmm0, DWORD PTR -8[rbp]                   ; load the float back to the register
	cvttss2si	eax, xmm0                          ; cvt = convert, t = ??, ss = float, 2 = to, si = signed integer, what is t???
	mov	DWORD PTR -4[rbp], eax                    ; store result to memory
	mov	eax, 0                                    ; return 0
	pop	rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
```

It turns out that the cvtsi2ss and cvttss2si are the instructions that we care about. 

`cvtsi2ss`: turns out to be same as the ARM `scvtf`, in x86_64, there is also a register that pick the rounding mode.
`cvttss2si`: what the heck is the `t` in the middle? Let me search this in the x86_64 CPU manual (https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html).

cvttss2si stands for:
> Convert with truncation a scalar single precision floating-point value to a scalar double-word integer.

So `t` means truncation. It simply means round towards zero. But that provides no information about how this instruction handles overflow cases!

Let me search through the manual for this instruction:

> Conversion to integer when the value in the source register is a
NaN, ∞, or exceeds the representable range. ==> Return the integer Indefinite.

So it will give me the integer *Indefinite*, what is it? Let me search that in the manual:

```
0x80000000
```

Ah hah! That coincides with what is mentioned in the CSAPP book:

> From float or double to int the value will be rounded toward zero. For example, 1.999 will be converted to 1, while −1.999 will be converted to
−1. Furthermore, the value may overflow. The C standards do not specify a fixed result for this case. Intel-compatible microprocessors designate the bit pattern [10 . . . 00] (TMinw for word size w) as an integer indefinite value.

This is how I get a x86_64 binary on a mac with ARM chip:
```
gcc -arch x86_64 -O0 -g trytry.c -o trytry_x86
lldb trytry_x86
```

I can execute the x86_64 binary as my mac support x86 emulation with the rosetta 2 layer.

## C standard

From this experiment, we can tell the C standard did not mentioned about what should be returned when a floating point number is being casted back to a signed integer and overflow occured. So what is the lesson learnt? That is, never do such things that may result in undefined behavior! 

Hope you found this reading fun! I am Kelvin, a guy learning coding.