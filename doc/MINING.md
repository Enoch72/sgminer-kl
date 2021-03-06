# Mining scrypt

## Introduction

Mining scrypt-based cryptocurrencies using GPUs is completely different
to mining SHA256d (used in Bitcoin). The former was intentionally
developed in a manner that (it was hoped) would make it suitable
for mining on CPUs, but not GPUs. Thanks to some innovative work by
_Artforz_ and _mtrlt_, this was proven to be wrong.

However, it has very different requirements compared to SHA256d and
is a lot more complicated to get working well. It is a RAM-dependent
workload, and requires you to have enough system RAM as well as fast
enough GPU RAM. What is "enough" depends on setup specifics.


## Catalyst drivers and OpenCL SDK

The choice of driver version for your GPU is critical, as some are known
to break scrypt mining entirely while others give poor hashrates. It is
recommended that you first try with the latest stable version available.

Latest driver distribution versions may aready include the AMD APP
SDK, therefore presenting an OpenCL vendor conflict when building or
running. Systems with NVidia cards and NVidia drivers may have a similar
conflict. If this is the case, check which OpenCL vendor is used, and
consider removing unneeded ones.


## Runtime environment

Environment variables must be set to allow access from console /
terminal / screen.

On Linux:

```
export DISPLAY=:0
export GPU_MAX_ALLOC_PERCENT=100
export GPU_USE_SYNC_OBJECTS=1
```

On Windows:

```
setx GPU_MAX_ALLOC_PERCENT 100
setx GPU_USE_SYNC_OBJECTS 1
```


## Tuning

When mining is started, sgminer may fail in various ways. This is often
not a bug in the software, but rather misconfiguration. The failures may
occur due to parameters being outside what the GPU can cope with (both
too high and too low).

All parameters are optional for fine tuning.

**WARNING**: documentation below has not been reviewed to be up-to-date.


### --intensity XX (-I XX)

The scale goes from 0 to 31. The reason this is crucial is that too
high an intensity can actually be disastrous with scrypt because it CAN
run out of ram. High intensities start writing over the same ram and it
is highly dependent on the GPU, but they can start actually DECREASING
your hashrate, or even worse, start producing garbage with HW errors
skyrocketing, or locking up the system altogether. Note that if you do
NOT specify an intensity, sgminer uses dynamic mode which is designed
to minimise the harm to a running desktop and performance WILL be poor.
The lower limit to intensity with scrypt is usually 8 and sgminer will
prevent it going too low.

SUMMARY: Setting this for reasonable hashrates is mandatory.


### --shaders XXX

is an option where you tell sgminer how many shaders your GPU has. This
helps sgminer try to choose some meaningful baseline parameters. Use
this table below to determine how many shaders your GPU has, and note
that there are some variants of these cards. If this is not set, sgminer
will query the device for how much memory it supports and will try to
set a value based on that.

SUMMARY: This will get you started but fine tuning for optimal
performance is required. Using --thread-concurrency is recommended
instead.


 GPU  | Shaders
 ---  | ---
 7750 | 512
 7770 | 640
 7850 | 1024
 7870 | 1280
 7950 | 1792
 7970 | 2048
 6850 | 960
 6870 | 1120
 6950 | 1408
 6970 | 1536
 6990 | (6970x2)
 6570 | 480
 6670 | 480
 6790 | 800
 6450 | 160
 5670 | 400
 5750 | 720
 5770 | 800
 5830 | 1120
 5850 | 1440
 5870 | 1600
 5970 | (5870x2)

These are only used as a rough guide for sgminer, and it is rare that
this is all you will need to set.


### --thread-concurrency

This tunes the optimal size of work that scrypt can do. It is internally
tuned by sgminer to be the highest reasonable multiple of shaders that
it can allocate on your GPU. Ideally it should be a multiple of your
shader count. vliw5 architecture (R5XXX) would be best at 5x shaders,
while VLIW4 (R6xxx and R7xxx) are best at 4x. Setting thread concurrency
overrides anything you put into --shaders and is ultimately a BETTER way
to tune performance.

SUMMARY: Spend lots of time finding the highest value that your device
likes and increases hashrate.


### -g

Once you have found the optimal shaders and intensity, you can start
increasing the -g value till sgminer fails to start. This is really only
of value if you want to run low intensities as you will be unable to run
more than 1.

SUMMARY: Don't touch this.


### --lookup-gap

This tunes a compromise between ram usage and performance. Performance
peaks at a gap of 2, but increasing the gap can save you some GPU
ram, but almost always at the cost of significant loss of hashrate.
Setting lookup gap overrides the default of 2, but sgminer will use the
--shaders value to choose a thread-concurrency if you haven't chosen
one.

SUMMARY: Don't touch this.


### Related parameters

--worksize XX (-w XX)
Has a minor effect, should be a multiple of 64 up to 256 maximum.
SUMMARY: Worth playing with once everything else has been tried but will
probably do nothing.


Overclocking for scrypt mining: First of all, do not underclock your
memory initially. Scrypt mining requires memory speed and on most, but
not all, GPUs, lowering memory speed lowers mining performance.


Second, absolute engine clock speeds do NOT correlate with hashrate. The
ratio of engine clock speed to memory matters, so if you set your memory
to the default value, and then start overclocking as you are running it,
you should find a sweet spot where the hashrate peaks and then it might
actually drop if you increase the engine clock speed further.


Third, the combination of motherboard, CPU and system ram ALSO makes a
difference, so values that work for a GPU on one system may not work for
the same GPU on a different system. A decent amount of system ram is
actually required for scrypt mining, and 4GB is suggested.


Finally, the power consumption while mining at high engine clocks,
very high memory clocks can be far in excess of what you might
imagine. For example, a 7970 running with the following settings:
--thread-concurrency 22392 --gpu-engine 1135 --gpu-memclock 1890 was
using 305W!


## Example: tuning a 7970

On linux run this command:

    export GPU_MAX_ALLOC_PERCENT=100

or on windows this:

    setx GPU_MAX_ALLOC_PERCENT 100

in the same console/bash/dos prompt/bat file/whatever you want to call it,
before running sgminer.

First, find the highest thread concurrency that you can start it at.
They should all start at 8192 but some will go up to 3 times that. Don't
go too high on the intensity while testing and don't change gpu threads.
If you cannot go above 8192, don't fret as you can still get a high
hashrate.

Delete any .bin files so you're starting from scratch and see what bins
get generated.

First try without any thread concurrency or even shaders, as sgminer
will try to find an optimal value

    sgminer -I 13

If that starts mining, see what bin was generated, it is likely the
largest meaningful TC you can set. Starting it on mine I get:

    scrypt130302Tahitiglg2tc22392w64l8.bin

See tc22392 that's telling you what thread concurrency it was. It should
start without TC parameters, but you never know. So if it doesn't, start
with --thread-concurrency 8192 and add 2048 to it at a time till you
find the highest value it will start successfully at.

Then start overclocking the eyeballs off your memory, as 7970s are
exquisitely sensitive to memory speed and amazingly overclockable but
please make sure it keeps adequately cooled with --auto-fan! Do it
while it's running from the GPU menu. Go up by 25 at a time every 30
seconds or so until your GPU crashes. Then reboot and start it 25 lower
as a rough start. Mine runs stable at 1900 memory without overvolting.
Overvolting is the only thing that can actually damage your GPU so I
wouldn't recommend it at all.

Then once you find the maximum memory clock speed, you need to find
the sweet spot engine clock speed that matches it. It's a fine line
where one more MHz will make the hashrate drop by 20%. It's somewhere in
the .57 - 0.6 ratio range. Start your engine clock speed at half your
memory clock speed and then increase it by 5 at a time. The hashrate
should climb a little each rise in engine speed and then suddenly drop
above a certain value. Decrease it by 1 then until you find it climbs
dramatically. If your engine clock speed cannot get that high without
crashing the GPU, you will have to use a lower memclock.

Then, and only then, bother trying to increase intensity further.

My final settings were:

    --gpu-engine 1141  --gpu-memclock 1875 -I 20

for a hashrate of 745kH.

Note I did not bother setting a thread concurrency. Once you have the
magic endpoint, look at what tc was chosen by the bin file generated
and then hard code that in next time (eg --thread-concurrency 22392) as
slight changes in thread concurrency will happen every time if you don't
specify one, and the tc to clock ratios are critical!

Good luck, and if this doesn't work for you, well same old magic
discussion applies, I cannot debug every hardware combo out there.

Your numbers will be your numbers depending on your hardware combination
and OS, so don't expect to get exactly the same results!
