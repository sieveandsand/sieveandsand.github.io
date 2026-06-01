+++
date = '2020-10-06T00:00:00-07:00'
title = "Hacking the Exway X1 Pro"
summary = "Cracking open an electric skateboard remote to sniff its proprietary telemetry over a 2.4 GHz nRF24L01+ clone"
+++

*Cracking open an electric skateboard remote to sniff its proprietary telemetry over a 2.4 GHz nRF24L01+ clone*

---

## Intro to eSk8

About a year ago, I got myself into this new hobby called electric skateboarding. It's basically putting electric motors on the back of a longboard and having the motors do all the pushing for you. Some of you might be familiar with the Boosted board, which was once the "Apple" of eSkate and a very popular brand among YouTubers. The one I got for myself is called Exway X1 Pro, which comes at half the price while still delivering comparable performance. How is the X1 Pro? Simply said, it brings a lot of adrenaline and fun, and there is nothing like carving in the streets of Shanghai and Seattle that makes me appreciate how gorgeous these two cities are.

However, there is something missing for me as a hardware nerd. I wanted to customize the hardware and have a variety of accessories, like fancy LED braking lights and HUDs. None of these functionalities can be achieved without having access to the vehicle's telemetry data, which is proprietary. Each electric skateboard comes with, of course, a board as well as a remote. There is a throttle wheel on the remote that can be used to control the speed of the vehicle. All telemetry data (i.e. speed, battery status, and gear level) is delivered to the user through a small display on the remote. Now the question becomes: how do I get my hands on all this data?

## Cracking the Hardware

I had no idea how to approach this quest I set up for myself. However, the only sensible next step for an embedded engineer is to crack open the remote and see what is inside.



Upon opening up the casing of the remote, we are greeted with a Cortex-M3 microcontroller which is tucked under a small display. The vibration motor and push button can be seen on the PCB, and the antenna is the thin wire trailing off the board.



A closer look at the PCB reveals its microcontroller: [GD32F130](http://gd32mcu.21ic.com/data/documents/shujushouce/GD32F130xx_Datasheet_Rev3.1.pdf). This chip is manufactured by GigaDevice and shares the same LQFP48 pinout as ST's Cortex-M0+/M3 parts. It's essentially a drop-in clone. During my time playing around with this chip, it seemed to be perfectly compatible with STM's software and the ST-Link debugger.



The back of the remote PCB reveals two more ICs. The one on the right is the RFX2401C RF front-end module, which serves as a PA/LNA. The one on the left is the Si24R1, a 2.4 GHz wireless transceiver chip. This transceiver is where all the data is packaged and sent over the air, and the most interesting thing about it is that it's a clone of the popular nRF24L01+ chip made by Nordic. There are a lot of resources out there about the nRF chip, and it is a blessing to work with something that is so well documented.

To sum this up, the Si24R1 chip handles all the traffic and could be our entry point to the telemetry data, if we find a way to listen in on the communication between the remote and the board.

## Promiscuous Sniffing on nRF24L01+

Given the fact that the Si24R1 is a perfect clone of the nRF24L01+, and the nRF24L01+ is well documented, I decided to look around and see if there were any available methods of promiscuously sniffing nRF24L01+ packets. It turns out the nRF24L01+ has been successfully sniffed since 2011. Travis Goodspeed published a [blog post](http://travisgoodspeed.blogspot.com/2011/02/promiscuity-is-nrf24l01s-duty.html) on the technical implementation of this promiscuous sniffing procedure. My sniffing program for this project is based on his research, and I recommend anyone who is interested in this project to take a look at his write-up.

Travis' implementation of promiscuous sniffing can be boiled down to 4 parts:

1. Limit the MAC address to 2 bytes
2. Disable checksums
3. Set the MAC to be the same as the preamble
4. Sort received noise for a valid MAC address

### Hardware Setup

My project setup requires an nRF24L01+ chip and an Arduino. Both are readily available, and there are a lot of resources out there on how to wire them up. The nRF24L01+ breakout board is connected to the Arduino over SPI, and the CE and CSN pins on the breakout board can be connected to any digital pins on the Arduino.

Update: the Arduino and nRF24L01+ setup does come with a compromise. The Arduino UNO only has 2 KB of SRAM, which is not enough to buffer the data stream. Each packet is 32 bytes, and at a 2 Mbps air data rate the RAM fills up almost immediately.

## Finding the Right Frequency

Knowing only the fact that the Si24R1 is a 2.4 GHz ISM chip is not enough for sniffing. We need more specific information on the number of channels, what frequencies these channels are operating on, and the bit rate the data is transferred on. Luckily, all of this information can be found in the FCC database. X1 Pro's FCC ID is [2APTF-X1](https://fccid.io/2APTF-X1), and its test report submitted to the FCC contains the information we wanted.

From the test report, we can see that the X1 Pro operates between 2402 and 2480 MHz and has a total of 40 different channels. Each channel is separated by a 2 MHz frequency offset.

Knowing that each channel is separated by a 2 MHz offset is crucial in determining the air data rate the chip is operating on. In the below snippet of the nRF24L01+ datasheet (the Si24R1 is perfectly compatible with the nRF24L01+), it specifies that 2 MHz channel spacing translates to a 2 Mbps air data rate.



## Packet Protocol

Upon finding the right address for the remote and successfully sniffing it, I got the chance to take a look at the packets. The electric skateboard we are working with has very customizable controls. Users are able to toggle features like cruise control, free mode (a.k.a. bidirectional mode, where the board goes in reverse after braking to a stop), turbo mode (unlocks a higher top speed), and a lot more. These parameters are all encoded in the payload.

```
CE5280050180001C9A198AA6C39818000000002787403C8FD65B3B0D4D6DEBE9
```

Above is an example of the payload portion of a packet sent by the remote. Surprisingly, the payload is not encrypted in any way, and as long as you have the right address, you will be able to read it. There is still a lot of information in the payload that I don't know how to interpret, but here is what I have figured out so far.

### Gear Level



`0x18` represents the current gear level. If the board is put in free mode, the hex encoding changes depending on the direction of the throttle.


| HEX  | Gear Level (traveling forward) |
| ---- | ------------------------------ |
| 0x00 | Level 1                        |
| 0x08 | Level 2                        |
| 0x10 | Level 3                        |
| 0x18 | Level 4                        |



| HEX  | Gear Level (traveling backward) |
| ---- | ------------------------------- |
| 0x40 | Level 1                         |
| 0x48 | Level 2                         |
| 0x50 | Level 3                         |
| 0x58 | Level 4                         |


However, if the board is not in free mode, the hex encoding does not change with direction (which makes sense). It stays in the format shown in the *Gear Level (traveling backward)* table.

### Throttle



Above is when the throttle is idle, in free mode at gear level 4.



Above is full throttle, in free mode at gear level 4.

Same as gear level, the encoding for throttle is also affected by whether the board is in free mode. When the board is in free mode, throttle starts from 1 and grows in magnitude as the throttle is increased. However, as shown in the image above, the throttle does overlap with the least significant 4 bits of the gear level (maybe to prevent accidental gear changes when applying throttle?).

## What's Next?

There is still a chunk of the payload I haven't been able to decode, but the unencrypted gear and throttle fields are already enough to start building accessories. The ultimate goal of this project is to use the sniffed telemetry to drive a wireless braking light that reacts to what the board is doing. In a future article I'll dig deeper into the remaining payload bytes and the software side of the sniffer.