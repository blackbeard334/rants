# Table of contents

* [Prologue](#prologue)
* [Problem (C++)](#problem-c)
  + [Conclusion](#conclusion)
* [Problem 2.0 (Java)](#problem-20-java)
  + [WTF!? moment](#wtf-moment)
    + [What about C++?](#what-about-c)
    - [Benchmarking time](#benchmarking-time)
    - [Assemble](#assemble)
      * [C1](#c1)
* [Problem 2.1 (Java)](#problem-21-java)
  + [Disable inlining](#disable-inlining)
  + [Increasing the compile-threshold](#increasing-the-compile-threshold)
* [Problem 2.2 (Java)](#problem-22-java)
  + [C2](#c2)
    + [Bonus side-rant (C++ version)](#bonus-side-rant-c-version)
    - [Is this normal though?](#is-this-normal-though)
* [Epilogue](#epilogue)
* [links](#links)

## Prologue

For as long as I could remember (~2 weeks ago), every time I would write a `for loop`, I would wonder
whether `for(int i=0;i<len;i++)` is slower than `for(int i=0;i!=len+1;i++)`.

Granted, this is a _"possible"_ super-micro-optimization that (in the grand scheme of things) probably doesn't matter,
but a lot of great things start out with curiosity (dead cats notwithstanding).

## Problem (C++)

Okay, let's look at some code:

1. `for(i = 0; i < len; i++){}`
2. `for(i = 0; i != len + 1; i++){}`


1. The first one loosely compiles to something like this (both compiled with `-O2`):

```nasm
  mov edx, 0    ;; i = 0
.L3:
  ;; <-- body of loop goes here, not important for this rant
  add edx, 1    ;; increment i by 1
  cmp esi, edx  ;; compare i to len
  jl .L3        ;; jump back to L3 if len is larger than i  
```

2. The second one:

```nasm
  mov edx, 0    ;; i = 0
  add esi, 1    ;; len += 1
.L3:
  ;; <-- body of loop goes here, not important for this rant
  add edx, 1    ;; increment i by 1
  cmp esi, edx  ;; compare i to len
  jne .L3       ;; jump back to L3 if len != i  
```

Okay, so we get different jump instructions, and obviously one of them **has** to be much more expensive than the other,
otherwise, why are we even here!?

### Conclusion

Esteemed reader, we apologize for the disappointment. But according to
the [x86 documentation](http://www.intel80386.com/386htm/Jcc.htm) these instructions are identical T_T.

## Problem 2.0 (Java)

But fear not, we will not be discouraged!

Let's take a look at the Java™©☕.

Same parameters:

1. `for(i = 0; i < len; i++){}`
2. `for(i = 0; i != len + 1; i++){}`

Java of course doesn't directly compile to assembly, so let's look at some bytecode instead:

1.

```nasm
   L1
    ILOAD 1         ;; load i
    ALOAD 0         ;; load len
    IF_ICMPGE L2    ;; if i >= len, go to L2
    IINC 1 1        ;; increment i by 1
    GOTO L1         ;; jump back to beginning of loop
   L2
    RETURN
```

2.

```nasm
   L1
    ILOAD 1         ;; load i
    ALOAD 0         ;; load len
    ICONST_1        ;; load 1 onto stack
    IADD            ;; add 1 to len==WTF!
    IF_ICMPEQ L2    ;; if i==WTF, go to L2
    IINC 1 1        ;; increment i by 1
    GOTO L1         ;; jump back to beginning of loop
   L2
    RETURN
```

### WTF!? moment

Before even benchmarking them, there's a big red flag in the second bytecode snippet.

`1` is added to `len` **in every iteration of the loop**, before it is compared against `i`. Interesting...

This has some overhead of course, but knowing what we know about the JVM, the overhead could be negligible, or it is
simply optimized in a later stage.

Remember, this is simply the bytecode translation. Most of the magic happens later on in `C1` & `C2` (or GraalVM if you
are so inclined).

We could probably end this rant right here and now at this point, but we are far too ~~stupid~~ curious for that.

###### What about C++?

Seems strange that Java would make such a weird decision to not optimize this simple 'constant-value'.

Can we reproduce this in C++ though? Let us recompile with `-O1` & `-O0`.

And indeed, `-O0` produces the same unoptimized version:

`for(i = 0; i != len + 1; i++){}`

```nasm
.L7:
        mov     eax, DWORD PTR [rbp-4]    ;; copy i to eax
        add     DWORD PTR [rbp-8], eax    ;; total+=i
        add     DWORD PTR [rbp-4], 1      ;; i++
.L6:                                         
        mov     eax, DWORD PTR [rbp-20]   ;; load len into eax
        add     eax, 1                    ;; add 1 to eax(len)==WTF!?
        cmp     DWORD PTR [rbp-4], eax    ;; compare len to i
        jne     .L7                       ;; jump back to L7 if i!=len+1
```

#### Benchmarking time

There's this Dutch saying; `meten is weten`, which colloquially translates to something like `she who measures, knows`
🧙‍♂️. And nothing is truer of code. You can make all the deductions and write the nicest algorithms you like, your
program is either fast, or not.

Enough babble, let us set up a simple JMH benchmark (sorry, not gonna go over it line by line):

```java
public class Benchmark {
    public static void main(String[] args) {
        final Options options = new OptionsBuilder()
                .include(Benchmark.class.getSimpleName())
                .build();

        new Runner(options).run();
    }

    @Param({"1000", "10000", "100000", "1000000", "10000000"})
    public int x;

    @Benchmark
    public void test1(final Blackhole blackholeSun) {
        int total = 0;
        for (int i = 0; i < x; i++) {
            total += i;
        }

        blackholeSun.consume(total);
    }

    @Benchmark
    public void test2(final Blackhole blackholeSun) {
        int total = 0;
        for (int i = 0; i != x + 1; i++) {
            total += i;
        }

        blackholeSun.consume(total);
    }
}
```

And, what's the verdict?

`(not sure if JMH can generate output like this, but in this case it's some scrappy copy paste work)`

|      (x) |         test1 |         test2 |
|---------:|--------------:|--------------:|
|     1000 |       248.127 |       248.770 |
|    10000 |      2628.718 |      2630.839 |
|   100000 |     25027.878 |     26170.237 |
|  1000000 |    260002.502 |    256436.492 |
| 10000000 |   2469421.318 |   2515042.555 |

Okay, so it looks like `test1` is just **slightly** faster. This could mean a couple of things:

- this could simply be within the margin of error (remember, benchmarks are not perfect)
- if not, then:
    - `test1` and `test2` probably have different machine code
    - or `test1` and `test2` could have different `C1` or `C2` code (or both)

So let's see if we can make widen the gap between the test results. We won't bore our loyal readership with all
our ~~failed~~ experiments, but this small change does the trick:

```java
@Param({"1000", "10000", "100000", "1000000", "10000000"})
public long x; // <-- we change this to a long
```

|      (x) |        test1L |        test2L |
|---------:|--------------:|--------------:|
|     1000 |     22940.000 |     23340.000 |
|    10000 |    189420.000 |    192720.000 |
|   100000 |    178100.000 |    183920.000 |
|  1000000 |   1205300.000 |   1496420.000 |
| 10000000 |   5990540.000 |   7850100.000 |           

Boom! That's a whopping **~30% difference**! So no longer in the realm of _margin of error_.

There could be more going on here of course, but let's just proceed with the slow version, and see what we get.

#### Assemble

Before we proceed with the slower `test1L` & `test2L` version, we want you to know that similar to the bytecode we show
above, the bytecode version of `test2L` also seems to do the `x + 1` part in every iteration of the loop.

##### C1

Okay, so what does the `C1` output look like?

Run with `-XX:-UseCompressedOops -XX:PrintAssemblyOptions=intel -XX:CompileCommand=print,org/example/Benchmark.*`.

`test1L` (abridged version):

```nasm
  ;this block initializes i=0, total=0, and saves r8 (to restore it later?)
  0x000002adc77623b4:   mov    rsi,r8
  0x000002adc77623b7:   mov    edi,0x0
  0x000002adc77623bc:   mov    ebx,0x0
  0x000002adc77623c1:   jmp    0x000002adc7762415           ; note that we directly jump to the i<x part of the loop 
  
  ;this block does the total+=i & the i++ parts
  0x000002adc77623c6:   xchg   ax,ax                        ;NOP
  0x000002adc77623c8:   mov    r8,rdi                       ;copy i to r8
  0x000002adc77623cb:   add    r8d,ebx                      ;bla = i+total
  0x000002adc77623ce:   inc    edi                          ;i++
  
  ;this block checks if i<x                                                          
  0x000002adc7762415:   mov    r8,QWORD PTR [rdx+0x10]      ;copy x to the r8 register
  0x000002adc7762419:   movsxd rax,edi                      ;copy i to rax 
  0x000002adc776241c:   cmp    rax,r8                       ;compare i to x
  0x000002adc776244f:   jl     0x000002adc77623c8           ;go back to start of loop while x>i
```

`test2L` (abridged version):

```nasm
  ;same init block as test1L
  0x0000024e00894834:   mov    rsi,r8
  0x0000024e00894837:   mov    edi,0x0
  0x0000024e0089483c:   mov    ebx,0x0
  0x0000024e00894841:   jmp    0x0000024e00894895           
  
  ;same increment block as test1L
  0x0000024e00894846:   xchg   ax,ax
  0x0000024e00894848:   mov    r8,rdi
  0x0000024e0089484b:   add    r8d,ebx
  0x0000024e0089484e:   inc    edi

  ;ALMOST same condition block
  0x0000024e00894895:   mov    r8,QWORD PTR [rdx+0x10]      ;copy x to r8                                                       
  0x0000024e00894899:   movsxd rax,edi                      ;copy i to rax
  0x0000024e0089489c:   movabs r10,0x1                      ;put 1 in r10
  0x0000024e008948a6:   add    r8,r10                       ;add r10 to r8==>add 1 to x!!!!!!?
  0x0000024e008948a9:   cmp    rax,r8                       ;compare i to x
  0x0000024e008948dc:   jne    0x0000024e00894848           ;loop if not equal  
```

So yeah, similar to the bytecode version, `test2L` does `x + 1` in **every iteration** of the loop.

Can we make this even worse?

## Problem 2.1 (Java)

So what happens if we extract `x + 1` to a function?

Strangely enough, this version is actually faster, not slower. 😅

|      (x) |        test1L |        test2L |   test2L_func |
|---------:|--------------:|--------------:|--------------:|
|     1000 |     22940.000 |     23340.000 |     37940.000 |
|    10000 |    189420.000 |    192720.000 |    263360.000 |
|   100000 |    178100.000 |    183920.000 |    246280.000 |
|  1000000 |   1205300.000 |   1496420.000 |   1525420.000 |
| 10000000 |   5990540.000 |   7850100.000 |   5895140.000 |    

As you can see it's slower in all instances **except** when `x=10000000`. Could this be an inlining issue?

Let's disable inlining, and try again.

### Disable inlining

Run with `-XX:CompileCommand=dontinline,org/example/Benchmark.*`.

|      (x) |        test1L |        test2L |   test2L_func | test2L_func <br/>disable <br/>inlining |
|---------:|--------------:|--------------:|--------------:|---------------------------------------:|
|     1000 |     22940.000 |     23340.000 |     37940.000 |                              43100.000 |
|    10000 |    189420.000 |    192720.000 |    263360.000 |                             291880.000 |
|   100000 |    178100.000 |    183920.000 |    246280.000 |                             276600.000 |
|  1000000 |   1205300.000 |   1496420.000 |   1525420.000 |                            1366640.000 |
| 10000000 |   5990540.000 |   7850100.000 |   5895140.000 |                            5557340.000 |    

Okay, so it wasn't the inlining.

Let's try upping the compile-threshold.

### Increasing the compile-threshold

|      (x) |        test1L |        test2L |   test2L_func | test2L_func <br/>disable <br/>inlining | test2L_func <br/>compile <br/>threshold |
|---------:|--------------:|--------------:|--------------:|---------------------------------------:|----------------------------------------:|
|     1000 |     22940.000 |     23340.000 |     37940.000 |                              43100.000 |                               48120.000 |
|    10000 |    189420.000 |    192720.000 |    263360.000 |                             291880.000 |                              298160.000 |
|   100000 |    178100.000 |    183920.000 |    246280.000 |                             276600.000 |                              272060.000 |
|  1000000 |   1205300.000 |   1496420.000 |   1525420.000 |                            1366640.000 |                             1769000.000 |
| 10000000 |   5990540.000 |   7850100.000 |   5895140.000 |                            5557340.000 |                             7762280.000 |     

Okay, so that's what it was (should have been obvious).

But can we make it even...worse?

![https://media2.giphy.com/media/3ohc14lCEdXHSpnnSU/giphy.gif?cid=790b761110629d39b41b78cf69987ee449b489dfd3201165&rid=giphy.gif&ct=g](https://media2.giphy.com/media/3ohc14lCEdXHSpnnSU/giphy.gif?cid=790b761110629d39b41b78cf69987ee449b489dfd3201165&rid=giphy.gif&ct=g)

## Problem 2.2 (Java)

So far our `i < x + 1`(test2L) and `i < func()`(test2L_func) have been our slowest contestants. Both kind of work the
same way, as in that they add some kind of `action` that is executed **every** iteration.

So how could we make that even worse?

A bigger `action` I guess?

Let's change our function to something like:

```java
private double rand() {
    return x + new Random().nextInt(1); // *evil grin*
}
```

And the results are:

|      (x) |        test1L |        test2L |   test2L_func | test2L_func <br/>disable <br/>inlining | test2L_func <br/>compile <br/>threshold |   test2L_rand |
|---------:|--------------:|--------------:|--------------:|---------------------------------------:|----------------------------------------:|--------------:|
|     1000 |     22940.000 |     23340.000 |     37940.000 |                              43100.000 |                               48120.000 |    123580.000 |
|    10000 |    189420.000 |    192720.000 |    263360.000 |                             291880.000 |                              298160.000 |    606320.000 |
|   100000 |    178100.000 |    183920.000 |    246280.000 |                             276600.000 |                              272060.000 |   4829220.000 |
|  1000000 |   1205300.000 |   1496420.000 |   1525420.000 |                            1366640.000 |                             1769000.000 |  39910100.000 |
| 10000000 |   5990540.000 |   7850100.000 |   5895140.000 |                            5557340.000 |                             7762280.000 | 398303480.000 |

![https://content.invisioncic.com/r322239/monthly_02_2018/post-21941-0-98640500-1518217763.jpg](https://content.invisioncic.com/r322239/monthly_02_2018/post-21941-0-98640500-1518217763.jpg)

That's a whopping **50 times slower**!

While we're fairly certain we could go even slower, we will present that task as homework for our esteemed reader(s),
and proceed to dissect our latest creation.

### C2

We will spare you the `C1` details, but it does a call to `rand()` for every loop iteration, as expected.

But what about `C2`?

`test2L_rand` (with disabled inlining to exclude the `java.util.Random` stuff from the disassembly):

```nasm
  ;init block
  0x000001ce29eaead2:   mov    DWORD PTR [rsp+0x30],ebx
  0x000001ce29eaead6:   mov    DWORD PTR [rsp+0x34],r12d
  0x000001ce29eaeadb:   mov    QWORD PTR [rsp+0x38],rdi
  0x000001ce29eaeae0:   jmp    0x000001ce29eaeb19
  
  0x000001ce29eaeae2:   nop    DWORD PTR [rax+0x0]
  0x000001ce29eaeae9:   nop    DWORD PTR [rax+0x0]          
                              
  ;loop block                              
  0x000001ce29eaeaf0:   mov    r10,QWORD PTR [r15+0x120]
  0x000001ce29eaeaf7:   mov    r8d,DWORD PTR [rsp+0x34]     ; r12d=total
  0x000001ce29eaeafc:   add    r8d,DWORD PTR [rsp+0x30]     ; total+=i                                                              
  0x000001ce29eaeb01:   mov    r11d,DWORD PTR [rsp+0x30]    ; rd11=i
  0x000001ce29eaeb06:   inc    r11d                         ; i++
  0x000001ce29eaeb09:   test   DWORD PTR [r10],eax          
  0x000001ce29eaeb0c:   mov    DWORD PTR [rsp+0x30],r11d    ; put the incremented i value (edi) back into i 
  0x000001ce29eaeb11:   mov    DWORD PTR [rsp+0x34],r8d     ; put the total+=i value back into total
  0x000001ce29eaeb16:   mov    r10,rbp                      
  
  ;condition block
  0x000001ce29eaeb19:   vcvtsi2sd xmm0,xmm0,QWORD PTR [r10+0x10]
  0x000001ce29eaeb1f:   vmovsd QWORD PTR [rsp+0x20],xmm0
  0x000001ce29eaeb25:   mov    rbp,r10
  0x000001ce29eaeb28:   mov    r10d,DWORD PTR [rsp+0x30]
  0x000001ce29eaeb2d:   vmovd  xmm0,r10d
  0x000001ce29eaeb32:   vcvtdq2pd xmm0,xmm0                 
  0x000001ce29eaeb36:   vmovsd QWORD PTR [rsp+0x28],xmm0
  0x000001ce29eaeb3c:   mov    rdx,rbp
  0x000001ce29eaeb3f:   call   0x000001ce29eab2c0           ;*invokevirtual rand {reexecute=0 rethrow=0 return_oop=0}
                                                            ;   {optimized virtual_call}
  0x000001ce29eaeb44:   vaddsd xmm0,xmm0,QWORD PTR [rsp+0x20]
  0x000001ce29eaeb4a:   vmovsd xmm1,QWORD PTR [rsp+0x28]
  0x000001ce29eaeb50:   vucomisd xmm1,xmm0                  ; (fancy) compare i!=rand()
  0x000001ce29eaeb54:   jp     0x000001ce29eaeaf0           ; loop if parity flag set 
  0x000001ce29eaeb56:   jne    0x000001ce29eaeaf0           ; loop if not equal
```

As we can see, `rand()` is called every iteration of the loop 🤯. Aside from the overhead, this does seem a little
_absurd_. Think about it, all of a sudden this loop has become **non-deterministic**!?

###### Bonus side-rant (C++ version)

Compiled with `-O4`, and yeah, `rand()` is also called every iteration `¯\_(ツ)_/¯`.

```nasm
;; https://godbolt.org/z/odf6rWfdT
test2L_rand(int):
        push    r12
        mov     r12d, edi
        push    rbp
        xor     ebp, ebp
        push    rbx
        xor     ebx, ebx
        jmp     .L2
.L3:
        add     ebp, ebx
        add     ebx, 1
.L2:
        call    rand
        add     eax, r12d
        cmp     eax, ebx
        jg      .L3
        mov     eax, ebp
        pop     rbx
        pop     rbp
        pop     r12
        ret
```

#### Is this normal though?

To be honest dear reader, the author has spent so much time staring at assembly, that little to no time was left for him
to look up good documentation on this.

Suffice it to say, both the [C++ standard](https://en.cppreference.com/w/cpp/language/for) and
the [Java spec](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/for.html) mention in a very subtle manner that
the `termination/condition` block is evaluated **before each iteration**. So, normal I guess.

## Epilogue

We started this experiment with the assumption that different `jmp` instructions might be more expensive, but instead
discovered that it's easier to thwart the compiler's plans for optimizing our ~~crappy~~ beautiful code than we thought.

_The devil is in the details_ as they say. None of what I came across is secret knowledge, nor is it common knowledge
for some reason.

Part of that reason is probably, that in most simple cases, the compiler is smart enough to recognize a constant
expression (like `i < x + 1` or `i < x + bla()`) and inline it without having to recalculate it every iteration. This is
sometimes done in `C1`, but more likely in `C2`, which explains (some of) the performance gains there.

`Do not assume the numbers tell you what you want them to tell.` JMH is an exquisite tool, but this sentence here from
the JMH output is worthy of a Nobel Peace Prize. The numbers don't lie, but we certainly do.

![https://cdn.memegenerator.es/imagenes/memes/full/32/7/32078366.jpg](https://cdn.memegenerator.es/imagenes/memes/full/32/7/32078366.jpg)

I started out by assuming that the difference in numbers was due to different `cmp` or `jmp` instructions that were
costly. And sadly enough, I was hubristic enough to write the first draft of this _rant_ around this...**assumption** (
yes I actually looked at `C1/C2` code of the Java version before looking in the `x86` spec T_T).

## links

- https://stackoverflow.com/questions/18596300/performance-greater-smaller-than-vs-not-equal-to
- http://www.intel80386.com/386htm/SETcc.htm
- http://www.intel80386.com/386htm/Jcc.htm
- https://stackoverflow.com/questions/9617877/assembly-jg-jnle-jl-jnge-after-cmp
- https://marc.info/?l=linux-kernel&m=90222165330252
- https://stackoverflow.com/questions/12135518/is-faster-than
- https://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html
- https://stackoverflow.com/questions/39556649/in-x86-whats-difference-between-test-eax-eax-and-cmp-eax-0
- https://stackoverflow.com/questions/33721204/test-whether-a-register-is-zero-with-cmp-reg-0-vs-or-reg-reg/33724806#33724806
- https://en.wikipedia.org/wiki/List_of_Java_bytecode_instructions
- https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8256508
