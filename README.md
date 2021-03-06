# Performance tuning guide
The article describes ways to fine tune your linux machine and make it blazing fast.

I've made a linux set up on several of my laptops and tried to make it as fast as possible.

To remember what was done I decided to make a short list, which later grew and got some explanantions and extensions, as people were asking for addintional information.
 
So that is rather a memo for self, then a guidance, but it might be useful for power users and enthusiasts.

The result is quite impressive, - for my already outdated HP Revolve 810 G3 it takes less than a second to show a linux command line promt starting from cold.

Imagine that, the linux cold start is a way, way faster then the Windows wake up from hibernation. 


# Before you start

## Partiton table

**TL;DR: go for UEFI**

Before you start you need to create partition table and markup your hard drive. There are two different partition tables: newer GPT and legacy MBR.
If your system supports UEFI (and it likely is) and you are going to use UEFI for your linux installation it's better to create GPT partition table.
Apart from being more recent technology, UEFI allows us to load our kernel without bootloader, which will save us some seconds during the boot time.

The downside of UEFI is that it is proprietary and implemented by hardware vendors. It is quite often buggy and/or implemented with pity limitations, which could generate extra problems during installation. For example there was a firmware on HP 840 laptops, where path to bootloader was hardcoded to "EFI/Microsoft/Boot/bootmgfw.efi". Since linux is highly customizable and flexible, of course you can put your bootloader's file where it is hardcoded, but what a limitation!

BIOS systems from the other hand are conidered as legacy now, but well tested and simplier to set up.

## Filesystem
**TL;DR: SSD drive - F2FS, if you are newbie go for EXT4**

There are bunch of filesystems on linux, you might find dozens of benchmarks about it, but when these lines are written short summary is:

EXT4 good FS overall, most widely used, default choice and most distributives. Reliable, fast. There are some filesystems which sometimes outperforms EXT4 in particular (often exotic) kinds of load, but overall EXT4 is faster or same.


ReiserFS/Rieser4 is a promising FS with great features which makes it better chioce for certaint types of load. If you are an DevOps you already know what it is best for. Rarely needed for home use.


F2FS is a kind of acronym for "Flash Friendly File System" and have better results for different kinds of flash memory. Takes into consideration that for these kind of devices seek time is approaching to the speed of light. It also considers wearing-out feature of such devices and tries to distribute write and extend device lifetime. There are benchmarks that mobile phones converted to f2fs are faster and more responsive. That is my personal choice for installations running on SSD drives and sd-cards. It also supports on-the-fly LZO/LZ4 and other compression algorithms, which makes it even faster and reduce device worn even more.

XFS if you are a DevOps you know what it is for

JFFS/JFFS2 is somewhat exotic, but also desingned for NAND devices. Never tried this, if you did, please share.


# Your system installed
Congrats! Not everyone gets to this point.

Here I assume that yor system is installed the default way choosen by developers of your distro. 

For most of users I suggest "installing then tuning" way. 

## Low hanging fruits
### Kernel parameters
Kernel parameters are parameters which passed by bootloader to your kernel when it starts. For GRUB you list them in a GRUB configuration file. 
As it was mentioned before, UEFI systems are able to load your kernel without any bootloader, in that case you list your kernel parameters in your motherboard's NVRAM using efibootmgr tool or UEFI own command line.

You can start by tweaking kernel parameters. The best list of kernel parameters which speeds your linux up you can find at in the clearlinux repository: 
https://github.com/clearlinux-pkgs/linux/blob/master/cmdline

Clearlinux is a distribution maintained by Intel. Their distribution speed up (especially boot speedup) effort is huge and made by professionals, so we'd better to keep an eye on what they put in this file. But consider that clearlinux (for now) is aimed to be deployed as a virtual machine nodes, as well as emebedded software. So, having that, not everything which is good for virtual machine or embedded software good for bare metal machines.
You might find some options explanations below:

**mitigations=off**

Since linux 5.2. That disables spectre/meltdown vulnerability CPU mitigations. You decide yourself how secure should be your machine. If that is a server of any kind, I would recommend to keep all the mitigations enabled. The performance impact however is quite noticable and under certain types of load can even go up to 10-15%.
Worth to mention that vulnerability was discovered back in 2017, and that vaste majority of the CPUs produced between 1995 and 2017 are known to have it. 
The exploitation of vulnerability is somewhat advanced. If you read about how to exploit it on a wikipedia, you will see that the article is full of "if"s and "but"s.

**i915.enable_fbc=1**

That enables framebuffer compression which can reduce power consumption while reducing memory bandwidth needed for screen refreshes. ArchLinux wiki says 
"Framebuffer compression may be unreliable or unavailable on Intel GPU generations before Sandy Bridge (generation 6)."

**fastboot=1**

That feature makes boot smoother by doing some framebuffer manipulations so screen don't flicker upon boot.

**rootfstype=f2fs**

That is only needed if your kernel is compiled to not use initramfs image. If your root fs is EXT4, then parameter will look like rootfstype=ext4.
Skipping initramfs speeds up your boot process, and is described **HERE** (noteforself: make a link). 

**noinitrd**

That is only needed if your kernel is compiled to not use initramfs image.

**console=tty0**

With no console parameter the kernel will use the first virtual terminal, which is /dev/tty0. A user at the keyboard uses this virtual terminal by pressing Ctrl-Alt-F1.
If your computer contains a video card then we suggest that you also configure it as a console. This is done with the kernel parameter console=tty0.

Kernel messages will appear on both the first virtual terminal and the serial port. Messages from the init system and the system logger will appear only on the first serial port. This can be slightly confusing when looking at the attached monitor: the machine will appear to boot and then hang. Don't panic, the init system has started but is now printing messages to the serial port but is printing nothing to the screen. If a getty has been configured then a login: prompt will eventually appear on the attached monitor.


**cryptomgr.notests**

   [KNL] Disable crypto self-tests

**no_timer_check**

That option I think can be used only on virtual machine.
[X86,APIC] Disables the code which tests for broken timer IRQ sources.
						
						
**noreplace-smp**

The SMP alternative tables can be kept after boot and contain both
UP and SMP versions of the instructions to allow switching back to
SMP at runtime, when hotplugging in a new CPU, which is especially
useful in virtualized environments.


**page_alloc.shuffle=1**

 Randomization of the page allocator improves the average utilization of a direct-mapped memory-side-cache. See section
5.2.27 Heterogeneous Memory Attribute Table (HMAT) in the ACPI
6.2a specification for an example of how a platform advertises
the presence of a memory-side-cache. There are also incidental
security benefits as it reduces the predictability of page
allocations to compliment SLAB_FREELIST_RANDOM, but the
default granularity of shuffling on 4MB (MAX_ORDER) pages is
selected based on cache utilization benefits.

While the randomization improves cache utilization it may
negatively impact workloads on platforms without a cache. For
this reason, by default, the randomization is enabled only
after runtime detection of a direct-mapped memory-side-cache.
Otherwise, the randomization may be force enabled with the
'page_alloc.shuffle' kernel command line parameter.


**rcu_nocbs=0-64**

https://utcc.utoronto.ca/~cks/space/blog/linux/KernelRcuNocbsMeaning
rcu_nocbs=0-N
where N is the number of CPUs you have minus one
However, the rcu_nocbs=0-N setting we're using specifies all CPUs, so it shifts all RCU callbacks from softirq context during interrupt handling (on whatever specific CPU involved) to kernel threads (on any CPU). As far as I can see, this has two potentially significant effects, given that Matt Dillon of DragonFly BSD has reported an issue with IRETQ that completely stops a CPU under some circumstances. First, our Ryzen CPUs will spend less time in interrupt handling, possibly much less time, which may narrow any timing window required to hit Matt Dillon's issue. Second, RCU callback processing will continue even if a CPU stops responding to IPIs, although I expect that a CPU not responding to IPIs is going to cause the Linux kernel various other sorts of heartburn.

### Microcode updates







