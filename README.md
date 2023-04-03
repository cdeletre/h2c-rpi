
# Setup for recording and streaming with H2C-RPI-B01 board and a Raspberry Pi (HDMI to CSI adapter)

## The story

Years ago I bought a HDMI to CSI adaptor board from a well know chinese emarket place because it was cheap and I as interrested to play with such thing. It is as a copy of the [Auvidea B101 HDMI to CSI-2 bridge](https://auvidea.eu/product-category/cameras-2/csi2bridge/hdmi2csi/) and I already knew at this time that the sound may not be working as easy as with the orignal one. Time has passed and as my children started to play with HDMI enable videogames I thought it could be fun to record their gameplay. Thus I've made some archeology and find back the board, swimming amongst my other *have-to-try things*. It wasn't the hardest part. It was now time to find the forgotten documentation and all of that stuff that you need to make something work. Thanks to my order history on the emarket place I was able to get back some clues and after a lot of googling and google translating from asian language I found the all-in-one package howto make the sound work [here](https://www.mzyy94.com/blog/2022/03/01/hdmi-input-board-with-audio/). If you can't read japanese in a blink of an eye, like me, [here](https://www-mzyy94-com.translate.goog/blog/2022/03/01/hdmi-input-board-with-audio/?_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=fr&_x_tr_pto=wapp) is the translation.

With the hope of getting it work with the sound I've started to build my recording/streaming setup. You'll find here all my notes so that you can reproduce the hardware patching to enable the sound support and also how to get a stream of the HDMI in OBS Studio.

Of course if you own a HDMI to CSI board with the audio already working you can use this guide and just skip the hardware patching part.

## Known limitations
With a 2 lines CSI, considering you're using the UYVY mode, you can capture the following HDMI resolutions
- 1080p50
- 720p60

So far I was able to record 1080p30 and 720p60 with success. I haven't played yet a lot with 1080p50 so I cannot say.

A 4 lines CSI would support up to 1080p60. 4 lines CSI port is only available on the Raspberry Pi CM4 and Nvidia Jetson as far as I know.


## I2S link

You can follow these [instructions](https://wiki.blicube.com/blikvm/en/hdmi-csi-i2s/#features) to connect the I2S link between the HDMI2CSI board and the Raspberry Pi. You can double-check the pin positions on the Raspberry with [pinout.xyz](https://pinout.xyz/). You'll have to identidy OSCK, SCK, WFS, SD and optionnaly GND (it should be already connected with the CSI ribbon) on the HDMI2CSI board. On my HDMI2CSI board the pinout was labeled.

The I2S link, with the proper configuration done on the Raspberry Pi, enables the Raspberry to receive the audio samples.

## Enabling video and audio device

To enable the video and audio device on the Raspberry Pi it is really easy. You just need to enable the device tree overlay in the `/boot/config.txt`. Just pay attention to insert them in the global section, not in a specific section like Raspberry Pi 4 only.

     # TC358743
     dtoverlay=tc358743
     dtoverlay=tc358743-audio

You can find more details in this [thread](https://forums.raspberrypi.com/viewtopic.php?t=281972) on the Raspberry Pi forum.

## Software package for streaming

You can use any streaming software you want as soon as it support v4l2 device. VLC does for example. In my setup I've used Gstreamer because I wanted to learn how to use a new tool and it is also well design for this task. 

Apart from the usual `apt update && apt upgrade` you'll need no more than

     apt install tmux 
     apt install gstreamer1.0-tools
     apt install gstreamer1.0-plugins-good

You may substitue tmux by screen or another terminal multiplexer of your choice. You can also ignore it for the beginning if you're too afraid of using a terminal multiplexer.

## Additionnal requirements

You'll need an EDID txt file to setup the HDMI to CSI adapter before you can connect it to a HDMI source. This EDID file will enable the video and audio mode supported. I've used with success this file `1080P60EDID.txt`

     00ffffffffffff005262888800888888
     1c150103800000780aEE91A3544C9926
     0F505400000001010101010101010101
     010101010101011d007251d01e206e28
     5500c48e2100001e8c0ad08a20e02d10
     103e9600138e2100001e000000fc0054
     6f73686962612d4832430a20000000FD
     003b3d0f2e0f1e0a202020202020014f
     020322444f841303021211012021223c
     3d3e101f2309070766030c00300080E3
     007F8c0ad08a20e02d10103e9600c48e
     210000188c0ad08a20e02d10103e9600
     138e210000188c0aa01451f01600267c
     4300138e210000980000000000000000
     00000000000000000000000000000000
     00000000000000000000000000000015

## Enabling USB networking (Raspberry Pi zero and zero 2 only)

Enabling USB networking on a Raspberry Pi zero (and zero 2) is easy. First you need to enable the device tree overlay in `/boot/config.txt` by adding the following lines. Here again, pay attention to insert them in the global section:

     # USB gadget network adapter
     dtoverlay=dwc2

Then you must enable the module loading in `/boot/cmdline.txt` by adding `modules-load=dwc2,g_ether`. Your new `cmdline.txt` should look like

     console=serial0,115200 modules-load=dwc2,g_ether console=tty1 root=PARTUUID=c0ff33d2-02 rootfstype=ext4 fsck.repair=yes rootwait

Also you'll need to configure a static IP address on this new network device. Do to so add the following line in `/etc/dhcpcd.conf`

     interface usb0
     static ip_address=192.168.0.1/24

Shutdown the Raspberry Pi with a gentle `sudo shutdown -fh now` (wait until it's not blinking anymore). Unplug the USB cable from the power adapter and from the PWR_IN micro-USB port of the Raspberry Pi zero. Connect the USB cable to the second micro-USB port of the Raspberry (USB-OTG port). Power it from a USB port of the computer (Here it's the Streaming station)

 A RNDIS/Ethernet Gadget netwok interface must have appeared on the computer. As soon as a static address IP for this network interface is configured, eg. 192.168.0.2/24, you should be able to communicate with the Raspberry Pi.

     user@computer ~ % ping 192.168.0.2
     PING 192.168.0.2 (192.168.0.2): 56 data bytes
     64 bytes from 192.168.0.2: icmp_seq=0 ttl=64 time=0.175 ms
     64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.198 ms

You may need to set the MAC addresses permanent instead of randomly generated on each boot. This is welcome when your computer doesn't reapply the IP address each time the Raspberry Pi boots. To do so add `g_ether.host_addr` (computer side) and `g_ether.dev_addr` (Raspberry Pi side) parameters with your chosen MAC addresses (tip: you can simply copy the random ones) in `/boot/cmdline.txt` and reboot the Rapsberry Pi. Your new `cmdline.txt` should look like

     console=serial0,115200 modules-load=dwc2,g_ether g_ether.host_addr=ba:dc:0f:fe:e2:64 g_ether.dev_addr=ba:dc:0f:fe:e3:d2 console=tty1 root=PARTUUID=c0ff33d2-02 rootfstype=ext4 fsck.repair=yes rootwait

## What the sound ?

Indeed, as I highly suspected, after following the standard installation instruction for such board, the sound is not working on my board. Even if I have carrefuly soldered the I2S bus between the HDMI2CSI board and the Raspberry Pi (GPIO HAT). When I've tried to record audio it recorded almost nothing but (a little) garbage:

     pi@hdmipi2:~ $ gst-launch-1.0 alsasrc device=hw:tc358743 ! audio/x-raw,rate=48000,channels=2 ! audioconvert ! avenc_aac bitrate=48000 ! aacparse ! queue ! matroskamux name=mux ! filesink location=foo.mkv
     Setting pipeline to PAUSED ...
     Pipeline is live and does not need PREROLL ...
     Pipeline is PREROLLED ...
     Setting pipeline to PLAYING ...
     New clock: GstAudioSrcClock
     Redistribute latency...
     0:00:00.7 / 99:99:99.

This issue is confirmed and detailed by [mzyy94](https://twitter.com/mzyy94). TLDR: on some cheap copies, the APLL routing is not properly routed which cause the I2S bus not delivering any data even if all flags are green from I2C device on the CSI bus (audio presence detected, audiorate indicated).

## Hardware patching

Honestly, I could have bought a cheap plug and play HDMI to USB adapter for the same budget but I liked the challenge to patch my disabled HDMI2CSI board. Also all the hardest part of designing the patch was already done by [mzyy94](https://twitter.com/mzyy94) who has exactly the same board than me. I understood that the first hardware patch has been done in 2020 by a certain idcidc who posted on the raspberry forum [his story](https://forums.raspberrypi.com/viewtopic.php?p=1695619). If you want to see the missing pictures, at the date of writing this page, you can open the [same page with google translate](https://forums-raspberrypi-com.translate.goog/viewtopic.php?p=1695619&_x_tr_sl=auto&_x_tr_tl=en&_x_tr_hl=fr&_x_tr_pto=wapp#p1695619) which has still them in cache.

On my journey of hardware patching I've fight a lot with my *too* cheap soldering iron as you can read in my [thread on Twitter](https://twitter.com/kotzebuedog/status/1635338074468282368) (in french). So get a good one before you jump into this.  Basically you'll need:
 - a good soldering iron
 - solder
 - a protoboard
 - the component from th BOM
 - enameled wire like the ones in earphones cables
 - desoldering wire to fix your mistakes without killing the board
 - patience
 - another bit of patience, we never know

 If you go like me for SMD component, you may need for a cleaner work 
 - a SMD soldering station to remove the two capacitors on the board
 - SMD handling tools (SMD tips) because you'll know when you'll have 0805 SMD under your eyes
 - magnifying glass or a digital microscope for circuit board

## Hardware patch circuit

The circuit is very simple and requires few components

```
2V5
----------+
          |C
      +---+----+
      |        |
      |   Q1   |
      ++------++
       |B     |E          +------+     PCKIN
       |      +-------+---+  C5  +---------
       |              |   +------+
       |            +-+-+
       |            | R |
       |            | 3 |
       |            |   |
       |            +-+-+
       |   +------+   |                 GND
       +---+  C4  +---+--------------+-----
       |   +------+   |              |
     +-+-+          +-+-+          +-+-+
     | R |          | C |          | C |
     | 2 |          | 2 |          | 1 |
     |   |          |   |          |   |
     +-+-+          +-+-+          +-+-+
DAOUT  |              |   +------+   |  PFIL
-------+              +---+  R1  +---+-----
                          +------+
```

I have built my circuit on a 3x5 cut of a proto board. See [here](https://twitter.com/kotzebuedog/status/1636028046921564162/photo/1) a picture that demonstrates also why you need a good soldering iron. Don't pay attention to the wires, it was a bad idea because this kind of wire is not flexible enought and also to big to have a fast heat soldering on very small connection points. In the end I opted for enameled wire that I salvaged form old earphones.

## BOM
This is the list of components I needed for my board. I can't garanty it works for other boards.

|Component|Reference|
|-|-|
|Q1|[2SC2712](https://toshiba.semicon-storage.com/info/2SC2712_datasheet_en_20220203.pdf?did=19227&prodName=2SC2712)|
|R1|1 kΩ|
|R2|1.5 kΩ|
|R3|1 kΩ|
|C1|1000 pF|
|C2|0.1 µF|
|C4|5 pF|
|C5|1000 pF|

## Patch points

This is a zoom-in on the board schematic near the TC358743 chipset on my board model. You'll need to wire the 2.5V, DAOUT, PFIL, PCKIN and GND lines.

You'll need to unsolder and remove the two capacitors near PFIL and PCKIN points. You can see [here](https://twitter.com/kotzebuedog/status/1636046110371225600/photo/1) a picture with the two capacitors removed.

```
  +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+                         +---+
  +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+                         +---+
  |  | |  | |  | |  | |  | |  | |  | |  |                         |   |
  +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+                         +---+
  +--+ +--+ +--+ +--+ +--+ +--+ +--+ +--+                         +---+

                                     +--+   +--+        ++---++     O
                                     +--+   +--+        ||   ||     3.3V
                                     |  |   |  |        ++---++
                                     +--+   +--+
                                     +--+   +--+        ++---++
                                                        ||   ||
+----------------------------------------+              ++---++
|                                      O |
|                                        |              ++---++
|                                        |              ||   ||   +---+
|                                        |              ++---++   +---+
|                                        |                        |   |
|                                        |              ++---++   +---+
|                                        |              ||   ||   +---+
|                                        |              ++---++
|                                        |     ++--++               O
|                                        |     ||  ||               1.2V
|                                        |     ++--++
|                                        |                       O
|                TC358743                |     ++--++            DAOUT
|                                        |     ||  ||
|                                        |     ++--++
|                                        |
|                                        |     ++  ++
|                                        |PFIL ||  ||
|                                        |     ++  ++
|                                        |                        +---+
|                                        |     ++  ++             +---+
|                                        |PCKIN||  ||             |   |
|                                        |     ++  ++             +---+
+----------------------------------------+                        +---+
                                           O                   O
                                                               2.5V
```

GND can be wired on the 2x6 connection port of the board where the I2S but has been connected (OSCK, SCK, WFS and SD lines)

          +-----+
     WFS  | o o |  OSCK
     SCK  | o o |  SD
     GND  | o o |
          | o o |
          | o o |
     GND  | o o |
          +-----+

[Here](https://twitter.com/kotzebuedog/status/1636132644369866752/photo/1) and [here](https://twitter.com/kotzebuedog/status/1636132644369866752/photo/2) you can see pictures of the hardware patch installed. It's glued on the back of the board.

## Schematic of my setup

My setup looks like this. I use a Raspberry Pi zero 2 connected to my streaming station throught USB with the USB Network Interface enabled. It can also work with any CSI port able Raspberry Pi, Even the Raspberry Pi zero works when I started to deal with it at the beginning but it can be painful and long to debug the setup (boot, reboot, rereboot, ...) with this SOC. So I choosed Raspberry Pi zero 2, also because I had the chance to have one (this really is a beast in such small board)

```
                                        +-------------+
                                        |    HDMI     |
                                   +--->|             |
+--------+       +----------+ HDMI |    |   screen    |
|  HDMI  | HDMI  |   HDMI   +------+    +-------------+
|        +------>|          | HDMI
| source |       | splitter +------+    +-------------+  CSI   +--------------+
+--------+       +----------+      |    | HDMI to CSI +------->|              |
                                   +--->|             |  I2S   | Raspberry Pi |
                                        |   adaptor   +------->|              |
                                        +-------------+        +-------+------+
                                                                       |
                                                                       | USB
                                                                       v
                                                               +--------------+
                                                               |  Streaming   |
                                                               |              |
                                                               |   station    |
                                                               +--------------+
```

## Preconfiguration of the board before HDMI capture

Before connecting a HDMI source few steps are needed. I've learned all of the stuff here in the Raspberry Pi forum [post](https://forums.raspberrypi.com/viewtopic.php?t=281972) from *6by9*.

### Loading EDID data

First you have to load the EDID data with `v4l2-ctl` tool

     pi@hdmipi2:~ $ v4l2-ctl --set-edid=file=1080P60EDID.txt

     CTA-861 Header
     IT Formats Underscanned: yes
     Audio:                   yes
     YCbCr 4:4:4:             no
     YCbCr 4:2:2:             no

     HDMI Vendor-Specific Data Block
     Physical Address:        3.0.0.0
     YCbCr 4:4:4 Deep Color:  no
     30-bit:                  no
     36-bit:                  no
     48-bit:                  no

     CTA-861 Video Capability Descriptor
     RGB Quantization Range:  yes
     YCC Quantization Range:  no
     PT:                      Supports both over- and underscan
     IT:                      Supports both over- and underscan
     CE:                      Supports both over- and underscan

### Changing Pixel format

Then you'll switch the mode UYVY as the default mode BGR3 limit the maximum capture resolution. It may works with BGR3 for 720p resolution but I haven't deep tested what fps will be OK.

     pi@hdmipi2:~ $ v4l2-ctl -v pixelformat=UYVY

### Controlling screen resolution

Now you can connect a HDMI source and check the screen resolution

     pi@hdmipi2:~ $ v4l2-ctl --query-dv-timings
          Active width: 1280
          Active height: 720
          Total width: 1650
          Total height: 750
          Frame format: progressive
          Polarities: -vsync -hsync
          Pixelclock: 74250000 Hz (60.00 frames per second)
          Horizontal frontporch: 0
          Horizontal sync: 370
          Horizontal backporch: 0
          Vertical frontporch: 0
          Vertical sync: 30
          Vertical backporch: 0
          Standards:
          Flags:

### Applying the screen timing to the capture setup

This is needed to make the capture process what it has to deal with. It's not automaticaly done by the board.

     pi@hdmipi2:~ $ v4l2-ctl --set-dv-bt-timings query
     BT timings set

## Basic record test for HDMI video

Once you've done the preconfiguration steps you can try to record to a file the HDMI source. Then you can retrive this file to your computer (SCP is your friend) and check it.

     gst-launch-1.0 v4l2src ! "video/x-raw,framerate=60/1,format=UYVY" ! v4l2h264enc extra-controls="controls,h264_profile=4,h264_level=10,video_bitrate=256000;" ! "video/x-h264,profile=high,level=(string)4.1" ! h264parse ! queue ! matroskamux ! filesink location=video.mkv

If it's not working try different HDMI source devices but also try to force 720p60 on the device if you can. I've seen some devices (USB-C to HDMI cable) that doesn't work at all or doesn't negociate nicely with EDID. From what I have seen, it works with a Nintendo Switch and a Nintendo Wii U as soon as I forced the TV mode to 720p60.

## Basic record test for HDMI audio

After testing the video you can now test an audio record and check what it does sound.

     gst-launch-1.0 alsasrc device=hw:tc358743 ! audio/x-raw,rate=48000,channels=2 ! audioconvert ! avenc_aac bitrate=48000 ! aacparse ! queue ! matroskamux name=mux ! filesink location=audio.mkv

After the hardware patching has been done my first audio recording test were deceiving, audio was scrambling noting really clean to hear. Hopefuly this came from my bad USB-C to HDMI adaptor, as soon as I jumped to the Nintendo Wii U the sound was cristal clear.

## Streaming with RTP

When the basic video and audio record are working you can enjoy network streaming. Video+Audio recording test is up to you but I skipped.

First I started to understood how Gstreamer works for RTP streaming with SDP file on VLC. Basically the command is

     gst-launch-1.0 v4l2src device=/dev/video0 io-mode=4 ! "video/x-raw,framerate=60/1,format=UYVY" ! v4l2h264enc output-io-mode=5 capture-io-mode=4 extra-controls="controls,h264_profile=4,h264_level=10,video_b_frames=0,video_bitrate=8192000 ;" ! "video/x-h264,profile=high,level=(string)4.1" ! rtph264pay config-interval=1 pt=96 ! udpsink host=192.168.0.2 port=5000 alsasrc device=hw:tc358743 ! audio/x-raw,rate=48000,channels=2 ! audioconvert ! audio/x-raw,channels=2,depth=16,width=16,rate=48000 ! rtpL16pay pt=96 ! application/x-rtp, pt=96, encoding-name=L16, payload=96, clock-rate=48000, channels=2 ! udpsink host=192.168.0.2 port=5002

It launches two RTP stream to `192.168.0.2` on UDP port `5000` and `5002`. Be aware that the two ports can't be 5000 and 5001, a spacing of two is needed. The `video_bitrate` works up to *25000000*. `video_b_frames=0` should reduce latency but I'm not 100% sure.

The stream can be played in VLC by opening `stream-AV.sdp` 

     v=0
     c=IN IP4 192.168.0.2
     m=video 5000 RTP/AVP 96
     a=rtpmap:96 H264/90000
     m=audio 5002 RTP/AVP 97
     a=rtpmap:97 L16/48000/2

I got kind of 1000 ms delay, which makes the games impossible to play but should be OK for streaming while playing on the real screen.

You can get details of the supported mode for H.264 encoder:

     pi@hdmipi2:~ $ v4l2-ctl -L -d 11

     Codec Controls

               video_bitrate_mode 0x009909ce (menu)   : min=0 max=1 default=0 value=0 flags=update
                         0: Variable Bitrate
                         1: Constant Bitrate
                    video_bitrate 0x009909cf (int)    : min=25000 max=25000000 step=25000 default=10000000 value=10000000
               sequence_header_mode 0x009909d8 (menu)   : min=0 max=1 default=1 value=1
                         0: Separate Buffer
                         1: Joined With 1st Frame
          repeat_sequence_header 0x009909e2 (bool)   : default=0 value=0
                    force_key_frame 0x009909e5 (button) : flags=write-only, execute-on-write
               h264_minimum_qp_value 0x00990a61 (int)    : min=0 max=51 step=1 default=20 value=20
               h264_maximum_qp_value 0x00990a62 (int)    : min=0 max=51 step=1 default=51 value=51
               h264_i_frame_period 0x00990a66 (int)    : min=0 max=2147483647 step=1 default=60 value=60
                         h264_level 0x00990a67 (menu)   : min=0 max=15 default=11 value=11
                         0: 1
                         1: 1b
                         2: 1.1
                         3: 1.2
                         4: 1.3
                         5: 2
                         6: 2.1
                         7: 2.2
                         8: 3
                         9: 3.1
                         10: 3.2
                         11: 4
                         12: 4.1
                         13: 4.2
                         14: 5
                         15: 5.1
                    h264_profile 0x00990a6b (menu)   : min=0 max=4 default=4 value=4
                         0: Baseline
                         1: Constrained Baseline
                         2: Main
                         4: High

## Streaming with MPEGTS

I was not happy to use VLC in OBS studio to receive the stream so I dig a little bit and got it worked with a native Media Source and MPEGTS encapsulation over UDP (instead of RTP).

This is done with this command

     gst-launch-1.0 v4l2src device=/dev/video0 io-mode=4 ! "video/x-raw,framerate=60/1,format=UYVY" ! v4l2h264enc output-io-mode=5 capture-io-mode=4 extra-controls="controls,h264_profile=4,h264_level=10,video_b_frames=0,video_bitrate=25000000 ;" ! "video/x-h264,profile=high,level=(string)4.1" ! h264parse ! queue ! mpegtsmux name=mux ! rtpmp2tpay config-interval=1 ! udpsink host=192.168.0.2 port=5000 alsasrc device=hw:tc358743 ! audio/x-raw,rate=48000,channels=2 ! queue ! audioconvert ! avenc_aac bitrate=48000 ! aacparse ! mux.

In OBS studio you just add a Media Source, uncheck *Local File* and fill the *Input* field with `udp://192.168.0.2:5000`. You can set network buffering to zero. With this setup I think I just got a little bit than 1000 ms delay but I haven't checked deeply.

I've also tested MPEGTS without audio and I was able to get something like 100 ms delay with this command:

     gst-launch-1.0 v4l2src device=/dev/video0 io-mode=4 ! "video/x-raw,framerate=60/1,format=UYVY" ! v4l2h264enc output-io-mode=4 capture-io-mode=4 extra-controls="controls,h264_profile=4,h264_level=10,video_b_frames=0,video_bitrate=25000000 ;" ! "video/x-h264,profile=high,level=(string)4.1" ! h264parse ! mpegtsmux ! udpsink host=192.168.0.2 port=5000 

I'm still digging to find why I can't get this 100 ms delay when I add the sound in the streaming. If you have any clue please let me know.

## Streaming with MPEGTS with low latency

Up to now the only solution I have found is to use two separate streams for video and audio.

The video stream

     gst-launch-1.0 v4l2src device=/dev/video0 io-mode=4 do-timestamp=true ! "video/x-raw,framerate=60/1,format=UYVY" ! v4l2h264enc output-io-mode=4 capture-io-mode=4 extra-controls="controls,h264_profile=4,h264_level=10,video_b_frames=0,video_bitrate=25000000 ;" ! "video/x-h264,profile=high,level=(string)4.1" ! h264parse ! mpegtsmux ! udpsink host=192.168.0.2 port=5000

The audio stream

     gst-launch-1.0 alsasrc device=hw:tc358743 ! audio/x-raw,rate=48000,channels=2 ! audioconvert ! avenc_aac bitrate=48000 ! aacparse !  mpegtsmux ! udpsink host=192.168.0.2 port=5001

Two media sources must be configured in OBS, video is `udp://192.168.0.2:5000` and audio is `udp://192.168.0.2:5001`.