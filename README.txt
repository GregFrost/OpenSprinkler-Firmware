============================================
==== OpenSprinkler AVR/RPI/BBB Firmware ====
============================================

This is a unified OpenSprinkler firmware for Arduino, and Linux-based OpenSprinklers such as OpenSprinkler Pi.

For OS (Arduino-based OpenSprinkler) 2.x:
https://openthings.freshdesk.com/support/solutions/articles/5000165132-how-to-compile-opensprinkler-firmware

For OSPi/OSBO or other Linux-based OpenSprinkler:
https://openthings.freshdesk.com/support/solutions/articles/5000631599-installing-and-updating-the-unified-firmware

============================================
Questions and comments:
http://www.opensprinkler.com
============================================

This version of OS has been hacked to support LC Technology 4 channel relay boards.
These are the notes from my development:

Download and installed:
https://downloads.arduino.cc/arduino-1.8.18-windows.exe

Clone the OpenSprinkler firmware from:
https://github.com/OpenSprinkler/OpenSprinkler-Firmware.git into a directory called mainArduino
Open the mainArduino.ino file in the Arduino IDE.
Update the Arduino AVR Boards package as prompted.
Use Boards Manager to change esp8266 boards version to 2.7.4 (based on the compile instructions).
Install ThingPulse SSD1306 4.2.1 library.
Install sui77 RCSwitch 2.6.4
Install knolleary pubsubclient 2.8.0 (instructions said 2.8)
Install jandrassy EthernetENC 2.0.1 (all arduino ide offered - instructions said 2.1.9??).
Compile from IDE.

First problem: 
C:\Users\Greg Frost\Desktop\Designs\NodeMCU\ESP-01 OpenSprinkler\Firmware\mainArduino\OpenSprinkler.cpp: In static member function 'static uint8_t OpenSprinkler::start_ether()':
OpenSprinkler.cpp:514:11: error: 'class EthernetClass' has no member named 'init'
  Ethernet.init(PIN_ETHER_CS); // make sure to call this before any Ethernet calls
Many other similar errords.
This looks like it is caused by the built in Ethernet.h in the Arduino libraries folder being used rather than the one from the EthernetENC package.
Updated OpenSprinkler.h to include EthernetENC.h instead of Ethernet.h
Now the compilation completes without error.

Uploading this to the ESP-01 module results in a joinable AP OS_20B1AA.
Joining and visiting 192.168.4.1 gets to the config page.
Submitting WiFi details results in a reboot, but it starts in AP mode again.
I uncommented: 
	#define ENABLE_DEBUG  // enable serial debug
And compiled the firmware again and got several errors like this:
In file included from C:\Users\Greg Frost\Documents\Arduino\libraries\EthernetENC\src/Ethernet.h:31:0,
                 from C:\Users\Greg Frost\Documents\Arduino\libraries\EthernetENC\src/EthernetENC.h:1,
                 from C:\Users\Greg Frost\Desktop\Designs\NodeMCU\ESP-01 OpenSprinkler\Firmware\mainArduino\OpenSprinkler.h:38,
                 from C:\Users\Greg Frost\Desktop\Designs\NodeMCU\ESP-01 OpenSprinkler\Firmware\mainArduino\main.cpp:26:
C:\Users\Greg Frost\Documents\Arduino\libraries\EthernetENC\src/utility/Enc28J60Network.h: In function 'void do_loop()':
C:\Users\Greg Frost\Documents\Arduino\libraries\EthernetENC\src/utility/Enc28J60Network.h:65:18: error: 'static uint8_t Enc28J60Network::readReg(uint8_t)' is private
   static uint8_t readReg(uint8_t address);
I updated the library file Enc28J60Network.h to declare that function public. Then it compiled, but after updating I had the same issue (entered Wifi details, but it restarted in AP mode).
This time, with the serial monitor connected, I could see these messages on reboot:
 ets Jan  8 2013,rst cause:2, boot mode:(3,6)

load 0x4010f000, len 3584, room 16 
tail 0
chksum 0xb0
csum 0xb0
v2843a5ac
~ld
started
2106-02-07 01:28:25 - MQTT Init
2106-02-07 01:28:25 - MQTT Init: ClientId OS-40F52020B1AA
Each time I enter the WiFi details, it reboots and gives the same messages.
Im thinking this may be the lack of buttons. Is it doing something that makes it think the AP mode reset button is being pressed? Added some more debug lines to see what is going on.
Adding some debug revealed this on the restart:
read from iopts.dat pos 0 len 1open failed
invalid FW version. Factory Reset.
So it looks like the SPIFFS is not working properly. Some googling found this:
	https://github.com/esp8266/Arduino/issues/4061
Which suggests updating Esp.cpp. Coulnt work out how to do that, so in desparation, I updated the esp board version to 3.0.2 in the Board Manager. After doing this, I git the same error. The long discussion in the issue above suggested needing to clear the EEPROM to recover from the issue, so I tried that (Tools->Erase Flash->Erase all flash Contents), but it didnt help.
Then I tried some simple SPIFFs tests which also showed the SPIFFS was not working, but I had the epiphany that the Arduino IDE needs to be told that there is only 1M of flash on the ESP01 and I had the Board set to NodeMCU 0.9. Changed this to "Generic ESP8266 Module" and set the Flash Size to 1MB (FS: 128k) and suddenly it all started working!!!!
