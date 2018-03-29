# Control Itead S20 WiFi Smart Socket with MicroPython and MQTT

This project is a simple MicroPython program for
[Itead S20 Wifi Smart Socket](https://www.itead.cc/smart-socket.html).
It replaces the default firmware.  It allows remote control by using
MQTT broker.  For example [Node-red](https://nodered.org/) with
embedded MQTT broker
[node-red-contrib-mqtt-broker](https://github.com/zuhito/node-red-contrib-mqtt-broker)
can be used as a controller to build automation flows for the device.

The program registers with MQTT broker, subscribes topic and waits for
message with `on` or `off` payload.  When message is received, the state
of relay switch is set to on or off.

If button is pressed on S20 Wifi Smart Socket, the program will publish
`on` or `off` message to the broker.


## Pre-requisites

First, the default firmware on S20 Wifi Smart Socket needs to be
replaced with MicroPython firmware.  In order to flash MicroPython
you'll need USB UART with 3.3V option. Ebay sellers have various
options. Check that yours also provide 3.3V TTL level and not just
3.3V output voltage.

Download MicroPython firmware image for ESP8266 from http://micropython.org/download#esp8266

Install esptool flasher into a Python virtualenv

    virtualenv esptool-virtualenv
    . esptool-virtualenv/bin/activate
    pip install esptool


## Locating the Pads for Programming the Microcontroller

To open S20 Wifi Smart Socket you'll find one screw under the red
sticker at the back of the device.  After unscrewing it, pry the cover
open carefully from the sides.  There is only single clip holding the
cover at the bottom, so it is easiest to start there.

When the cover is open locate the pads marked in the figure as GND,
TX, RX and 3.3V:

![Imgur](http://i.imgur.com/TpKxdGK.jpg?1)



To program the microcontroller you can either solder pin header to the
PCB or just solder wires directly.  Yet another alternative is to
build a cable with pogo pins to create temporary connection:

![Imgur](http://i.imgur.com/ckwMiwD.jpg?2)


## Flashing MicroPython Firmware

Connect USB UART to the 3.3V, GND, TX and RX pins:

![Imgur](http://i.imgur.com/QdrVQXC.jpg)


Keep reset button (the button in the middle of the PCB) pushed to
enter programming mode while connecting USB UART to the device to
enter the programming mode.  You can release the button after the
device has been powered.  Then execute:

    $ esptool.py --port /dev/ttyUSB0 erase_flash

    esptool.py v2.1
    Connecting....
    Detecting chip type... ESP8266
    Chip is ESP8266
    Uploading stub...
    Running stub...
    Stub running...
    Erasing flash (this may take a while)...
    Chip erase completed successfully in 2.4s
    Hard resetting...


Disconnect the programmer and connect it again while again pushing the
reset button to enter programming mode.  To flash micropython image run:

    $ esptool.py --port /dev/ttyUSB0 --baud 460800 write_flash --flash_size=detect 0 ~/Downloads/esp8266-20170823-v1.9.2.bin

    esptool.py v2.1
    Connecting....
    Detecting chip type... ESP8266
    Chip is ESP8266
    Uploading stub...
    Running stub...
    Stub running...
    Changing baud rate to 460800
    Changed.
    Configuring flash size...
    Auto-detected Flash size: 1MB
    Flash params set to 0x0020
    Compressed 601136 bytes to 392067...
    Wrote 601136 bytes (392067 compressed) at 0x00000000 in 8.8 seconds (effective 546.4 kbit/s)...
    Hash of data verified.

    Leaving...
    Hard resetting...


Disconnect the programmer and connect it again but do NOT push the
reset putton this time.  Use serial terminal emulator program such as
miniterm which you should have now installed in esptool-virtualenv
(part of pyserial which was installed as dependency of esptool).
After launching terminal emulator press enter to get python prompt `>>>`.

    $ miniterm.py --raw /dev/ttyUSB0 115200

    --- Miniterm on /dev/ttyUSB0  115200,8,N,1 ---
    --- Quit: Ctrl+] | Menu: Ctrl+T | Help: Ctrl+T followed by Ctrl+H ---
    #5 ets_task(4020edc0, 29, 3fff9150, 10)
    OSError: [Errno 2] ENOENT

    MicroPython v1.9.2-8-gbf8f45cf on 2017-08-23; ESP module with ESP8266
    Type "help()" for more information.
    >>>


## Configure Wifi Settings

To get S20 Wifi Smart Socket to join to your Wifi network at boot you
must connect wifi network at least once.  ESP8266 will store the connection
information into EEPROM and use it each time when rebooting to reconnect.

Enter paste mode by pressing CTRL+E in MicroPython prompt and copy &
paste following code:

    import network
    sta_if = network.WLAN(network.STA_IF)
    sta_if.active(True)
    sta_if.connect('MYSSID', 'MYPASSWORD')


Change MYSSID to your wifi network name and MYPASSWORD to your wifi password.
Print the IP address your device got from your network by running:

    sta_if.ifconfig()[0]


## Set WebREPL Password

WebREPL is a method to use the Python prompt remotely via a web
socket.  This is much more convenient way for trials and development
than using serial console, especially if you did not solder header
pins to the S20 Wifi Smart Socket to get permanent USB UART
connection.

To set WebREPL password, execute following in Python promopt while still
being connected with serial console:

    >>> import webrepl_setup
    WebREPL daemon auto-start status: disabled

    Would you like to (E)nable or (D)isable it running on boot?
    (Empty line to quit)
    > e
    To enable WebREPL, you must set password for it
    New password: NNNNNN
    Confirm password: NNNNNN
    Changes will be activated after reboot
    Would you like to reboot now? (y/n) y


This password will be prompted from clients who connect WebREPL server
via network.

To connect your device over the network you can use WebREPL client in
web browser.  You can access it here http://micropython.org/webrepl/.
It requires the IP address and port of the WebREPL server running in
S20 Wifi Smart Socket `ws://YOUR_IP:8266/` where YOUR_IP is the
address that the `if_config()` call returned.

Alternative method for the browser based WebREPL client is
[mpfshell](https://github.com/wendlers/mpfshell).  Install by running:

    pip install mpfshell


Then connect to the WebREPL server by running:

    mpfshell ws:YOUR_IP,password


## Test Switching the Relay with Python

Connect to the webrepl and write following code to the Python prompt:

    from machine import Pin
    relay = Pin(12, Pin.OUT)
    relay.on()  # turn power on
    relay.off() # turn power off


If the device is only powered by USB UART you will not hear the relay
click, but you will see the LED lit.



## Installing the Program

First modify the configuration in `config.json` with your MQTT broker
address, username, password and topic.  Next upload the program and the
config file:

    mpfshell "ws:YOUR_IP,password"
    put main.py
    put config.json


After rebooting the device, the program will establish connection to
MQTT broker.  The connection between device and broker is not secured
by TLS and the intention is to use the program with broker in trusted
network only.


## References

[1] S20 Wifi Smart Socket Schematic https://www.itead.cc/wiki/S20_Smart_Socket
