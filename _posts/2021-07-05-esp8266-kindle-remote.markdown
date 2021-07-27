---
layout: post
title:  "Controlling Kindle Remotely with ESP8266"
date:   2021-07-05 18:30:00 +0800
category: projects
image: /assets/2021/07/espkindleremote/cover.jpg
tags: esp8266 arduino kindle
---
![The front of the remote](/assets/2021/07/espkindleremote/front.jpg)

![The back of the remote](/assets/2021/07/espkindleremote/back.jpg)

<div style="position:relative;padding-bottom:56.25%;">
 <iframe style="width:100%;height:100%;position:absolute;left:0px;top:0px;" src="https://www.youtube.com/embed/gm3em-5v1Pw" frameborder="0" allow="autoplay">
 </iframe>
</div>  
<br/>

I got myself a Kindle Paperwhite 3 to read light novels. After using it for a while, I found that holding on to the Kindle during my reading sessions (usually a couple of hours) to be tiring on my wrist. To solve this issue, I got myself a tablet mount to hold the Kindle on my desk. Now with the Kindle mounted, I find it troublesome to raise my hand to change pages especially when I am trying to read on my bed. With that, I went online looking for a way to change pages without physically tapping on the touchscreen.

After some searching, I found an [Instructables post][kindle-footswitch] detailing a WiFi connected foot switch for remotely changing pages on the Kindle. With that as my starting point, I started working on my own Kindle remote.

## How it works
This remote works by triggering a CGI script running on the Kindle to simulate a finger tap. The Kindle needs to be jailbroken so that the CGI script can be installed. A ESP8266 microcontroller is used to trigger the CGI scripts over WiFi.

### Jailbreaking the Kindle
There is not much to say here as there are many guides out there detailing how to jailbreak the Kindle. I followed the one [here][jailbreak-guide].

### Installing the CGI scripts on the Kindle
After the Kindle is jailbroken, I installed the USBNet tool which comes with busybox that has the httpd applet. After that I copied the required files into the Kindle's internal memory over SSH.

The CGI scripts works in the same way as the [Instructables post][kindle-footswitch] so I will just briefly explain how it works. By reading the output of the touchscreen into a file when I am tapping at the screen lets me "save" the tap into a file. By replaying this data into the touchscreen event file, the kindle will think that I have physically tapped the touchscreen and change the page.

#### Saving the tap to file
- Connect to the Kindle over SSH.
- Run `cat /dev/input/eventX > next.event` (the event number depends on your Kindle, for mine it was `event1`)
- Tap the screen at a position that will make the Kindle go to the next page.
- The command should complete with the `next.event` file in the current directory (You may need to Ctrl+C after tapping the screen if it does not return to the prompt itself).
- Repeat with the tap to go to the previous page.

#### Replaying the tap to the Kindle.
- This can be done either via SSH or with the CGI script
- Run `cat next.event > /dev/input/eventX`
- The page should have changed as if you physically tapped on the screen.

### Running the httpd applet on startup
The `starthttpd.sh` script needs to be run on every startup so that we can access the CGI scripts over the network. This is achieved by adding a configuration file in `/etc/upstart` that runs the script on startup. The configuration is based on [this Mobileread forum post][run-on-startup].

## Prototyping the hardware
The hardware is simple, with 1 LED and 4 buttons connected to the ESP8266. I started by following the [footswitch Instructables][kindle-footswitch] by building the same circuit on a breadboard. With a working circuit and code running, I was able to then add my own changes to the project.

![Breadboard prototype](/assets/2021/07/espkindleremote/breadboard.jpg)

## Building the Remote
![The internals](/assets/2021/07/espkindleremote/internal2.jpg)

![Schematic diagram](/assets/2021/07/espkindleremote/schematic.png)

The remote I used was a 4 button remote for an aftermarket car security system. Building this was quite a challenge as I had very little space to squeeze all the parts in. I wanted to use the remote's original tactile buttons so I removed all the parts from the remote's circuit board and then wired the original indicator LED and buttons to the ESP8266. The connections are made with magnet wire so that I can fit all the wires inside the remote. I used a small LiPo battery from my parts bin and as for the charger, I cut a TP4056 charger board so it fits inside the remote. As for the microUSB input, I took a USB breakout board and cut it so it fits upright (facing the back of the remote) and wired it to the charger.

For power, I added a switch which connects the battery to a 3.3v buck/boost converter. The power switch had to be mounted on the outside of the remote unfortunately as I couldn't find any more space inside the remote.

After wiring all the parts, programming and testing to make sure it works, I squeezed everything together and screwed the back cover shut. The ESP8266's shielding case was too tall so I ended cutting a large hole in the back cover to let it through. It should also help with the heat dissipation.

![The internals before closing the case](/assets/2021/07/espkindleremote/internal1.jpg)

## Programming the ESP8266
### The code
With the hardware working, its time for the code! The remote starts up, connects to the WiFi network and then waits for a button press. When a certain button is pressed, it sends a GET request to the respective URL defined in the code. This allows the remote to control other devices as well.

The original code from the [footswitch Instructables][kindle-footswitch] set the ESP8266 to host a WiFi access point. This can be good for using anywhere but as I am only using it at home (and would like to have network access for adding files remotely, installing OTA updates), I modified mine to connect to my home network. I have also assigned a static IP (DHCP reserved IP) to the Kindle.

I have also added OTA support for the ESP8266. This allows me to quickly change the code without needing to connect the device to my computer every time. The ESP8266 checks for a combination of button presses during startup, after it has connected to WiFi. If the combination is detected, it switches to OTA mode and waits for a new firmware to be uploaded. Otherwise, it will proceed with normal operation.

As the ESP module comes without the code, I had to program the module the first time over UART. I wired a USB to serial adapter to the device with small clips and uploaded the program with the Arduino IDE. After that, any subsequent programming was done over the air.

You can find the code and files used here: [ESPKindleTouch][repo]

## Issues/ Discussions
### Was this practical?
No. The battery I used was too small and the ESP8266 draws so much power that it probably only lasts for up to an hour from a full charge. I ended up using the remote with it plugged in all the time. The battery life was useless, but at least it was nicer to hold compared to my breadboard version. This remote now finds its use when I am reading on the bed. I plug the remote to a power bank to keep it charged. When reading it on the desk, I continue to use my breadboard prototype.

### The Kindle sometimes takes a few seconds to switch pages
I think this is due to using my home network to communicate, the best solution for lowest button-press-to-page-change latency will be having the ESP8266 host its own access point and have the Kindle connect to it. My main reason for not doing that was so that I can still connect to the Kindle from my PC.

The delay between button press and page change is usually less than a second but when it is slow, it can take up to 2 or 3 seconds before the page changes. This can be annoying sometimes but I can live with it for now so I will continue to use it as is.

### Security
This project was built with no security in mind. I ran the project within my home network so I am relying on the security of my home network. If the device is connected to a unsecured network, the Kindle can be easily controlled by any HTTP client and if OTA was enabled on the ESP8266, anyone on the network can upload code to it.

The ESP8266 can be modified to host its on access point which the Kindle can connect to to allow for use outdoors or outside of home network. I only read on my Kindle at home so I did not add this feature into the project.

### Sleep Timer Reset
During testing, I have found that after approximately 10 minutes the Kindle will go to sleep regardless of whether the remote is used or not. After some investigation, the issue was narrowed down to the sleep timer not being reset after changing pages remotely. With more searching, I found a [discussion][sleep-timer-reset] showing a command which resets the sleep timer by simulating a touch event. The command was added into the CGI scripts and with that, every remote page change will reset the sleep timer and prevent it from sleeping when I am still reading.

### Possible upgrades
This method of changing pages is quite crude as it only simulates a touch, meaning that only basic tasks can be performed (chaining taps is possible but with no form of feedback to the script, if the devices lags out this can end up not working properly).

One solution is to directly trigger a page change in software (I am using [KOReader][koreader] as my reader so perhaps writing a pluginfor it, no idea if it is even possible on the default Kindle reader). With a software trigger, it would be possible to confirm that the device has already changed page, or even perform other tasks such as changing font size, zooming, skipping to front/end of page. The possibilities are endless.

I have no idea on how to write a plugin for KOReader so this will have to wait for another time.

## References
- [Jailbreaking the Kindle][jailbreak-guide]
- [Instructables post on using the ESP8266 to change pages][kindle-footswitch]
- [Guide on setting up httpd applet and controlling pages from remote device][kindle-web-remote]
- [Keeping the kindle awake when changing pages with the remote][sleep-timer-reset]
- [Running scripts on startup][run-on-startup]

[jailbreak-guide]: https://wiki.mobileread.com/wiki/5_x_Jailbreak
[kindle-tool-snapshots]: https://www.mobileread.com/forums/showthread.php?t=225030
[kindle-footswitch]: https://www.instructables.com/Kindle-WiFi-Remote-Footswitch/
[kindle-web-remote]: https://www.instructables.com/Kindle-Web-Remote-Control/
[sleep-timer-reset]: http://www.mobileread.mobi/forums/showthread.php?t=220810&page=3
[run-on-startup]: https://www.mobileread.com/forums/showthread.php?t=221019
[koreader]: https://github.com/koreader/koreader
[repo]: https://github.com/bneo99/ESPKindleTouch
