# Java Explicit Vectorization

- [Part 1 - background (written by dummies, for curious cats)](#part-1---background-written-by-dummies-for-curious-cats)
  - [What is _'Vectorization'_?](#what-is-_vectorization_)
    - [Bullshit usecase](#bullshit-usecase)
      - [Vivisection](#vivisection)
    - [Soothing light at the end the tunnel](#soothing-light-at-the-end-the-tunnel)
      - [Enter the SIMD](#enter-the-simd)
        - [The nail to SIMD's Hammer](#the-nail-to-simds-hammer)
        - [Soothing light at the end the tunnel](#soothing-light-at-the-end-the-tunnel)
        - [What's my name?](#whats-my-name)
- [Part 2 - let's talk about Java](#part-2---lets-talk-about-java)
  - [Auto Vectorization(AV)](#auto-vectorizationav)
    - [-- Interlude --](#---interlude---)
    - [What abouts the Java?](#what-abouts-the-java)
  - [Explicit Vectorization(EV)](#explicit-vectorizationev)
    - [Prelude](#prelude)
    - [Enter the Dragon](#enter-the-dragon)
    - [Show me the money](#show-me-the-money)
      - [Prerequisites](#prerequisites)
      - [Hello, 世界](#hello-)
      - [Vanilla](#vanilla)
      - [With Nuts](#with-nuts)
        - [Species](#species)
        - [Read input into SIMD register(s)](#read-input-into-simd-registers)
        - [Do operation(s) on SIMD register(s)](#do-operations-on-simd-registers)
        - [Write SIMD register(s) back to input](#write-simd-registers-back-to-input)
        - [Hello hello](#hello-hello)
      - [Benchmark Vanilla vs Nuts](#benchmark-vanilla-vs-nuts)
        - [Let sleeping dogs lie](#let-sleeping-dogs-lie)
        - [Monkey Wrench Drop - where AV fails and EV thrives](#monkey-wrench-drop---where-av-fails-and-ev-thrives)
      - [Real world example](#real-world-example)
        - [Problem statement](#problem-statement)
        - [Vanilla MinMax() version](#vanilla-minmax-version)
        - [SIMD MinMax() version](#simd-minmax-version)
          - [Elephant in the room](#elephant-in-the-room)
  - [Epilogue](#epilogue)
- [References](#references)

## Part 1 - background (written by dummies, for curious cats)

### What is _'Vectorization'_?

Without beating about the bush, does Vectorization, explicit or otherwise, have anything to do with the
fantabulous `java.util.Vector` class? Well, not much.

Allow me to explain with this very long example(there is probably some proper etymology that goes along with this
answer, but I can't be bothered to look for it).

#### Bullshit usecase

Let's say you are writing an image editing tool similar to Photoshop. One of the functions you want to implement is
adjusting the brightness. In this particular example:

- Your input is an array of 3 bytes of `RGB` values per pixel, meaning a `1920 x 1080` picture would
  have `3 x 1920 x 1080 = 6220800` bytes.
- Increasing brightness is basically adding 1(or more) to each RGB value per pixel

So how would we do that?

```java
  for (int i = 0; i < 6220800; i++) {
      rgb[i]++;
  }
```

We could probably do something fancy like unrolling the loop(eliminating some bounds checking) to speed this up:

```java
  for (int i = 0; i < 6220800; i += 2) {
      rgb[i + 0]++;
      rgb[i + 1]++;
  }
```

The reason we have to do it this way, is because of real world physics. Your fancy shmancy CPU is basically
a `digital logic gate` on steroids, that can do A LOT of ~~dumb~~ simple things really REALLY fast. 

##### Vivisection

To give you an idea of what "simple" means, if you want to add 2 numbers you have to:

- 1st simple operation(`mov reg1, num1`): load first number into register(small memory inside CPU)
- 2nd simple operation(`add reg1, num2`): add second number to register
- 3rd simple operation(`mov num1, reg1`): write register back to first number

That is 3 CPU operations for something that seems rather innocent. The good news is, is that your CPU(depending on
speed, and the type of ops) can do millions of these operations per second.

Let's say our CPU can do a million operations per second. Let us do a simplified dissection of the first example from
above:

- 3 instructions: save current state
- 3 instructions: setting up the loop
    - 3 instructions: checking/incrementing the loop
    - 3 instructions: adding 1 to a value(`mov, add, mov`)
- 3 instructions: restore saved state

So about 6 instructions per loop iteration. Multiply that by the size of our loop `6 x 6220800 = 37324800`, which when
divided by a million operations per second results in `37324800 / 1000000 = 37` seconds to increase the brightness of
a `1920 x 1080` picture(on a fundamental level, computers have a mechanical nature to them).

If you think this sounds ridiculous dear reader, know that it is the truth, albeit half the truth. Your computer truly
does operate in this way, but that does not mean that this is the **only** way it can operate. If changing every pixel
on a screen took 37 seconds, then neither videogames nor videoplayback would be possible on a computer.

#### Soothing light at the end the tunnel

It might surprise/scare you to know, that a lot of the time, software boils down to loads of repetitive operations like
the ones we described above(disclaimer: this depends of course, but it's our story, so it is true). The point being, our
example is not an edge case. Meaning that a solution is needed, which in turn results in companies inventing ways to
address the ~~business opportunity~~ problem.

##### Enter the SIMD

Again, etymology and history are available on the Wikipedia, so forgive me for skipping them. SIMD stands for **Single
Instruction Multiple Data**. Let's put a pin in that, and recap our problem.

###### The nail to SIMD's Hammer

- In our example, it takes us `37324800` instructions to add 1 to every element of our array.
- The instructions are very repetitive
- The instruction sequences are also very repetitive
- Almost half the instructions are **checking/incrementing the loop**
- And finally, the array elements do not depend on each other

Okay, so common sense would dictate that our program would run faster in 1 of 2 cases:

- Less instructions
- Faster instructions - we won't go into this, but some CPU instructions are faster than others. For this example, we
  will assume we are using the fastest instructions.

So less instructions is the key. We previewed such a technique in the 2nd example above -loop unrolling-, which would
reduce our need to check the loop after every element update. So if we update 5 elements at a time, we
eliminate `4 x 3 loop check instructions` for every `5 updates`, which results in a total of `18/5 x 6220800 = 22394880`
instructions. That's nice, so why not go further and do 100 or 1000 elements at a time? Well, the short answer is, we
forgot to mention that `not all CPU instructions are as fast as each other`. Okaaaaay, but why is this important? Well,
even though a `loop checking` iteration has:

1. 3 instructions for checking the loop
2. And 3 instructions for incrementing the element

The second 3 instructions take 50% slower, so take 75% of the total time it takes to do an iteration(again, this might
not be accurate, but it makes for a better story). So **even** if you manage to remove all of the `loop checking`
instructions, the program will **never** be twice as fast.

Also, friendly neighbourhood compiler is *_usually_* smart enough to know if and how much loop unrolling is necessary :)

###### Soothing light at the end the tunnel

So instead, we want the copying, adding, and copying back parts to be faster. But how, we are already using the fastest
instructions available?

`forgive the delay in reaching this point, but trust me when I say this makes much more sense with some context.`

This is where SIMD comes in. SIMD allows us to execute *special instructions* that do a simple action for a **sequence**
of data. How much of that last part sounds familiar?

Okay, so in our first example we started out by copying the data byte to the CPU register by invoking a `mov`
instruction. Is there a SIMD version for this operation? Yessur! `vmovdqu` to the rescue.

So what does `vmovdqu` do? Well, to quote the official Intel
manual[1] `"Move unaligned packed byte integer values from xmm2/m128 to xmm1 using writemask k1"`. To translate that
into mere mortal
terms `"Move a SEQUENCE of bytes from anywhere memory to SPECIAL registers, and some other stuff we don't care about yet"`
. Exactly what we are looking for then!
But what are these "special registers" they talk about.

Well, like we briefly mentioned at the top, a register is a memory block inside the CPU. What we didn't mention is that
registers are traditionally very small. Like `1 or 2 bytes` small. Which of course, doesn't help when we want to copy a
sequence of bytes into a register. That's why these SIMD instructions came hand in hand with the special registers. A
special register can usually hold about `8 times` what a normal register can hold(this depends on the CPU). Great! So we
can copy `8 values` per operations!

But does this mean that this **single instruction** is **8 times faster** than doing 8 ordinary instructions? The short
answer is, alas no. The good news is, it about **60% faster**!

So if all our program did was copy bytes to the CPU registers, by moving to the SIMD flavour, we gain 60%. Meaning 100
seconds becomes 40.

Furthermore, if this assumption holds for the rest of the operations of our simple program, then our runtime
becomes `37 * 0.4 = 14.8` seconds.

###### What's my name?

This phenomenon, is called Vectorization. The term stems from Vector Processors(see wikipedia), which operate on
Mathematical Vectors. And if you remember your linear algebra, a Vector is basically something like `[1 2 3 0]`, which
I'm sure your Java laden eyes mistook this for an Array. Fear not, it basically is an array, only on a much lower level.

## Part 2 - let's talk about Java

So before we get into what any of this has to do with Java, know that there 2(or maybe more?) different types of
Vectorization:

1. Auto Vectorization(AV)
2. Explicit Vectorization(EV)

Of course we recognize EV from the title of this rant, so we will talk about it extensively. But to get to that, we need
to talk about AV first.

### Auto Vectorization(AV)

So the stuff we talked about in the previous section was more or less on Assembly/Machine Code level. But naturally,
most of us mere mortals program in higher level languages. So how does my `for loop` become Machine Code?
                                                      
We all know this one, the compiler does it for us.

But...is that all a compiler does? Well, without going too much into naming semantics, almost all modern compilers
optimize code. 

Optimizations come in many shapes and forms, but one of the most basic ones is choosing the best `opcode/instruction`
for our operation. This might sound like a non-optimization requirement of a compiler, but please remember that some
CPU's have hundreds upon thousands of `instructions`, and any given operation usually has a couple of variations that
are best used under different conditions.    

For example, addition in Assembly is done by using the `add reg1, reg2` instruction. Pretty straight-forward. However,
in the case of adding 1 to a loop index for instance, a smart compiler will most likely use the `inc reg`(increment)
instruction. While the speed gain is usually negligible, your code becomes smaller by saving the additional bytes[2].

#### -- Interlude --

So again, forgive the long-winded introduction, but as usual, it's easier to relate points to each other.

If we reduce the fact that SIMD instructions are simply (fancy)CPU instructions, and add that to the fact that an
Optimizing Compiler chooses the best `opcode/instruction` for the job, then we might conclude that the Compiler might
also choose a SIMD instruction, if the situation arises. And we would be correct.

And this process my dear comrades, is called: Auto Vectorization(https://youtu.be/y8Kyi0WNg40). 

Does this mean my trustworthy compiler will Vectorize all my code?
As a wise man once said...it depends.

To explain this, we will need to quote a different wise man, who once said that compilers are basically "stupid regex
pattern matching engines"(sorry, can't remember the source). Meaning, like everything your compiler does, it only does
so under very specific circumstances. So while the magical compiler might rewrite the above loop to use SIMD, it doesn't
take a lot of deviation for the same wizardly compiler to skip it.

Non-SIMD example:

 ```java
  for (int i = 0; i < 6220800; i++) {
      if (i > 0)
          rgb[i]++;
  }
```

Granted that you wouldn't write code like this(on purpose), it is still baffling that something like this would keep us
from our deserved Vectorization goodness ¯\\_(ツ)_/¯

#### What abouts the Java?

Okay then, back to Java reality with the million dollar question; ~~to be or not to be~~ what about Java? Well,
considering that Java compilers don't actually compile to Assembly/Machine Code T_T, we are sorry to say that...the JVM
architects/developer got you covered and already implemented this in the JVM portion for your pleasure(we're not worthy).

So yeah, Java supports a substantial subset of Auto Vectorization[3]. But like everything in JavaLand that concerns
real-life hardware, there is an added layer of indirection that needs to be taken into consideration.

If you remember, the Java code lifecycle towards the CPU is an arduous one:

- Interpreter
- C1
- C2

As such, the AV optimizations are usually only activated from the latter part of C1 onwards(downwards?).

Meaning(among other things), that very short lived pieces of code that don't reach C2 would unfortunately not be
considered AV candidates. This on top of the compiler level requirements for a piece of code(such as a simple loop) to
be Vectorizable.

### Explicit Vectorization(EV)

#### Prelude

We've always been taught the magic words `"the compiler is smart enough to..."`, which in turn would mean that if a
compiler fails to do whatever it is we expect of it, then the 'fault' lies with layer 8.

The point being that one way or another, the compiler has to recognize certain rules before it can apply an
optimization. As we will see, this is sometimes very difficult to do with our code.

So instead, a lot of compilers provide so called **Compiler Overrides** that in some cases advise the compiler to take a
closer look at a piece of code with regards to the optimization named by the **override**, while in others the compiler
heuristics are completely ignored, and the optimization(or deopt in some cases) is simply performed.

Even though Java doesn't have the traditional Compiler structure, it still maintains the spirit of being overridable in
certain areas.

#### Enter the Dragon

*Even though a lot of this applies to areas outside of Java, we will try to focus on Java only from this point on.*

We have finally reached the meat of the rant. *Explicit Vectorization(**EV**).*

Nutshell: <ins>where *AV* is an optimization provided by the compiler, *EV* is the override the programmer provides to force
the compiler to Vectorize/SIMD-ify the code.</ins>

#### Show me the money

Let's give it a try then.

##### Prerequisites

- JDK 16+
- Add command line argument `--add-modules=jdk.incubator.vector`(this won't be necessary once the Vector API exits
  incubation status)
- x64 or ARM AArch64 CPU(as of JDK16, only these architectures are properly supported)
- AVX support(if you have a CPU from the last decade[4], then just assume you have this)

##### Hello, 世界

Let's take our increasing brightness example from above.

##### Vanilla

```java
static void increaseBrightness(int[] rgb) {
    for (int i = 0; i < 6220800; i++) {
        rgb[i]++;
    }
}
```          

##### With Nuts

Okay, so before going on, I would like to *warn* ye with an important takeaway from thishere rant, which is "magic is
not cheap".

###### Species

That being said, the first thing you need to know is the `vector.VectorSpecies` of your operation.

The magnitude of concurrent operations you can execute are determined by 2 things:

- the register width of our CPU
- the width of the data type we are trying to manipulate

The way this works is as follows:

- if your CPU only supports AVX, that means the widest register it has `XMM#` is `128 bits` wide.
- in our example `rgb[]` is an int array, meaning the data type is `int` which has a width of `4 bytes` or `32 bits`.

So by dividing `128 / 32 = 4`, meaning we can do operations on 4 ints at a time.

```java
final vector.VectorSpecies<Integer> species=vector.IntVector.SPECIES_128;
```

Sweet, but do I need to do all these tedious calculations everytime I want to do me some vectors?

Luckily, no. The API contains a couple of convenience constants that do this for you.

```java
final vector.VectorSpecies<Integer> species = vector.IntVector.SPECIES_PREFERRED;
```

###### Read input into SIMD register(s)

So next, we have to transform our plain old `int[]` into the Vector version. Or, in other words, move the data into the
special SIMD registers.

The funny thing about this part though, is that it seems very counterintuitive from the performance point of view. I
mean, we are literally copying the data into smaller chunks, instead of iterating over an array?? At the very least you
use double the memory size. 

The problem is, our naïveté assumes that an array stays where it is, but know this, SIMD or not, a CPU only deals with
registers.

```java
final int offset = 0;
IntVector vector = IntVector.fromArray(species, rgb, offset);
```

Here we create a _Vector_ of the type and width we determined with the `species` before.

Meaning, if `species == 128 bytes wide`, and we are dealing with `integers`, then `vector` will contain the
first `4 ints` from the `rgb` array, at the specified `offset`.

###### Do operation(s) on SIMD register(s)

Now that the data is where we want it to be, we can unleash our CPU prowess upon it!!!

As you will no doubt discover when you press ctrl + space on `vector.*`, there's a lot of fun things you can do with the
vector. These of course are but a sane subset of SIMD, and instead of whining about it, be grateful(and send John Rose
& Paul Sandoz a gift basket).   

```java
IntVector result = vector.add(1);
```

This step is self-explanatory.

###### Write SIMD register(s) back to input

```java
result.intoArray(rgb, offset);
```

And finally, we write the updated values back to the array. The way we do it here isn't really the 'proper' way, but it
works nonetheless. By proper, I mean writing the values back to the input array. It's much better to write to an
intermediate array(as we might explain later). 

###### Hello hello

This is what our end result looks like:                                                      

```java
final VectorSpecies<Integer> species = IntVector.SPECIES_PREFERRED;
int offset = 0;
for (offset = 0; offset < rgb.length; offset += species.length()) {
    final IntVector vector = IntVector.fromArray(species, rgb, offset);
    final IntVector result = vector.add(1);
    result.intoArray(rgb, offset);
}
```

A casual observer might...observe, that we are still looping over our array. This is a necessary evil. Computers are
magical, but not THAT magical. Maybe someday there will be a single instruction that operates on any length of data.
Until that day though, loops rule! 
                            
And of course this loop might have a tail/remainder, but we will leave it up to your imagination to implement this.

##### Benchmark Vanilla vs Nuts

Is the SIMD version faster or slower? Good question. To be honest, I expected it to be slightly slower because I assumed
the `Vector` API had some slight overhead compared to the compiler provided `AV`.

Seems I was only half right though, as in it is indeed **a little** slower, but not for the reasons I thought.

So running both versions for 1000 iterations(after warmup) yields the following results:

|vanilla      |simd         |
|---          |---          | 
|1249416200ns |1295295700ns |

That's a difference of ~45ms in favor of the vanilla loop.

Looking at the assembly code for both, we note one major difference; the vanilla version unrolls 8 iterations of the
loop, while the SIMD version only unrolls 4.

```nasm
;vanilla                                                                ||          simd
0x0000020d44a0c9b0:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+rbx*4+0x10]     ||          0x0000020d44a0e020:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x10]
0x0000020d44a0c9b7:   vmovdqu YMMWORD PTR [r12+rbx*4+0x10],ymm1         ||          0x0000020d44a0e027:   vmovdqu YMMWORD PTR [r12+r8*4+0x10],ymm1
0x0000020d44a0c9be:   movsxd r8,ebx                                     ||          0x0000020d44a0e02e:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x30]
0x0000020d44a0c9c1:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x30]      ||          0x0000020d44a0e035:   vmovdqu YMMWORD PTR [r12+r8*4+0x30],ymm1
0x0000020d44a0c9c8:   vmovdqu YMMWORD PTR [r12+r8*4+0x30],ymm1          ||          0x0000020d44a0e03c:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x50]
0x0000020d44a0c9cf:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x50]      ||          0x0000020d44a0e043:   vmovdqu YMMWORD PTR [r12+r8*4+0x50],ymm1
0x0000020d44a0c9d6:   vmovdqu YMMWORD PTR [r12+r8*4+0x50],ymm1          ||          0x0000020d44a0e04a:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x70]
0x0000020d44a0c9dd:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x70]      ||          0x0000020d44a0e051:   vmovdqu YMMWORD PTR [r12+r8*4+0x70],ymm1
0x0000020d44a0c9e4:   vmovdqu YMMWORD PTR [r12+r8*4+0x70],ymm1          ||          0x0000020d44a0e058:   add    r8d,0x20
0x0000020d44a0c9eb:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0x90]      ||          0x0000020d44a0e05c:   cmp    r8d,edi
0x0000020d44a0c9f5:   vmovdqu YMMWORD PTR [r12+r8*4+0x90],ymm1          ||          0x0000020d44a0e05f:   jl     0x0000020d44a0e020
0x0000020d44a0c9ff:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0xb0]      ||        
0x0000020d44a0ca09:   vmovdqu YMMWORD PTR [r12+r8*4+0xb0],ymm1          ||        
0x0000020d44a0ca13:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0xd0]      ||      
0x0000020d44a0ca1d:   vmovdqu YMMWORD PTR [r12+r8*4+0xd0],ymm1          ||      
0x0000020d44a0ca27:   vpaddd ymm1,ymm0,YMMWORD PTR [r12+r8*4+0xf0]      ||      
0x0000020d44a0ca31:   vmovdqu YMMWORD PTR [r12+r8*4+0xf0],ymm1          ||
0x0000020d44a0ca3b:   add    ebx,0x40                                   ||
0x0000020d44a0ca3e:   cmp    ebx,ecx                                    ||
0x0000020d44a0ca40:   jl     0x0000020d44a0c9b0                         ||         
````

Interesting...or is it T_T

###### Let sleeping dogs lie

Warning: comparing EV results to AV is not necessarily useful in the real world, since AV can easily deviate with a
simple modification, while the point of EV is to have consistent compiler output. So while this might be fun, and very
educative...don't do it.

So I should have ignored this 50ms discrepancy, but I just couldn't.

This one is a long story (that I might write up later), but the gist of what we found thus far, is that EV version
translated to bigger instructions, which are double the size, so the C2 compiler ends up unrolling half as much. 

Also...while the EV version is a tad slower, it was in fact Vectorized, so in a sense, EV works perfectly. Loop
unrolling is a different kind of optimization, that might have failed for a completely different(and unrelated) reason.

The assembly snippet above rather the result, not the cause. For now, just accept this inconsistency, and if
you're interested, check back for a (possible)follow up rant explaining why. To be continued...

###### Monkey Wrench Drop - where AV fails and EV thrives

So why even show or mention the assembly/unrolling snafu you ask? Well, showing off is one part of it. But more
importantly, to show off the power of EV.

Let's tweak the examples above a little, by making the `1` constant a bit more dynamic. Something like:

```java
static void increaseBrightness(int[] rgb) {
  final double val = Math.random() * 100;
  for (int i = 0; i < 6220800; i++) {
    rgb[i] += getValue();
  }
}

private static int val = -1;
private static int getValue() {
  if (System.currentTimeMillis() > 0 && val < 0) {
    val = (int) (Math.random() * 100);
  }
  return val;
}
```

Purists might think this is cheating, but to prove this specific point, I would daresay it is an acceptable cheat.

So what are we doing here, well, we are tricking the compiler into believing that `getValue()` does not **always**
return the same value(even though it does). By doing so, the compiler declines to Auto Vectorize the loop :(

On the other hand, the EV variant is still vectorized, making the SIMD version in our benchmark(1000 iterations)
about `1 second` faster!

To offset the number of times `getValue()` is called, we hacked the SIMD version as follows:
```java
    final IntVector result = vector.add(getValue() + getValue() + getValue() + getValue() + getValue() + getValue() + getValue() + getValue());
```

##### Real world example

As the esteemed reader(all 1 of you) might be accustomed to, most articles of this nature provide examples that are
often a bit removed from what one might do in an actual application. So to continue our quest to show off, let's do some
actual production quality coding here.

My first instinct was to do a Javanese version of the "[What to Do When Auto-Vectorization Fails?](5)" article by Intel,
but instead chose to do something closer to home. Doom.

Doom 3 in particular. A bit of background; Doom 3 like many games was very performance driven, and to achieve this, the
smart folks at idSoft wrote different method implementations for different CPU/GPU architectures. This achieved
performance where possible, and compatibility where needed.

###### Problem statement

```
AS A 'frustrated' programmer
I WANT TO calculate the min/max of a huge list of vertices
IN ORDER TO (magically)remove the 'duplicates' from the vertex list generated by the artist tools, that somehow
  manage to generate duplicate vertices with infinitesimal deviations. ¯\_(ツ)_/¯
```

###### Vanilla MinMax() version

So let's take a look at the C version of how this method was implemented in Doom 3:

```C
/*
============
idSIMD_Generic::MinMax
============
source: https://github.com/id-Software/DOOM-3/blob/master/neo/idlib/math/Simd_Generic.cpp#L523
*/
void VPCALL idSIMD_Generic::MinMax( idVec3 &min, idVec3 &max, const idVec3 *src, const int count ) {
	min[0] = min[1] = min[2] = idMath::INFINITY; max[0] = max[1] = max[2] = -idMath::INFINITY;
#define OPER(X) const idVec3 &v = src[(X)]; if ( v[0] < min[0] ) { min[0] = v[0]; } if ( v[0] > max[0] ) { max[0] = v[0]; } if ( v[1] < min[1] ) { min[1] = v[1]; } if ( v[1] > max[1] ) { max[1] = v[1]; } if ( v[2] < min[2] ) { min[2] = v[2]; } if ( v[2] > max[2] ) { max[2] = v[2]; }
	UNROLL1(OPER)
#undef OPER
}
```

Which (after expanding the macros)translates to something like this in Java:

```java
public class idSIMD_Generic {
    // NB we tweaked the method a little to accept float arrays as input/output instead of idVec3 classes
    // public void MinMax(idVec3 min, idVec3 max, idVec3[] src, int count) {
    public void MinMax(float[] min, float[] max, float[] src) {
        min[0] = min[1] = min[2] = +idMath.INFINITY;//  Float.POSITIVE_INFINITY;
        max[0] = max[1] = max[2] = -idMath.INFINITY;//  Float.NEGATIVE_INFINITY;
        for (int _IX = 0; _IX < src.length; _IX += 3) {
            final float x = src[_IX + 0];
            final float y = src[_IX + 1];
            final float z = src[_IX + 2];
            if (x < min[0]) min[0] = x;
            if (x > max[0]) max[0] = x;
            if (y < min[1]) min[1] = y;
            if (y > max[1]) max[1] = y;
            if (z < min[2]) min[2] = z;
            if (z > max[2]) max[2] = z;
        }
    }
}
```

This version of the method is at the mercy of any compiler + hardware available optimizations, so it should work on all
platforms(including your toaster). Looking closely at the rest of the Doom 3 repo, we find multiple
files `idSIMD_SSE, idSIMD_3DNow, idSIMD_AltiVec...etc`(wikipedia is your friend). Some having a different(probably
faster) implementations.

###### SIMD MinMax() version

So let's take a look at an EV variant of this method:

```C
/*
============
idSIMD_SSE::MinMax
============
source:https://github.com/id-Software/DOOM-3/blob/master/neo/idlib/math/Simd_SSE.cpp#L3696
*/
void VPCALL idSIMD_SSE::MinMax( idVec3 &min, idVec3 &max, const idVec3 *src, const int count ) {
	__asm {

		movss		xmm0, idMath::INFINITY
		.
		.
		.
		.	
	loop4:
		movss		xmm4, [esi+eax+0*12+8]
		movhps		xmm4, [esi+eax+0*12+0]
		minps		xmm0, xmm4
		maxps		xmm1, xmm4
		.
		.
		.

		add			eax, 4*12
		jl			loop4
		.
		.
		.
	}
}
```

Looking at the parts of the code we show here(method is too long to explain), we notice 2~4 things:
* movss/movhps: bulk moving instructions, which we already know we can do in Java.
* minps/maxps: which stands for `Minimum/Maximum Packed Single-Precision Floating-Point`; compares 2 registers, and
  returns the lowest float values of both.
  
At this point you should be crossing your fingers(I have my toes crossed) while you go check the Vector API javadocs.

Indeed, the API contains **exactly** what we are looking for[6].

So how would this we write this in Java:

```java
public class idSIMD_SSE {
    // 256bits == 8 floats
    private static final VectorSpecies<Float> SPECIES = FloatVector.SPECIES_256;

    private static void MinMaxSimd(float[] min, float[] max, float[] src) {
        // divisibility by 6 makes our lives easier 
        assert src.length % 6 == 0;
        
        // init with min/max values
        FloatVector minVec = FloatVector.broadcast(SPECIES, Float.POSITIVE_INFINITY);
        FloatVector maxVec = FloatVector.broadcast(SPECIES, Float.NEGATIVE_INFINITY);
        
        // increment every iteration by 6 floats
        final int stride = 6;
        for (int i = 0; i < src.length - stride; i += stride) {
            // read 8 floats into memory
            final FloatVector vector = FloatVector.fromArray(SPECIES, src, i);
            // basically a Math.min(minVec, vector) between every entry in both
            minVec = vector.min(minVec);
            maxVec = vector.max(maxVec);
        }
        
        // each vector contains 2 values for x, y, z
        // so we choose the min/max of the 2 accordingly
      
        // x
        min[0] = Math.min(minVec.toArray()[0], minVec.toArray()[3]);
        max[0] = Math.max(maxVec.toArray()[0], maxVec.toArray()[3]);

        // y
        min[1] = Math.min(minVec.toArray()[1], minVec.toArray()[4]);
        max[1] = Math.max(maxVec.toArray()[1], maxVec.toArray()[4]);

        // z
        min[2] = Math.min(minVec.toArray()[2], minVec.toArray()[5]);
        max[2] = Math.max(maxVec.toArray()[2], maxVec.toArray()[5]);
    }
}
```

So I tried to make the code self explanatory by adding as many comments as possible(as I always do). 

So let's look at some benchmarks of 20000 vertices(60000 floats):

|vanilla      |simd         |
|---          |---          |
|34000ns      |18500ns      |

Wow, that's almost twice as fast. Not bad, not bad at all. There's probably some minor tweaking (or problems similar to
the unrolling issue what talked about before) to be done here, but I will spare you the need to look at more assembly
code, and myself to pretend that I write my shopping lists in assembly.

####### Elephant in the room

A keen observer might have...observed, that we use a `FloatVector.SPECIES_256`, meaning we can read/write/operate
on `8 floats` simultaneously, and yet we increment our loop by `6` at a time. This ofcourse is due to the fact that the
floats are arranged in a sequence of `xyzxyzxyz...`, meaning reading 8 floats would read `xyzxyzxy` and the following
iteration `zxyzxyzx`, and so on.

Hang on, so what if instead of comparing less floats, we go for more?
Not a bad idea(if I may say so myself), so how high do the `SPECIES` go?
Well, `FloatVector.SPECIES_512` is the max. Which would mean `xyzxyzxy_zxyzxyzx`. Dang, we get the same problem.

Okay, so how many bits do we need for a full sequence read? Considering our sequence `xyz` is 3 floats long, and our
lane `FloatVector.SPECIES_256` is 8 floats wide, we probably need `3 x 8 = 24 floats`. `xyzxyzxy_zxyzxyzx_yzxyzxyz`.
Problem is, that results in `768 bits`, something our API(and hardware) doesn't support.

So instead, let's tackle this from a programmer point of view, and worry about the rest later. We need `768 bits`, but
we can only read `256 bits` at a time. Simple, we iterate! Or, in this case, we unroll our iteration.

Meaning our code becomes something like this:

```java
public class idSIMD_SSE {
    // 256bits == 8 floats
    private static final VectorSpecies<Float> SPECIES = FloatVector.SPECIES_256;

    private static void MinMaxSimd768BitEdition(float[] min, float[] max, float[] src) {
      // divisibility by 24 makes our lives easier 
      assert src.length % 24 == 0;
        
      // init with min values
      min[0] = min[1] = min[2] = Float.POSITIVE_INFINITY;
      FloatVector firstMinVec   = FloatVector.broadcast(SPECIES, Float.POSITIVE_INFINITY);
      FloatVector secondMinVec  = FloatVector.broadcast(SPECIES, Float.POSITIVE_INFINITY);
      FloatVector thirdMinVec   = FloatVector.broadcast(SPECIES, Float.POSITIVE_INFINITY);
      
      // init with max values
      max[0] = max[1] = max[2] = Float.NEGATIVE_INFINITY;
      FloatVector firstMaxVec   = FloatVector.broadcast(SPECIES, Float.NEGATIVE_INFINITY);
      FloatVector secondMaxVec  = FloatVector.broadcast(SPECIES, Float.NEGATIVE_INFINITY);
      FloatVector thirdMaxVec   = FloatVector.broadcast(SPECIES, Float.NEGATIVE_INFINITY);  
        
      // increment by 24 floats * 32 bits = 768 bits
      final int stride = 24;
      for (int i = 0; i < src.length - stride; i += stride) {
          // xyzxyzxy
          final FloatVector firstVec = FloatVector.fromArray(SPECIES, src, i + 0);
          firstMinVec = firstVec.min(firstMinVec);
          firstMaxVec = firstVec.max(firstMaxVec);

          // zxyzxyzx
          final FloatVector secondVec = FloatVector.fromArray(SPECIES, src, i + 8);
          secondMinVec = secondVec.min(secondMinVec);
          secondMaxVec = secondVec.max(secondMaxVec);

          // yzxyzxyz
          final FloatVector thirdVec = FloatVector.fromArray(SPECIES, src, i + 16);
          thirdMinVec = thirdVec.min(thirdMinVec);
          thirdMaxVec = thirdVec.max(thirdMaxVec);
        }
        
        // we now have 8 values in total for each of x, y, z
        // so min/max-ing them takes a little more effort 
      
        final float[] allMins = new float[24];
        firstMinVec.intoArray(allMins, 0);
        secondMinVec.intoArray(allMins, 8);
        thirdMinVec.intoArray(allMins, 16);

        final float[] allMaxs = new float[24];
        firstMaxVec.intoArray(allMaxs, 0);
        secondMaxVec.intoArray(allMaxs, 8);
        thirdMaxVec.intoArray(allMaxs, 16);

        for (int i = 0; i < 8; i++) {
            // x
            min[0] = Math.min(min[0], allMins[(i * 3) + 0]);
            max[0] = Math.max(max[0], allMaxs[(i * 3) + 0]);

            // y
            min[1] = Math.min(min[1], allMins[(i * 3) + 1]);
            max[1] = Math.max(max[1], allMaxs[(i * 3) + 1]);

            // z
            min[2] = Math.min(min[2], allMins[(i * 3) + 2]);
            max[2] = Math.max(max[2], allMaxs[(i * 3) + 2]);
        }
    }
}
```

So, let's do a little benchmarking, and see how we did:

|vanilla      |simd         |simd-768bit  |
|---          |---          |---          |
|34000ns      |18500ns      |6100ns       |

Holy guacamole, that's almost 3 times as fast as the simd version and almost **6 times** faster than the vanilla one!!!!
1One

How is this possible you ask? This is only conjecture, but most CPU's have 8 SSE(Streaming SIMD Extension) registers,
meaning our 3 loads/mins/maxs are most likely done concurrently to different registers, instead of sequentially to the
same one. Could we go up to 6 registers then...you might ask?

We could, and we did, but we didn't gain any performance. Meaning that this would be an interesting subject-matter for a
follow-up rant :) 

### Epilogue

The Vector API is a very exciting addition to the Java realm. At first glance it looks very complicated and difficult to
use, but it is easy to get used to, and pleasantly documented.

A lot of libraries that might depend on JNI or external more native calls for performance will benefit drastically from
the readability and maintainability that writing code with the Vector API will provide.

The bigger problem lies with this new hammer to which many a programmer will seek to convert their problem into a
befitting nail. My advice to you dear reader is, don't. You will know when the time is right.

My final piece of advice(before I go eat), is to always measure, and to understand what your compiler/CPU wants to do.
You take care of them, and they will take care of you.


## References

1. https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf
2. https://stackoverflow.com/a/36510865
3. https://cr.openjdk.java.net/~vlivanov/talks/2017_Vectorization_in_HotSpot_JVM.pdf
4. https://en.wikipedia.org/wiki/Advanced_Vector_Extensions#CPUs_with_AVX2
5. https://software.intel.com/content/www/us/en/develop/articles/what-to-do-when-auto-vectorization-fails.html
6. https://docs.oracle.com/en/java/javase/16/docs/api/jdk.incubator.vector/jdk/incubator/vector/Vector.html#min(jdk.incubator.vector.Vector)


- https://arstechnica.com/features/2000/03/simd/
