---
title: "Coreboot-ing my X230"
date: 2021-02-04
layout: post
oneLiner: "Computer surgery is fun!"
splash: 
    src: /img/romFlashSplash.jpg
    alt: "a close-up of the SOIC-8 clip improperly seated on a ROM chip"
---

Well, it's been a year since my first post and *nearly* a year since I replaced the BIOS on my ThinkPad X230 with Coreboot.
The dust has settled, the machine has performed admirably, and I've had plenty of time to forget the important details.
This is mostly going to be a dump of the notes I took with some nicer images and longer explanations.
(Keeping some kind of notes/log/lab-book for this type of thing is a really helpful!)

While going through this process, I found it very frustrating how the necessary knowledge was scattered around online.
All the sources I found seemed to assume the reader was already familiar with electrical hardware shenaniganery.
My hope is that by narrating the whole process, it will help someone save a few days of cross-referencing web pages.
At the very least, it will stop me from forgetting it all.

# R60
I got an old ThinkPad R60 off of eBay for 20 buckarooskis, and I was going to use it as a practice computer. 
Unfortunately, I didn't realize that it didn't use a SOIC-8 ROM chip for the BIOS.
Of course, I only made this realization *after* I'd disassembled the whole thing, 
assuming ThinkPads would all use SOIC-8 ROM chips.
(To be fair, my c720p has a SOIC-8 ROM. So I had a basis for assuming pretty much every laptop used that form factor.)

![](img/R60_disassembled.jpg)
*my poor R60 waiting to be whole again... I fell ya, buddy.*

To flash a new BIOS to it, I'd need to solder to some pretty small components.
I decided I'd rather just skip the practice run and use it as a media/APRS server.
I still need to buy a drive for it, to make it useful. So, it's still in pile form.
I've also lost the screws... so it'll be a while before I boot this machine again.

# CH341A Fix 
According to [Chuck Nemeth's guide][nemeth x230] following his X230 Coreboot adventure, 
the CH341A programmers sometimes output 5 Vinstead of 3.3 V, depending on the exact version you get.
I checked mine by simply probing the output pins, and it was indeed running at 5 V!
The [Coreboot wiki][coreboot flash] says that potentials higher than 3.3 V can damage the ROM chips, so __this is a super important thing to check.__

I modded my CH341A to operate on 3.3 V instead of 5 V.
I found this [EEVBlog forum thread] had a better explanation of exactly *how* to do the mod though.
TODO: summarize the fix
I also covered the mod wire in kapton tape to prevent it from snagging on anything.

![](img/CH341A_modWire.jpg)
*my two absolutely gorgeous mod wires*

# x230
## Relevant Guides
- [Chuck Nemeth's x230 flashing walkthrough][nemeth x230] was definitely the overall best resource I found.
- [Coreboot's Wiki article on flashing an x230][coreboot flash] is supposedly the authoritative source... for the time being.
The wiki says it's being deprecated, but the new documentation system is still way behind the wiki and categorically sucks more in various ways.
The wiki page is really written for people who are already experienced with flashing ROMs.
- [The Sphinx documentation][sphinx docs] exists, but I didn't find it very helpful. Hopefully that will change in the future.
It also requires JS to search (seriously??) and the pages are overly short/long. 

## Package Installation
These are the packages I installed on my Acer c720p ("peppy"), following Chuck Nemeth's guide.
For reference, that machine was running 4.8.17-galliumos, which is basically Ubuntu 16.04 xenial.

- bison (a program for creating language parsers)
- bison-doc
- build-essential (already installed; for building C/C++ programs)
- curl (already installed; for downloading web pages)
- flashrom (for messing with ROMs)
- flex (another parsing tool)
- git (already installed; version control)
- gnat-5 (compiler for Ada)
- gnat-5-doc
- libncurses5-dev (libraries for drawing terminal GUIs)
- ncurses-doc
- m4 (already installed; macro processor)
- zlib1g-dev (already installed; decompression library)

According to the Coreboot README, these are also required:

- pkg-config (already installed; more make-ish functionality)
- iasl (ACPI support; I'm not sure if I needed this. Oh well.)
- libssl-dev (already installed; library for SSL)

## Setting Up Coreboot
### Clone All the Things
This is straightforward. You just need to know to grab the submodules.

```
git clone https://review.coreboot.org/coreboot.git # clone the main Coreboot repo
cd coreboot # enter that repo
git submodule update --init --checkout # clone all the submodules that Coreboot depends on
```

### Building Coreboot
`make help` is... helpful. ( ͡~ ͜ʖ ͡°)

Note that the `CPUS=$(nproc)` portion of the following snippet is just saying to use all available threads for compilation.
In theory, I think building the cross compilers should be unnecessary, 
since the building computer and the target computer use the same CPU architecture.
However, the Coreboot Makefile claims that there are lots of weird things in the Coreboot
compilation process that are usually broken by patches applied to the compilers that come
with Linux distros. 
So, they recommend just building a cross compiler specifically for Coreboot.
Note that these commands will download the source files for a bunch of dependencies, so
you need to have an internet connection.
On my computer, I think compiling the 32 bit cross compiler took around 40 minutes.

```
make crossgcc-i386 CPUS=$(nproc) # build Coreboot for 32 bit processors
make crossgcc-x64 CPUS=$(nproc) # build Coreboot for 64 bit processors
cd util/ifdtool
make # build ifdtool to extract blobs from our BIOS
cd ../nvramtool
make # build nvramtool to read/write coreboot parameters
```

Chuck mentions `nvramtool`, but nothing about building it. 
So, I built it...
Actually, for ifdtool and nvramtool, I had to cd into their directories and run `make`.

## Disassembly
Thinkpads are arguably the easiest laptops to take apart and maintain.
For this procedure, the process is especially easy.

### Removing the Battery
You start by turning off the machine, unplugging it, and removing the battery.
There are two switches on the bottom with nearby silk-screening that read `🔓 🔒 ◀ 1` and `2 ▶ 🔒 🔓`.
Slide these to the `🔓` position (towards the outside of the case) and slide out the battery.
You'll need to actively hold switch `2` to do this.

### Removing the Keyboard and Palm Rest
Lenovo has [a nice video on how to remove the keyboard and palm rest][keyboard removal].
Remove the two screws marked with nearby `⌨` (keyboard symbol) silk-screening 
and the five screws with nearby `□` (square with four dots at the bottom) silk-screening.
Next, press your palms down on the keyboard and slide it up.
(This is a little different than what they show in the video, but it's less scary.)
You can push on the lip of plastic just below the space bar to get a little extra force if you need it.
They keyboard should unseat fairly easily.

![](img/x230_removeScrews.jpg)
*the screws you need to remove*

At this point, you can move the keyboard slightly out of the way, and disconnect the low-profile connector.
However, unless you're doing more than just flashing the firmware, I recommend leaving the keyboard plugged in.
Those little connectors aren't designed for lots of plug/unplug cycles.
Also, if you end up needing multiple flash attempts like I did, 
you'll save a fair bit of time by not fussing with the keyboard, palm rest, and SOIC-8 clip every attempt.

The palm rest uses a different kind of connector. 
You need to flip up the white plastic lever and pull out the cable using the blue bit of plastic.
Don't try to pull out the cable without flipping the lever! That could damage the cable.
Again, I found it was more convenient to leave this plugged in, similar to the keyboard connector.

### Attaching the SOIC-8 Clip

## Reading ROMs
### Top Chip
Checking the connection to the chip:

```
~/Documents/x230
$ sudo flashrom -p ch341a_spi
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
No EEPROM/flash device found.
Note: flashrom can never write if the flash chip isn't found automatically.
exit 1
~/Documents/x230
$
```

As you can see, I had some trouble getting it to see the top chip. 
So, I moved on to the bottom chip, since succesfully reading that would at least give me a
hint as to what *isn't* going wrong with the top chip.


### Bottom Chip

Checking the connection to the chip:
```
~/Documents/x230
$ sudo flashrom -p ch341a_spi
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L6405" (8192 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L6405D" (8192 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L6406E/MX25L6408E" (8192 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L6436E/MX25L6445E/MX25L6465E/MX25L6473E" (8192 kB, SPI) on
ch341a_spi.
Multiple flash chip definitions match the detected chip(s): "MX25L6405", "MX25L6405D",
"MX25L6406E/MX25L6408E", "MX25L6436E/MX25L6445E/MX25L6465E/MX25L6473E"
Please specify which chip definition to use with the -c <chipname> option.
exit 1
~/Documents/x230
$
```

Reading and verifying the ROM:
```
~/Documents/x230
$ sudo flashrom -p ch341a_spi -r factory_bottom-1.bin -c "MX25L6406E/MX25L6408E"
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L6406E/MX25L6408E" (8192 kB, SPI) on ch341a_spi.
Reading flash... done.
67.5 s
~/Documents/x230
$ sudo flashrom -p ch341a_spi -r factory_bottom-2.bin -c "MX25L6406E/MX25L6408E"
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L6406E/MX25L6408E" (8192 kB, SPI) on ch341a_spi.
Reading flash... done.
67.5 s
~/Documents/x230
$ md5sum factory_bottom-*
e960653ffe218c95863a1fc7f82a99b6  factory_bottom-1.bin
e960653ffe218c95863a1fc7f82a99b6  factory_bottom-2.bin
~/Documents/x230
$
```

### Top Chip: Take Two
I tried re-seating the clip many times, but I kept getting the same result as before. 
After reading the [UpWiki] page on flashing, a tried very carefully re-seating the clip
and wishing really hard. 
That seemed to work:
```
~/Documents/x230
$ sudo flashrom -p ch341a_spi
[sudo] password for joedang:
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L3205(A)" (4096 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L3205D/MX25L3208D" (4096 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L3206E/MX25L3208E" (4096 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L3273E" (4096 kB, SPI) on ch341a_spi.
Multiple flash chip definitions match the detected chip(s): "MX25L3205(A)",
"MX25L3205D/MX25L3208D", "MX25L3206E/MX25L3208E", "MX25L3273E"
Please specify which chip definition to use with the -c <chipname> option.
exit 1
~/Documents/x230
$
```

The fact that this magically worked after reseating really makes me think I should check
the connection before flashing.
(flashrom probably does this, but I'm paranoid.)

After checking the connection, I verified the ROM:
```
~/Documents/x230
$ sudo flashrom -p ch341a_spi -r factory_top-1.bin -c "MX25L3206E/MX25L3208E"
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L3206E/MX25L3208E" (4096 kB, SPI) on ch341a_spi.
Reading flash... done.
34.4 s
~/Documents/x230
$ sudo flashrom -p ch341a_spi -r factory_top-2.bin -c "MX25L3206E/MX25L3208E"
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L3206E/MX25L3208E" (4096 kB, SPI) on ch341a_spi.
Reading flash... done.
35.1 s
~/Documents/x230
$ md5sum factory_top-*
35fac0c1cbc4940b99681c2a1abd945a  factory_top-1.bin
35fac0c1cbc4940b99681c2a1abd945a  factory_top-2.bin
~/Documents/x230
$
```

```
$ cat factory_bottom-1.bin factory_top-1.bin > x230-bios.rom
```

After concatenating the binaries (I don't think Chuck ever mentions how he knows the
bottom chip is the start of the ROM...), I took a look at the result using binwalk.
There are some interesting things in there. 
It could be fun to dissect that more later...

```
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
270030        0x41ECE         YAFFS filesystem
562858        0x896AA         Unix path: /ZAYRqQrgY/O/eskJ/FOu1Betigl
2440415       0x253CDF        Unix path: /schemas.dmtf.org/wbem/wsman/1/wsman/secprofile/http
8388608       0x800000        UEFI PI firmware volume
8388713       0x800069        LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, uncompressed size: 7864336 bytes
11376693      0xAD9835        Certificate in DER format (x509 v3), header length: 4, sequence length: 1495
11378236      0xAD9E3C        Certificate in DER format (x509 v3), header length: 4, sequence length: 1552
11379836      0xADA47C        Certificate in DER format (x509 v3), header length: 4, sequence length: 942
11380958      0xADA8DE        Certificate in DER format (x509 v3), header length: 4, sequence length: 937
11381943      0xADACB7        Certificate in DER format (x509 v3), header length: 4, sequence length: 1512
11383531      0xADB2EB        Certificate in DER format (x509 v3), header length: 4, sequence length: 935
11399535      0xADF16F        GIF image data, version "89a", 600 x 260
11405938      0xAE0A72        Copyright string: "Copyright (C) By Lenovo"
11430260      0xAE6974        GIF image data, version "89a", 546 x 308
11433951      0xAE77DF        Copyright string: "Copyright (C) By Lenovo"
11608388      0xB12144        Certificate in DER format (x509 v3), header length: 4, sequence length: 1495
11609931      0xB1274B        Certificate in DER format (x509 v3), header length: 4, sequence length: 1552
11611531      0xB12D8B        Certificate in DER format (x509 v3), header length: 4, sequence length: 942
11612653      0xB131ED        Certificate in DER format (x509 v3), header length: 4, sequence length: 937
11613638      0xB135C6        Certificate in DER format (x509 v3), header length: 4, sequence length: 1512
11615226      0xB13BFA        Certificate in DER format (x509 v3), header length: 4, sequence length: 935
11796480      0xB40000        UEFI PI firmware volume
11796636      0xB4009C        Microsoft executable, portable (PE)
11804292      0xB41E84        Microsoft executable, portable (PE)
11807452      0xB42ADC        UEFI PI firmware volume
11822188      0xB4646C        Microsoft executable, portable (PE)
11837668      0xB4A0E4        Microsoft executable, portable (PE)
11847420      0xB4C6FC        SHA256 hash constants, little endian
11911724      0xB5C22C        Microsoft executable, portable (PE)
11920452      0xB5E444        Microsoft executable, portable (PE)
11922460      0xB5EC1C        Microsoft executable, portable (PE)
11924851      0xB5F573        mcrypt 2.2 encrypted data, algorithm: blowfish-448, mode: CBC, keymode: 8bit
11980724      0xB6CFB4        SHA256 hash constants, little endian
12249676      0xBAEA4C        SHA256 hash constants, little endian
12283528      0xBB6E88        Microsoft executable, portable (PE)
12286016      0xBB7840        Microsoft executable, portable (PE)
12289020      0xBB83FC        Microsoft executable, portable (PE)
12289960      0xBB87A8        Microsoft executable, portable (PE)
12292476      0xBB917C        Microsoft executable, portable (PE)
12293516      0xBB958C        Microsoft executable, portable (PE)
12302124      0xBBB72C        Microsoft executable, portable (PE)
12303596      0xBBBCEC        Microsoft executable, portable (PE)
12315116      0xBBE9EC        Microsoft executable, portable (PE)
12316708      0xBBF024        Microsoft executable, portable (PE)
12317684      0xBBF3F4        Microsoft executable, portable (PE)
12319520      0xBBFB20        Microsoft executable, portable (PE)
12322432      0xBC0680        Microsoft executable, portable (PE)
12323412      0xBC0A54        Microsoft executable, portable (PE)
12325144      0xBC1118        Microsoft executable, portable (PE)
12338028      0xBC436C        Microsoft executable, portable (PE)
12341628      0xBC517C        Microsoft executable, portable (PE)
```

## Extracting Blobs
I find it curious that ifdtool doesn't sort the "regions" by their physical addresses.
```
~/Documents/x230
$ cd coreboot/util/ifdtool
master ~/Documents/x230/coreboot/util/ifdtool
$ ln -s ../../../bios-backup/x230-bios.rom
?,master ~/Documents/x230/coreboot/util/ifdtool
$ ./ifdtool -x x230-bios.rom
File x230-bios.rom is 12582912 bytes
  Flash Region 0 (Flash Descriptor): 00000000 - 00000fff
  Flash Region 1 (BIOS): 00500000 - 00bfffff
  Flash Region 2 (Intel ME): 00003000 - 004fffff
  Flash Region 3 (GbE): 00001000 - 00002fff
  Flash Region 4 (Platform Data): 00fff000 - 00000fff (unused)
?,master ~/Documents/x230/coreboot/util/ifdtool
$ ls *.bin
flashregion_0_flashdescriptor.bin  flashregion_1_bios.bin
flashregion_2_intel_me.bin  flashregion_3_gbe.bin
?,master ~/Documents/x230/coreboot/util/ifdtool
$ mkdir -p ../../3rdparty/blobs/mainboard/lenovo/x230
?,master ~/Documents/x230/coreboot/util/ifdtool
$ mv *.bin ../../3rdparty/blobs/mainboard/lenovo/x230
?,master ~/Documents/x230/coreboot/util/ifdtool
$ cd ../../3rdparty/blobs/mainboard/lenovo/x230
?,7ad2d22 ~/Documents/x230/coreboot/3rdparty/blobs/mainboard/lenovo/x230
$ qmv
Plan is valid.

flashregion_0_flashdescriptor.bin -> descriptor.bin
flashregion_1_bios.bin -> bios.bin
flashregion_2_intel_me.bin -> me.bin
flashregion_3_gbe.bin -> gbe.bin
  Regular rename

flashregion_0_flashdescriptor.bin -> descriptor.bin
flashregion_1_bios.bin -> bios.bin
flashregion_2_intel_me.bin -> me.bin
flashregion_3_gbe.bin -> gbe.bin
```

## Configuring Coreboot
Check out the files in `nconfigScreens/` for how I configured Coreboot through 
the ncurses interface.

Diverging from Chuck's setup a little, I decided to add a custom bootsplash
screen.
I placed my `bootslpash.jpg` in `coreboot/`.
I also set the maximum width and height to match the resolution of the x230.

## Making Coreboot
I got an error when running `make` due to the `bootsplash.jpg` not having any
make rule. 
I just added a trivial rule to make this go away. (I made the image manually. I
don't want `make` to mess with it, and I don't want to dig through the makefiles
to figure out how to tell it that the bootsplash image is a source file.)
```make
bootsplash.jpg:
	touch bootsplash.jpg
```

I had another problem:
```
    MAKE       /home/joedang/Documents/x230/coreboot/vboot_lib/libvboot_host.a
vboot hash algos built with tight loops (slower, smaller code size)
    CC            cgpt/cgpt_add.o
gcc: error: unrecognized command line option '-Wimplicit-fallthrough'
Makefile:1054: recipe for target '/home/joedang/Documents/x230/coreboot/build/util/vboot_lib/cgpt/cgpt_add.o' failed
make[1]: *** [/home/joedang/Documents/x230/coreboot/build/util/vboot_lib/cgpt/cgpt_add.o] Error 1
util/cbfstool/Makefile.inc:132: recipe for target '/home/joedang/Documents/x230/coreboot/build/util/vboot_lib/libvboot_host.a' failed
make: *** [/home/joedang/Documents/x230/coreboot/build/util/vboot_lib/libvboot_host.a] Error 2
```
The `-Wimplicit-fallthrough` option to GCC seems to be a new addition that just 
warns about oddly/dangerously written case statements.
I think I should be able to live with any such warnings.
So, I'm removing that flag from `coreboot/3rdparty/vboot/Makefile`.
We'll see if that works...
Nope. :(
Coreboot uses the `-Werror` flag a lot, which turns all the warnings into
errors.
Without the `-Wimplicit-fallthrough` flag, all the little annotations to say "I
know I wrote a dangerous case statement, trust me" are unrecognized syntax.
```
host/lib/crossystem.c: In function 'GetVdatInt':
host/lib/crossystem.c:314:4: error: empty declaration [-Werror]
    __attribute__ ((fallthrough));
        ^
```
Grep-ing for the Werror flag, there are comments that say it's relied upon as a
feature... so, I can't just gut it from all the makefiles. 

Hmmm. I'm not sure why this error is occuring. I kind of wonder if it's trying
to use my out-of-date GCC installation, rather than the one installed by
Coreboot. 
That would kind of make sense, since it's in `3rdparty/`.
vboot isn't even checked in my configuration, so I wonder if I can just modify
the makefile to skip vboot.

## Flashing Coreboot
I re-read and verified the contents of each chip after seating the clip and before flashing the new ROMs.
This was just to be extra paranoid and make sure the clip was properly seated. 

### Top Chip
```
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ sudo flashrom -p ch341a_spi -r factory_top-3.bin -c "MX25L3206E/MX25L3208E"
[sudo] password for joedang:
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L3206E/MX25L3208E" (4096 kB, SPI) on ch341a_spi.
Reading flash... done.
38.2 s
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ md5sum factory_top-3.bin
35fac0c1cbc4940b99681c2a1abd945a  factory_top-3.bin
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ md5sum ../../bios-backup/factory_top-1.bin
35fac0c1cbc4940b99681c2a1abd945a  ../../bios-backup/factory_top-1.bin
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ sudo flashrom --chip "MX25L3206E/MX25L3208E" --programmer ch341a_spi --write
coreboot-top.rom
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L3206E/MX25L3208E" (4096 kB, SPI) on ch341a_spi.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
138.4 s
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$
```

bottom chip:
```
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ sudo flashrom -p ch341a_spi
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.

cb_in: error: LIBUSB_TRANSFER_ERROR
ch341a_spi_spi_send_command: Failed to read 5 bytes

cb_in: error: LIBUSB_TRANSFER_NO_DEVICE

cb_out: error: LIBUSB_TRANSFER_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 36 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 36 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 36 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 36 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 36 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 40 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 38 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 37 bytes
ch341a_spi_spi_send_command: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
ch341a_spi_spi_send_command: Failed to write 39 bytes
No EEPROM/flash device found.
Note: flashrom can never write if the flash chip isn't found automatically.
enable_pins: failed to submit OUT transfer: LIBUSB_ERROR_NO_DEVICE
enable_pins: Failed to write 4 bytes
Could not disable output pins.
exit 1
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ sudo flashrom -p ch341a_spi
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L6405" (8192 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L6405D" (8192 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L6406E/MX25L6408E" (8192 kB, SPI) on ch341a_spi.
Found Macronix flash chip "MX25L6436E/MX25L6445E/MX25L6465E/MX25L6473E" (8192 kB, SPI) on ch341a_spi.
Multiple flash chip definitions match the detected chip(s): "MX25L6405", "MX25L6405D", "MX25L6406E/MX25L6408E", "MX25L6436E/MX25L6445E/MX25L6465E/MX25L6473E"
Please specify which chip definition to use with the -c <chipname> option.
exit 1
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ sudo flashrom -p ch341a_spi -r factory_bottom-3.bin -c "MX25L6406E/MX25L6408E"
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L6406E/MX25L6408E" (8192 kB, SPI) on ch341a_spi.
Reading flash... done.
67.5 s
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ md5sum factory_bottom-3.bin
e960653ffe218c95863a1fc7f82a99b6  factory_bottom-3.bin
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ md5sum ../../bios-backup/factory_bottom-1.bin
e960653ffe218c95863a1fc7f82a99b6  ../../bios-backup/factory_bottom-1.bin
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$ sudo flashrom --chip "MX25L6406E/MX25L6408E" --programmer ch341a_spi --write coreboot-bottom.rom
flashrom v0.9.9-rc1-r1942 on Linux 4.8.17-galliumos (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Macronix flash chip "MX25L6406E/MX25L6408E" (8192 kB, SPI) on ch341a_spi.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
204.1 s
M,?,dev_joe ~/Documents/x230/coreboot/romsFromXubuntu
$
```
As you can see, I had some trouble connecting to the bottom chip at first. 
Judging by the error messages, it seemed to be something to do with the ch341a_spi, since it's a USB device.
So, I just unplugged and replugged it. That fixed the problem. 

Booting into Xubuntu with the stock BIOS and my preferred BIOS settings, I get a login in 23 seconds.


# Links
[nemeth x230]: https://www.chucknemeth.com/flash-lenovo-x230-coreboot/
[coreboot flash]: https://www.coreboot.org/Board%3Alenovo/x230
[EEVBlog forum thread]: https://www.eevblog.com/forum/repair/ch341a-serial-memory-programmer-power-supply-fix/
[UpWiki]: https://wiki.up-community.org/BIOS_chip_flashing
[sphinx docs]: https://doc.coreboot.org/mainboard/lenovo/x230s.html
[keyboard removal]: https://youtu.be/RCBRVuwr06A
