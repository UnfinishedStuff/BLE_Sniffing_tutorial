# BLE_Sniffing_tutorial

This is a very quick, rough reminder guide for using [Adafruit's Bluefruit LE Sniffer](https://www.adafruit.com/product/2269) to sniff BLE traffic.  This isn't meant to be an expert guide (because I'm very much not an expert), it's a reminder for when I inevitably forget the stuff I've worked out in about a week's time.  This was worked out using the Adafruit guides which I'll link throughout this repo, as well as [Stuart Patterson's Sniffing, Reverse Engineering, and Coding the ESP32 Bluetooth LE series, found via Hackaday](https://hackaday.com/2021/03/23/a-crash-course-on-sniffing-bluetooth-low-energy/).

This was done using a few bits of hardware and software:
* The aforementioned [Adafruit Bluefruit LE Sniffer](https://www.adafruit.com/product/2269) is used to sniff the traffic
* [Wireshark](https://www.wireshark.org/download.html) is used to process the packets which the BLE sniffer captures
* An [Adafruit Feather nrf52840 Express](https://www.adafruit.com/product/4062) is used to run a BLE peripheral where the colour of the on-board Neopixel can be set.
* The [Adafruit Bluefruit Connect app](https://learn.adafruit.com/bluefruit-le-connect) is needed on a smartphone/tablet to interact with the nrf52840 Express in the way which Adafruit intended, so that you can sniff the commands to see how the system works during normal use.
* The [Nordic nRF Connect app](https://www.nordicsemi.com/Products/Development-tools/nrf-connect-for-mobile) is then used to spoof these commands to set the nRF52840 Express Neopixel by imitating the commands of the Bluefruit Connect app.


## Setup

### Wireshark

Follow the [Adafruit guide for setting up Wireshark](https://learn.adafruit.com/introducing-the-adafruit-bluefruit-le-sniffer/using-with-sniffer-v2-and-python3).  When you plug the BLE Sniffer into a USB port and restart the software, it should show the LE sniffer in the list of interfaces at start.  One very useful thing which the Adafruit guide doesn't tell you to do is to go to View > Interface Toolbars, and enable the nRF Sniffer for Bluetooth LE, which add a very helpful toolbar at the top of the software (thanks to Stuart Patterson for that, it's extremely helpful and I don't know why Adafruit's guide doesn't suggest it).

### The nRF52840 Express

[Install CircuitPython on the board as per this guide from Adafruit](https://learn.adafruit.com/introducing-the-adafruit-nrf52840-feather/circuitpython).  I then copied the [Neopixel Color example CircuitPython script](https://learn.adafruit.com/circuitpython-nrf52840/neopixel-color) into the board's `code.py` file.  I modified line 20 to read `pixels = neopixel.NeoPixel(board.NEOPIXEL, 1, brightness=0.1)` so that it was only trying to set 1 pixel instead of 10.  Use Adafruit's Bluefruit Connect app to test that this works as per their guide.

## Sniffing the traffic

*It seems to be essential that the Express board and your phone aren't connected before trying to sniff the traffic*.  Otherwise, the sniffer misses the initial pairing (?) and then fails to decode the subsequent packets, giving a `bad MIC` error.  So, start by hitting back on the Adafruit Bluefruit Connect app until you come to the Select Device page.  If you've used the nRF Connect app, find the device called CIRCUITPYxxxx in the Scanner tab and make sure it says `NOT BONDED` under the board's MAC address.

1. Open Wireshark.  Initially it should show a page which says "Capture ...using this filter:".  Somewhere in the list below should be "nRF Sniffer for Bluetooth LE" and a COM port.  Double-click that.  If you can't see that, make sure your Bluefruit Sniffer is plugged in, and that you've set up Wireshark properly.
2. In the main window of Wireshark a lot of traffic should start scrolling past.  This is *all* BLE traffic in your local environment.  We're just interested in the nRF Express board, so make sure that is plugged in.  Open the first page of Adafruit's Bluefruit Connect app.  It was able to recognise the nRF Express board and listed it as a device called CIRCUITPY77d0 (though I'm guessing the last 4 characters of yours may be different).  If you tap on the board name (**not** the Connect button) then it should expand to show more information, including the MAC address of the board (in xx:xx:xx:xx:xx:xx format).  In Wireshark's Interface toolbar at the top, click on the second drop-down ("All advertising devices"_ and look for and select that MAC address.  All of the traffic scrolling through Wireshark should now have that address as the Source or Destination; you're no longer getting traffic from all devices around you, just things being sent by/to the nRF Express board.  This should mostly have Info types of `ADV_IND`, `SCAN_REQ`, and `SCAN_RSP`. 
3. In the Adafruit Bluefruit Connect app, connect to the nRF Express board.  After a short burst of traffic, most of the Info types should become `Empty PDU`.  Apparently when devices are connected over BLE they must send packets at frequent intervals.  If there is no traffic to send, they send empty Protocol Data Units (PDUs).  The frequent `Empty PDU` packets are just an indication that the connection is being kept alive without any data to send/receive.
4. In the Adafruit Bluefruit Connect app, navigate to Controller > Color Picker.  Slide the bar underneath the colour wheel to the left to select White (the RGB indicator should show `255-255-255`).  
5.  You need to be quick here.  Tap on Select, and in Wireshark immediately look for a packet with a Protocol type of `ATT` and an Info type of `Sent Write Command, Handle: 0x0021 (Nordic UART Service: Nordic UART Tx)`.  This will scroll past very quickly, so you need to scroll up to it to keep it in view immediately.  This is the command which your phone/tablet just sent to the nRF Express board, which should now have a white light coming out of the Neopixel.  Double-click on the packet in Wireshark and it should open it in a new window.  There are  a few things you can poke at here, but the important thing is in the Bluetooth Attribute Protocol > Handle in the top window.  This is the packet which was sent.  It should show the Service UUID it was sent to (the BLE Attribute), and UUID (the Attribute's Characteristic) - make a note of these. Finally, it shows the data which was sent: `UART Tx: !C����` (don't worry about the last four characters being odd).  If you click on that text in the upper window, then the binary bytes in the lower window should highlight to show you where they are.  In this case, starting near the end of row `0010`, the values `21 43 ff ff ff 9e` should be highlighted.
6. If you repeat this to send full red (255-0-0), green (0-255-0), and blue (0-0-255) colours, you should get packet values of `21 43 ff 00 00 9c`, `21 43 00 ff 00 9c`, and `21 43 00 00 ff 9c`.  
7. If you know your LED colours you might be spotting a pattern here.  the messages always start with `21 43`, then there are three values which correspond to the red, green, and blue values in an 8-bit byte, and then there is always `9c`.  For anything else we'd need to do a bit of guesswork here, and if you chose any other colour to send then the final byte wouldn't be `9c`, but otherwise the format would hold.  It turns out, if you check the documentation for the Color Picker section of the [Adafruit Bluefruit Connect app](https://learn.adafruit.com/bluefruit-le-connect/controller), that they explain exactly what the format is.  The hexadecimal values `21` and `43`, when compared to an ASCII table, turn out to mean `!` and `C`, respectively.  This is the code to tell the RF Express board that it is about to receive a Neopixel colour value.  The next three bytes encode the colour.  Finally, the last value is a checksum, which the board can use to tell if the values were received properly, or if they may have been corrupted.  This is calculated by adding the R, G, and B values together, taking only the last 8 bits of that value, and inverting them.  


<details>
<summary>A quick Python script to work this out is shown in the collapsible section below.</summary>

```python

# The list of bytes to send, without the checksum
bytesToSend = [0x21, 0x43, 0xff, 0x00, 0x00]

checksum = 0

# Sum the bytes
for value in bytesToSend:
    checksum += value

# We only need the last 8 bits of this value
checksum = checksum & 0xff

# Then invert the bits
checksum = 0xff - checksum

# Add the checksum to the bytes to send
bytesToSend.append(checksum)

# Print the bytes in hexadecimal format
for value in bytesToSend :
    print(hex(value), end=' ')

```

</details>

8. So, now that we know which bytes need to be sent to control the colour of the LED, we can try to spoof these using the nRF Connect app.  This allows you to send BLE signals manually to check that you've understood the protocol properly.  In the Adafruit Bluefruit Connect app, hit the back arrow until you get to the Select Device screen.  This is important, as BLE devices can only pair with one device at a time, so you'll not be able to use the nRF Connect app to send the nRF Express board data until you've disconnected the Adafruit Bluefruit Connect app.
9. Open the nRF Connect app, and search for the CIRCUITPYxxxx device.  Tap the Connect button beside it.  Once connected, it should list Generic Access and Generic Attribute data, as well as some services.  First, tap on the Service with the Service UUID which we found earlier.  This should expose the Characteristics of that service, including one with the UUID which we found earlier.  Beside that characteristic UUID there should be an upward pointing arrow.  This is what we use to send data to that Characteristic, which is the Neopixel colour.  Before we send the data, we need to disable something which seems to mess the process up.  In the top right, click the three-button menu and disable "Parse known characteristics".  For whatever reason this seems to screw up the way that the data is sent.  Now click the upward arrow beside the Characteristic UUID, and in the box beside `0x` type your bytes.  This should be a single string, with no spaces, where each value is represented by two characters.  E.g., if the Python script earlier produced the values `0x21 0x43 0xff 0xff 0x0 0x9d` you would need to type `2143ffff009d`.  Tap the Advanced option and make sure `Request` is ticked, and then tap on Send.  The Neopixel should change colour, and you should see the packet transmission in Wireshark.

This process will probably be much more difficult to figure out if there isn't a documentation page showing what the bytes are, but this should maybe be a starting point for working out the protocol for other devices.
