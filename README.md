# Bike Tracker
## Background
There are actually plenty of bicycle trackers around, so why make your own? Well, so that it works the way you want it… And it’s fun!

I was looking for a tracker that would let me know if the bike is moving and that will start tracking it through GPS if it’s stolen. I did not want it to work through bluetooth, because that requires proximity to the device and that enough users of the same tracker are around (never liked that concept). And I did not want to tie myself into using an app that likely will stop working if the company goes belly up. Most of the time, you’re also forced to use the SIM-card provided by the company that sells the device. Nah… I want to be in control of this thing.

So, I needed a microcontroller that I could hook an accelerometer and a GPS to, and that reliably could send me the data gathered from those sensors. A simple SMS seemed like the best way to transmit, and it’s unlikely to stop working in the near future.
Hardware
After some research I ended up with the following hardware:
* Arduino MKR GSM 1400 as the mainboard
* GSM/GPRS 2dBi internal U.FL antenna
* ADXL345 3-axis accelerometer
* Arduino MKR GPS Shield
* 3.7V 4400mAh Li-Po battery
* On-off micro switch
* Sparkfun LiPo charger plus
* JST PHR-2 connectors

A note on the charger listed above:
While testing the hardware I noticed that the MKR GSM board wouldn’t charge the Li-Po batteries I had, and after some further research I concluded that there was either something wrong with the board or that it simply wasn’t compatible with the batteries I had. Since it was functioning otherwise, and that I didn’t want to spend a lot of time investigating further I decided on using an external charging circuit instead and got the Sparkfun charger. If you make something similar and the MKR GSM board charges your battery, this external charger will of course not be necessary.

## Wiring
### MKR GMS 1400
I couldn’t find the MKR GSM 1400 without the headers, so those had to be desoldered before anything else could be connected to the board. The headers just take up too much space for them to be kept on there.

Pull the plastic blocks off the pins with a pair of pliers and then desolder each pin individually. Unless it gets really messy there’s also no need to remove any of the old solder, and it might even be that the solder that is left on the board can be used for the wires later. It might be necessary to create a new hole for the wires though, but this can be achieved by pushing a toothpick through the solder while heating it up.

Pin 10 and the reset pin have been connected so that the board can be reset programmatically when needed.

To save a little bit of battery the power-LED has been desoldered.

### MKR GPS Shield
This connects easily through the provided I2C cable and the I2C port on both the main board and the GPS module. No soldering required. The digital 7 pin is used as a GPS wakeup pin.

### ADXL345 Accelerometer
MKR GSM 1400 | ADXL345
------------ | -------
Pin 6 | INT1
Pin 8 | INT2
Pin 11 | SDA
Pin 12 | SCL
GND | GND
VCC | VCC

### Sparkfun LiPo Charger Plus
The positive wire goes through the micro switch to the positive lead on the JST PHR-2 connector that connects to the main board. The negative wire goes directly to the negative lead on the JST PHR-2 connector.

## Software
The code for this project is found on Github:
https://www.github.com/Didgeridoohan/BikeTracker

I’m not actually a programmer (and this is my first ever Arduino project), I just like to dabble. As a result, the code might not be very pretty, effective or “correct”, but it works. If you’ve got the time and skill, please feel free to contribute.

## Features
### Basics
The basic features include (a more detailed description follow):
* Movement detection.
* GPS and GPRS location.
* Control the device through SMS-commands.

### Details
#### Movement detection
If the tracker moves, you will be notified through an SMS. If things then stay quiet for a certain amount of time (default 30 seconds), another SMS letting you know that everything is quiet again is sent out.

#### Location
When things don’t quiet down, but movement is continuously registered over a period of 1.5 minutes the GPS will activate and start transmitting. If no GPS lock can be acquired within one minute the device will instead try to get the location from the mobile network, via GPRS (much less accurate, but can get an approximate location when the device is indoors or otherwise cannot get a GPS fix).
The location of the device will be transmitted every 2 minutes for a minimum of 5 times (not including the initial location ping). If there’s still movement after 5 successful location pings the device will continue to send the location 5 more times. But if no movement is registered once the 5th location message goes out the device will stop transmitting its location.

#### Deep sleep
To use the least amount of battery, the device will go into deep sleep mode if no movement is registered or if location transmission isn’t active. By default this happens after 5 minutes. In deep sleep mode only movement can wake up the device, everything else is shut off. The device will wake up at set times (by default 8:00 and 20:00 o’clock), to transmit the battery level and possibly receive SMS commands (more on that below). If nothing happens within 5 minutes the device goes back to deep sleep.
If the device has been in a location sending mode, and once the device stops sending the location data, the period until it goes back into deep sleep mode is set to 20 minutes. This is so that you can have a longer window of opportunity to send SMS-commands (see below), for example to manually trigger a location ping.

#### SMS-commands
There are a number of commands that can be sent to the device. The prerequisite is that the device is on and that it isn’t in deep sleep mode.

To send a command, create an SMS that starts with your device password (see the setup section below), directly followed by the command. The password can contain any characters or numbers you’d like.

**Example**: thisismypasswordgpsloc

In this example, "*thisismypassword*" is the used password, and this command would trigger a single location ping.

**Start-up**  
*Command: startup*  
Use this command to start the device up again after having put it in idle mode with the Idle command.

**Idle**  
*Command: idle*  
Put the device in an idle mode where the device is still on but nothing runs except the GSM module. Does not save as much battery as the automatic deep sleep mode, but it can instead still receive SMS-commands. The device will automatically exit idle mode after a set time (default 2 hours).

**Restart**  
*Command: restart*  
Reset the device, as if you had toggled  the power button off and on.

**GPS Location**  
*Command: gpsloc*  
Trigger a single location ping.

**GPS Start**  
*Command: gps*  
Start a location transmitting cycle.

**GPS Stop**  
*Command: gpsstop*  
Stop transmitting the device location.

**Deep sleep**  
*Command: deepsleep  
Alternative command: deepsleep12*  
Manually enter deep sleep mode. If the command is followed by a number between 0 and 23 (24-hour clock, 0 is midnight), the wake-up event will occur the next time that hour passes. Once it has, or the device wakes up from movement, the default wake-up times will be used.

**Set Sleep Timer**  
*Command setsleep5*  
Set the time in minutes until the device should enter deep sleep. The number after “setsleep” is the number of minutes you want to set the timer to.

**Battery Check**  
*Command: battery*  
Check the device’s battery.

**Uptime**  
*Command: uptime*  
Check how long the device has been active without a reset.

## Setup
When preparing the software for upload to the board, the arduino_secrets.h file needs to be set up with the following variables (download the file from GitHub for ease of setup, it needs to be in the same project directory as the main .ino file):  
SECRET_PINNUMBER "*SIM-card pin number*"  
SECRET_PHONENUMBER "*phone number you want to send SMS-messages to*"  
SECRET_PASSWORD "*SMS commands password*"  
SECRET_GPRS_APN "*your mobile provider’s APN address*"  
SECRET_GPRS_LOGIN "*the APN login*"  
SECRET_GPRS_PASSWORD "*the APN password*"  

## Mounting on the bike
I’ve had several different ideas on how to mount the tracker on a bike. Commercial trackers can often be hidden in the handlebars or similar, but the MKR GSM 1400 module can be a bit too wide for that with its 25 millimeters. Other options could be in the seat-post hole or in a case mounted to the frame somewhere.

### Inside the frame
When hiding the tracker in the frame it’ll be necessary to construct it in a tube-like fashion. All the components need to be laid out in a line and the battery needs to be single-cell or two cells laid out one after the other rather than next to each other.

If the handlebars are wide enough to fit the tracker that’s great. It’ll be easy to access and can be well hidden.

Other options for hiding the tracker inside the frame pretty much only includes under the seat post. But, the tracker will be difficult to get too if it needs to be turned off, charged, updated, etc. Here I have an idea that involves drawing cables from the tracker up through holes drilled in the seat post and then have the power button and charging/data port mounted under the seat. I have not tried this option (yet), and it will take a bit of work, but once completed I think it could be a pretty nice setup with all the parts that need to be accessible hidden under the seat. With this option the only time you would need access to the tracker would be if it needs repairs. Protecting it with some kind of padding would probably be necessary.

One thing to keep in mind when planning to put the tracker inside the frame is the possibly detrimental effect it will have on reception, both for the GSM module and the GPS shield. Putting the antenna inside a metal tube isn’t the best option if you want a strong signal...

### On the frame
If hiding the tracker inside the frame isn’t an option, putting it on the frame somewhere is the only way. Hiding it under the seat might be possible, in some kind of box you bolt/strap to the frame, etc.

I opted for a solution where I merged the box with the mounting holder for my bicycle lock. That way the box kind of looks like it is part of the mount, but it is still easily accessible. For my prototype I used a multipurpose plastic enclosure from Hammond Electronics (1591HBK) that I (heavily) modified to accommodate the lock mount and to also make it somewhat waterproof. It took a lot of cutting, grinding, gluing and swearing, but eventually I got it to work somewhat like I want it to. If I had access to a 3D-printer I would have designed a more streamlined case that doesn’t stick out like a sore thumb, but that’s for another time.

Another possible way to mount everything “on the frame” would be to mount every component separately underneath the saddle and then weatherproof with heat shrink and other plastic wrappings. Easily accessible, at least.

## Licence
MIT License

Copyright (c) 2021 Didgeridoohan

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
