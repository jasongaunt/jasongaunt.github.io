# Alpine Ai-NET Documentation

- [Preamble](#preamble)
- [The Ai-NET interface](#the-ai-net-interface)
- [The physical layer](#the-physical-layer)
- [Signalling](#signalling)
- [The Ai-NET protocol](#the-ai-net-protocol)
- [Known nodes](#known-nodes)
- [Known commands](#known-commands)
- [Credits and thanks](#credits-and-thanks)
- [Disclaimer and Copyright](#disclaimer-and-copyright)

## Preamble

I recently bought a [used car](https://en.wikipedia.org/wiki/Mitsubishi_Galant_VR-4#Seventh_generation_(E84A/E74A)) that came with an old Alpine CDA-7893R head unit. At first I wanted to swap this to Android Auto, but then I realised I wanted to keep this classic head unit that perfectly fit the aesthetic of the 33 year old car it came from.

I opted to keep the head unit but with one caveat; it **must** have *good* Bluetooth support. That is, the ability to pause and skip tracks directly from the head unit itself.

From what I can tell, Alpine never sold an official Bluetooth Ai-NET receiver for this generation of head unit and aftermarket options either didn't support AVRCP (remote control) or were stupidly expensive if they did.. so the [Ai-NET Bluetooth Receiver](https://github.com/jasongaunt/Ai-NET-Bluetooth-Receiver) project was born.

This github pages entry serves as the documentation leading up to the implementation of that project. May it help you with yours.

## The Ai-NET interface

This documention only covers the staggered 8 pin connector you see in this pinout:

![Ai-NET Male and Female connectors](ai-net%20connectors.jpg)

## The physical layer

There are two pins used for communication, `Ai-NET +` and `Ai-NET -`. These behave very similar to RS485 and CAN-BUS in that they are inverted reflections of each others behaviour. There are two states for this bus, dominant and recessive, with recessive being the default state.

| State     | Ai-NET +    | Ai-NET -   |
| --------- | ----------- | -----------|
| Recessive | 2.0~ Volts  | 3.0~ Volts |
| Dominant  | 5.0~ Volts  | 0.0~ Volts |

These voltages are guidelines, they can *and do* fluctuate, what's important is the following:

* To signal recessive, `Ai-NET +` must be a lower voltage than `Ai-NET -`
* To signal dominant, `Ai-NET +` must be a higher voltage than `Ai-NET -`
* Voltages should not exceed 5 Volts. Above 5.5 Volts can damage hardware
* All voltages are relative to chassis ground not audio or any other ground

The recessive state voltages are achieved by weak pull-up resistors in a potential divider set up, one to 5V, another to ground. The resistors vary between `Ai-NET +` and `Ai-NET -` but the principle is the same, holding the floating voltage to either 2.0 or 3.0 volts. Dominant signals are achieved by applying a strong pull-up (to `Ai-NET +`) and strong pull-down (to `Ai-NET -`) at the same time. See the example circuit below for more information.

Although you can "cheat" and just pull `Ai-NET -` to ground to signal dominant, this is not advised as it ignores the fault-tolerance of having mirrored signals. You should however implement a circuit like the following:

![Ai-NET Interface Circuit](ai-net%20circuit.jpg)

An interactive version of this circuit can be found on [Falstad CircuitJS here](https://www.falstad.com/s.php?s=w1FSDC).

## Signalling

The Ai-NET protocol signalling is almost identical to SAE J1850 *PWM* (not to be confused with the much more popular VPW variant) and was commonly used in early 90s Ford's. The protocol *aims* for a **41.67 kHz cycle / baud rate** but this is only the timing between each bit and additional factors such as the Start of Frame pulse cause this to differ. 

```
                        <-----Start of Frame------><-Logic 1-><-Logic 0-><-Logic 1-><-Logic 1-><--End of transmission
          ______________                    ______     ______        ___     ______     _____________________________
RECESSIVE               |                  |      |   |      |      |   |   |      |   |                             
DOMINANT                .__________________.      .___.      .______.   .___.      .___.                             

State                   D                  R      D   R      D      R   D   R      D   R
Time (microseconds)     0                  32     48  56     72     88  96  104    120 128
```

Communication is achieved by varying the duty cycle between 33% and 66%:

* A logical 1 would have a dominant state of 33% (or 8 microseconds) of the pulse widths time
* A logical 0 would have a dominant state of 66% (or 16 microseconds) of the pulse widths time

Each bits pulse is 24 microseconds in length (41.67 kHz) and forms usually 11 bytes of data called a *frame.* At the start of each frame is the SoF (Start of Frame) pulse which is 32 microseconds dominant followed by 16 microseconds recessive.

Each frame is usually 10 bytes of data plus an 11th CRC byte, which is the [standard SAE J1850 CRC calculation](https://crccalc.com/?crc=&method=SAE&datatype=hex&outtype=hex).

The only exception to the above is an ACK frame. The ACK frame is a 1 byte transmission that retransmits the first byte of the received transmission to confirm receipt. No Start of Frame is sent and this must be transmitted as soon as possible *after* 32 microseconds has elapsed (so as not to clash with an incoming SoF). This is a little flexible but ideally it should be around 40 microseconds.

## The Ai-NET Protocol

Ignoring the CRC byte at the end, an Ai-NET frame has 10 bytes of data (as we mentioned earlier). Each byte can be loosely defined as follows (starting with byte 0):

| Byte 0      | Byte 1 | Byte 2 | Byte 3 | Byte 4 | Byte 5 | Byte 6 | Byte 7 | Byte 8 | Byte 9 |
| ----------- | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ | ------ |
| Destination | Sender | Data 0 | Data 1 | Data 2 | Data 3 | Data 4 | Data 5 | Data 6 | Data 7 |

**Destination** and **Sender** are single byte identifiers for nodes (devices) and can be found in the [Known nodes](#known-nodes) section below.

Data 0 is *typically* a command identifier and Data 1 a sub-command, although there are a few frames where this isn't the case. A list of known commands can be found in the [Known commands](#known-commands).

## Known nodes
## Known commands

These are currently stored on [a Google Sheet here](https://docs.google.com/spreadsheets/d/1nzEyYhiziHJZetRuuANDKpPxwW4N5Pad-qV-VGqKM3o/edit?usp=sharing).

## Credits and thanks

Nik1976 from the [Ai10](https://groups.google.com/g/proj-ai10) project for providing basic protocol and electrical interface circuit information.

My friends Garry and Jake for putting up with my excited ramblings as I discovered something new.

My partner, who wishes to remain nameless, for putting up with broken sleep as I frequently came to bed late.

## Disclaimer and Copyright

All information contained herein was developed privately by monitoring communication on an Ai-NET bus between an Alpine head unit and various Alpine devices.

Alpine (or any parent or subsidiary) do not support this in any way. They did not contribute to or endorse it. Do not pester them for support. This information is provided "as-is" without any express or implied warranty.

Ai-NET is a trademark of Alps Alpine Co., Ltd. Alpine aka Alpine Electronics, Inc. is a subsidiary of Alps Alpine Co., Ltd. All other trademarks are the property of their respective owners.
