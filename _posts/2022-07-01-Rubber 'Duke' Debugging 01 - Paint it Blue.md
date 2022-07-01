# Table of contents

- [Prologue](#prologue)
- [Problem](#problem)
- [YUV](#yuv)
- [The (pseudo)code](#the-pseudocode)
- [Debugging](#debugging)
- [Cause](#cause)
- [Lesson of the day](#lesson-of-the-day)
    * [Lesson within a lesson](#lesson-within-a-lesson)
    * [Will anybody fix this?](#will-anybody-fix-this)
- [Conclusion](#conclusion)


## Prologue

Back in the 90s, Hercules (<a href="[http://youtu.be/WS6-vI70oc0](https://youtu.be/kO_EdmtR5Ck)">half man &amp; half machine</a>) was a huge phenomenon, kind of like zombies are nowadays. You had the tv series, movies, cartoons, games...and probably even sandwiches.

[![https://upload.wikimedia.org/wikipedia/en/6/65/Hercules_%281997_film%29_poster.jpg](https://upload.wikimedia.org/wikipedia/en/6/65/Hercules_%281997_film%29_poster.jpg)]

After Disney released their cartoon film, a <a href="http://en.wikipedia.org/wiki/Disney%27s_Hercules_(video_game)">cool game</a> based on the film was released. A game, that I played extensively for some weeks.

A couple of months later, a friend of mine also bought the game. At one point he was trying to do something very trivial, like moving/breaking boulders I think, but good ol' Herc kept failing. I kept telling him just press `X` (I was kind of the go to guy for games and computers), and he kept screaming that it wasn't working, and he concluded that it didn't work because I had an `NTSC-J` console, while he had the `PAL`.

About a week later I visited him, and thought what the hell, let's show him how it's done. So he fired up the PSX, and it all becomes clear.

Turns out...we were talking about <a href="http://en.wikipedia.org/wiki/Herc%27s_Adventures">different games</a>!

Both Hercules games, both released around the same time (both pretty cool), but totally different games. We simply _assumed_ it was the same game.

## Problem

So the piece of code that caused me 3 weeks of sleepless nights, was
the [Doom 3 video player in my Java port](https://github.com/blackbeard334/djoom3/blob/develop/src/main/java/neo/Renderer/Cinematic.java).
The video player was one of the first pieces of code I got working in the game (besides the error console), and I was so
happy it was working, that I didn't even notice that the videos were tinted <strong><span style="color:#0000ff;">
blue</span></strong> (see below).

When I did notice, I just assumed I had forgotten to remove some of my test code, so I just noted it as a very low
priority bug. After all, my code was finally working, and a blue tint was a mere annoyance, deemed _unworthy_ of
being a "real" bug.

After a very long week, I thought to myself, "let's do some light bugs this weekend to relax" (yes, I actually said that to myself).
Then came the low priority tinting bug. And all hell broke loose. A weekend turned into 3, accompanied by the nights in between.

What it looked like:

[![blue](/rants/images/rdd01_blue.png)]("blue")

What it should have looked like:

[![normal](/rants/images/rdd01_normal.png)]("normal")

## YUV

The story behind YUV is up on wikipedia (if you can understand it), the TL;DR; version is, YUV was invented to add color
to Black & White TV signals. So very analogue, and very different from traditional RGB.

Regardless, most hardware needs the data in the RGB language, which means we have to translate YUV into RGB. Which is
exactly what an MPEG player (like this one) does.

I know what you're thinking, because this was exactly what I thought when I started, which was that the conversion formula/method/whatever I was using had a bug(s) in it.
But after some major debugging, rewriting, and screaming, I came to the conclusion that the `yuvToRgb` method was sound as sound stuff could be.

That didn't stop me from rechecking it a couple of hundred times, but as you can see
from <a href="http://www.pcmag.com/encyclopedia/term/55166/yuv-rgb-conversion-formulas">here, the conversion is pretty straightforward basic math.</a>

There must be an overflow in there somewhere I kept thinking.
A byte mask gone awry. Anything!?
But no matter how I did it, I kept getting the expected results.

Now this really threw a wrench in the works. This was basically the only place the code did anything color manipulation related.

## The (pseudo)code
Logic dictates that it had to be in the YUV function.
I mean, we were getting video, which meant that the other steps were performing admirably.

A basic outline of what the code does is the following:
1. read video file into `ByteBuffer`.
2. increment through `ByteBuffer` per frame.
3. copy frame and convert from YUV to RGB.
4. duplicate and manipulate.
5. copy RGB frame to RGB `ByteBuffer`.
6. send RGB `ByteBuffer` to screen.
7. rinse and repeat.

## Debugging

So how what did I try?

* Went over the YUV to RGB algorithm with a fine-comb
* Went over the data with a fine-comb (looking for corruption or forgetting to clear some old values or what not)
* Calculated YUV to RGB values by hand, and compared to method output
* Calculated YUV to RGB values with different open source programs (and the original Doom 3), and compare to method output
    * Thinking that my data set was too narrow, I compared **millions** of conversions
* Sacrificed a rubber chicken (with a pulley in the middle) 
* Dumped RGB frame-buffers to disk (step 6 from the pseudocode)
    * That way they would be easier to analyze, and perhaps some patterns would emerge. Like a fixed offset in the
      bytes; always +1 for example.
    * I then did the same with the C++ version of the game, and compared my blue frames vs the correct RGB ones. And it
      took me all but 1 second to see what the fuck was going on.

**My blue frame had the opposite endianness of the RGB counterpart!!!!1One**

[![https://gulliver.readthedocs.io/en/latest/_images/endianpig.png](https://gulliver.readthedocs.io/en/latest/_images/endianpig.png)]

Symptom finally found!

Now the cause!?

## Cause

You would think that finding the cause is trivial once you found the symptom (will disprove this notion in a future
installment), which would be a correct assumption for such a simple (and local) piece of code.

But still, it took longer than a minute because of disbelief mostly.

To make a long story a bit shorter,
the [ByteBuffer.duplicate()](http://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html#duplicate()) method we
were calling to clone our buffer, was reverting the endianness of the clone back to the default mode (step 4 from pseudocode).

## Lesson of the day

Just like my friend and I assumed we were talking about the same game, a lot of unnecessary energy (and brain cells) was
wasted here because I didn't bother to confirm the minute details, which proved to be the achilles heel of the whole
situation.

Even though I was debugging the code very thoroughly, I just assumed certain truths about the underlying API.

**I just assumed the `duplicate` would be the same as the `original`, so I skipped checking the `duplicate` altogether ¯\\_(ツ)_/¯**

I mean sure, when I say
```java
BlaObject a = new BlaObject();
BlaObject b = a.clone();
// BREAKPOINT <---
```
then checking the equality of `b` and `a` at this point would seem stupid, right? ...but is it really?
It is said, that this is one of the main reasons operator overloading was excluded from Java. People overriding the `+`
to do multiplication for example (yes yes, an extreme example).

Granted, that since Java does **not** have operator overloading, one should not assume this about methods. But when an
object's method is called "<a href="http://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html#duplicate()">
duplicate</a>", and the javaDoc begins with "_Creates a new byte buffer that shares this buffer's content._",
then it is safe to assume that the properties of said object shall be duplicated.

I mean, what good is
```java
Car car = new Car();
car.wheels = 4;

Car newCar = car.duplicate();
if(newCar.wheels != 4){
    System.out.printf("WTF!!\n");
}
```

Sure, the javaDoc goes on to explain some of the peculiarities of this duplicate method. But not a single mention of
ByteOrder being excluded from a duplicate. And why should it?

One could argue that since a byte is endian-agnostic by nature, then copying a `ByteBuffer` should just ignore it. But
then I ask myself, why does a `ByteBuffer` have any ByteOrder'ing to begin with? Why does it have functions such as
asIntBuffer(), to which ByteOrder clearly makes a difference?

A quick look at the jdk source code, shows that `duplicate()` is a copy constructor in disguise, or rather calls a copy
constructor. And the constructor that is called copies everything <strong><span style="text-decoration:underline;">
EXCEPT</span></strong> the ByteOrder.

[![https://nathanbweller.com/wp-content/uploads/2015/01/arrested-development-always-leave-note-stakes-and-value.png](https://nathanbweller.com/wp-content/uploads/2015/01/arrested-development-always-leave-note-stakes-and-value.png)]

### Lesson within a lesson

The funny thing about this bug/feature, is that it is mentioned in the JavaDoc of a **different** method.
The <a href="http://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html#order()">ByteBuffer.order()</a> method JavaDoc,
mentions that all newly-created ByteBuffers are Native Endian.

### Will anybody fix this? 

According to the Java architects, this is a bug. But if you look at
[some of the bug reports](https://bugs.openjdk.java.net/browse/JDK-4715166), and how ardently we follow the temple of
"_backwards compatibility_" in JavaLand, I would say that the chances are very slim.

## Conclusion

* `ByteBuffer.duplicate()`(and some other methods) <span style="text-decoration:underline;">**ALWAYS**</span> returns ~~Big~~ **Native** Endianness.
* Sometimes the information you want is **NOT** in the JavaDoc of the method you're using.
* API designers are not infallible, and sometimes their intentions and yours don't necessarily align.
* Double-checking your results means you should start over, instead of comparing certain drawn conclusions to each other.
* It wasn't the YUV conversion function. Logic made a mistake.
* Chapter 3 from the seminal <a href="http://twimgs.com/ddj/abrashblackbook/gpbb3.pdf">Graphics Programming Black Book</a> says it all.
