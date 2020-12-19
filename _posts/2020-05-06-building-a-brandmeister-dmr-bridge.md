---
layout: post
title:  "Building a BrandMeister DMR software bridge"
date:   2020-05-06 00:00:00 +0000
category: Programming
tags: [HAM]
---

DMR (short for Digital Mobile Radio) is a digital mode that is used on the amateur
radio bands. Its main advantage over regular, analog radio is that the quality degrades
better than analog radio: when moving further away from an analog radio, the quality
decreases. With digital radio, the quality more or less stays the same up until the
cut-off point. Another advantage is that it has two timeslots, so two users can use the
same frequency without interfering with each other.

DMR is also an open standard, so in theory multiple manufacturers should be able to
build DMR radios. This happens, but the vocoder (voice encoder/decoder) is not open:
it uses AMBE as its codec, which is proprietary and patented. This is less than ideal
in my opinion: I think it goes against the radio amateur spirit to use a closed,
proprietary codec. Other codecs should be used, such as the open-source Codec 2.
The main issue holding back innovation is compatibility with existing radios: if you had a
hand-held that only talks Codec 2, it would not be able to communicate with all
other radios still using AMBE as vocoder.

In amateur radio, there's a concept of repeaters: they are often stationed on top
of a high vantage point, and relay the messages radio amateurs send to them. Hams
use repeaters to extend their coverage. DMR also has a concept of repeaters:
they take messages for one talk group and send them over the internet to other repeaters
that are listening to this talk group. These repeaters then retransmit the message,
enabling even radio amateurs that are on different continents to communicate.

One of these repeater networks, and by far the largest, is the BrandMeister network.
It has repeaters all over the globe that are each linked to their country's master server.
These master servers are in turn also linked to each other. It's possible to
add your own hotspot (a mini-repeater with very low transmitting power) to the network.
By doing this, one could have DMR coverage in their house, even if they aren't able
to reach other repeaters.

The issue me and some friends have been dealing with is that without a radio, it's
almost impossible to listen to DMR calls: hose.brandmeister.network is a web interface
for doing exactly that, but it's often very laggy and frequently just doesn't work.
In the past we solved this by having one person hold their radio near their computer's
microphone and re-broadcast the audio to our Mumble server (Mumble is a free, open
source, low latency, high quality voice
chat application). This is less than ideal, because this person would appear much
louder, and because the already pretty low audio quality would get even worse.
This led me to the idea of hooking up the BrandMeister network to Mumble directly:
that way, there is almost no latency and the quality is optimal.

BrandMeister has two ways to do this: they have a Simple External Application API,
and an Open DMR Terminal protocol. Both are built on top of their self-written
Rewind-protocol, on top of UDP. The Simple External Application API is, as I later
found out, meant for bridges to different services, so this would be the appropriate
protocol to use. It however requires a separate set of credentials you need to ask for,
and the master server of your country needs to be configured to allow this. The Open
DMR Terminal protocol is meant for hotspots, and only requires a password you can
get from the BrandMeister self-care settings panel. The two protocols are very similar,
so adapting the current bridge to use SEA in the future should be easy.

The entire pipeline:

## `./dmr-brandmeister | python2 decode72to49.py | python3 to_ambe_format_file.py | qemu-arm md380tools/emulator/md380-emu -d | sox --buffer 256 -r 8000 -e signed-integer -L -b 16 -c 1 -t raw /dev/stdin -t raw -r 48000 /tmp/dmr.fifo`

The software I adapted to get the AMBE-encoded audio is BrandMeister's [callrec](https://github.com/BrandMeister/callrec).
This piece of Go software connects to the BrandMeister network and gets all AMBE
packets destined for a configured talk group. Each audio packet received from this tool
consists of three forward error corrected AMBE frames. Each frame is 9 bytes (72 bits) long,
but I saw most AMBE decoders take 49-bit packets. This puzzled me for a while, but
I eventually figured out the 72 bit packets are still wrapped in Forward Error Correction.
The BrandMeister wiki calls this "mode 33", but I didn't really find any information about this
anywhere online.

I then removed the FEC with some code adapted from [dmr_utils](https://github.com/n0mjs710/dmr_utils/).
This turns the packets from 72 bits long into 49 bits long packets.
The next step in the chain would be an AMBE decoder. This could
be a dongle with an AMBE vocoder chip on it, or it can be software; when choosing
the latter, please verify that this is legal where you live since the AMBE codec
is patented. In Belgium, patent law doesn't apply for non-commercial actions in a
private setting, but I am not a lawyer, and certainly not yours, so verify this before
decoding AMBE in software.

The software decoder I used is based on the MD380 firmware: Travis Goodspeed extracted
the firmware of this radio and then packed the software AMBE decoder into an ARM ELF
(ELF is the Linux binary format, like you have EXE on Windows). This binary can then
be executed with `qemu-arm`, a userland emulator for ARM binaries, or natively executed
on single board computers like the Raspberry Pi. Before sending the packets to this tool,
they were first formatted into the `.amb` file format (the same format that DSD, Digital Speech
Decoder, also uses).

This results in a stream of 8000 bits per second of raw, PCM audio. This audio then gets upsampled
to 48000 bits per second and sent to a FIFO (a special sort of file that is used to exchange data between
processes). We upsample the audio for two reasons: by default, Mumble uses this bitrate and because
effects applied on this audio stream will sound better.
A small Ruby application then sends this to the Mumble server (with the
help of the mumble-ruby library).

That concludes the whole setup of bridging DMR to Mumble. The repositories are [here for the SEA protocol](https://github.com/redfast00/brandmeister-dmr-sea)
and [here for the Open DMR terminal protocol](https://github.com/redfast00/brandmeister-dmr-opendmr). I recommend using the SEA protocol repo if at all possible (it has better documentation and setup instructions).

If you have any questions, remarks or find this article useful, please send me an email.

73,
ON3WPI
