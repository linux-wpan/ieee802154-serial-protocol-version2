IEEE 802.15.4 serial protocol version 2
=======================================

Foreword
--------

*This document describes an update to the [IEEE 802.15.4  serial protocol
version 1][serial-v1]. It is a work in progress and will be subject to frequent
changes.*

This serial protocol is designed to be a common protocol for any [IEEE 802.15.4][IEEE 802.15.4-2006]
device which uses a serial interface. Typically these devices are programmable
by the user and their firmware can be easily changed. The protocol proposes in
the document must be implemented in both the firmware and the driver. Devices
using the serial interface use the (currently unofficial) driver named
["serial"][serial-driver] must be attached to a *wpan* device using
[izattach][linux-wpan-userpace-tool]. Note that the serial driver is not in the
mainline kernel but is easily ported to recent kernel. If you are interested in
using the serial driver with the mainline kernel, please contact [the mailing list][linux-zigbee-ml].

No device currently fully implement this protocol. Once this specification
becomes stable, plans are to add support to the [Redwire Econotag][econotag]
and to the [virtual IEEE 802.15.4 device][virtual-ieee802154].

Motivation
----------

The [serial protocol version 1][serial-v1] was designed several years ago with the RedBee
Econotag in mind. At that time most of its hardware functionalities were not
implemented in the firmware (e.g. hardware auto-ACK). 
Also, some commands defined in the specification are not needed (e.g. *set state*) and/or
will not be implemented in software because of timing requirements (e.g. *clear
channel assessment*).  I believe now is time, before the serial driver is
merged in the Linux kernel, to think about improving the protocol so as to
better match the current devices capabilities.

Overview of the protocol
------------------------

Host and serial dongle exchange messages. Each message starts with sequence of
two bytes ('s' = 0x73, '2' = 0x32), for Serial version 2. This is later
referred as the *starting bytes*. After which comes command/response ID (named
"cmdID" thereafter).  Messages coming from the host have highest bit in cmdID
clear, messages coming from the dongle have high bit set.
All commands are acknowledged by a corresponding response (whose cmdID is
the same, except for the highest bit set). The response indicates the status of
the command (SUCCESS or FAILURE). A failure (FAILURE) is followed by an error
code, that indicate the reason of the failure.

This protocol specifies mandatory commands that MUST be implemented.
It also defines optional commands (such as enabling hardware
auto-acknowledgment), that MAY be implemented, depending on the device
capabilities.

Constant values
---------------

### Status

* SUCCESS (0x00): the command succeeded
* FAILURE (0x01): the command failed, and an error code providing more details follows
* SUCCESS_WITH_EXTRA (0x02): the command succeeded, extra information follows

### Error codes

* BUSY_RX (TBA): device is busy receiving a packet
* BUSY_TX (TBA): device is busy transmitting a packet
* BUSY_UNSPEC (TBA): device is busy (reason not specified)
* TRX_OFF (TBA): transceiver is off
* UNSUPPORTED_CHAN (TBA): device does not support requested channel
* UNSUPPORTED_PAGE (TBA): device does not support requested channel page
* NOT_IMPLEMENTED (TBA): command is not implemented
* UNKNOWN_ERR (TBA): error unknown

### Modes

* ENABLED (TBA): function is enabled
* DISABLED (TBA): function is disabled

### Extra information

* NON_PROMISC: device is not in promiscuous mode



Mandatory commands
------------------

### No-op

This command is sent by the host to the dongle and does nothing. It always
succeeds. This is a way to check the hardware readiness and/or aliveness.
It can help determining at which baudrate the device runs. Here, lack of
response can indicate that the command was not interpretable by the device.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x00</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x80 SUCCESS</td> </tr>
</table>

### Open

Power up transceiver, initialize the hardware for receiving packet.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x01</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x81 &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.

### Close

Power down transceiver, shut down current operations, etc.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x02</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x82 &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.

### Set Channel

Change used page and channel. page&#35; ranges from 0 to 31. chan&#35; ranges from 1 to
26. Typically, a device implementing the IEEE 802.15.4-2006 version of the
standard in the 2.4GHz band will set page to 0 and choose a channels from 11 to 26.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x03 &lt;page&#35;&gt; &lt;chan&#35&gt;</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x83 &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is SUCCESS, the page and channel has been set.
When *status* is FAILURE, the error_code is also returned.
When the channel page is unsupported, the device returns an *error_code* of UNSUPPORTED_PAGE.
When the channel is unsupported, the device returns an *error_code* of UNSUPPORTED_CHAN.

### Transmit Block

Transmit a block of data. *len* is the length of the data block (it MUST not
exceed 125 bytes). *data* SHOULD contain a valid IEEE 802.15.4 frame (without
the FCS field, that is calculated by the dongle).

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x04 &lt;len&gt; &lt;data * len&gt;</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x84 &lt;status&gt; [level/error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.

### Receive Block

The only message that is initiated by the dongle. Indicates received block.
*lqi* is LQI measured during reception, &lt;len&gt; is the length of &lt;data&gt; block.
<data> is the MAC frame without the FCS field. *lqi* values ranging from 0 to
127 (included) indicate a normalized LQI. *lqi* value 255 indicates that no LQI
is (temporary or not) available. Other *lqi* MUST not be used.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (dongle)</td><td>'s' '2' 0x05 &lt;lqi&gt; &lt;len&gt; &lt;data * len&gt;</td> </tr>
<tr><td>Response (host)</td><td>'s' '2' 0x85 &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned to the dongle.

### Get 64-bit (long) address

Request a 64-bit (long) address from the dongle.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x06</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x86 &lt;status&gt; [address * 8 bytes]</td> </tr>
</table>

When *status* is SUCCESS, *address* (8 bytes) is returned and contains the
64-bit hardware address. The first byte on the wire contains the Least
Significant Byte (LSB) of the address. When *status* is FAILURE, an
error_code is returned.

Optional commands
-----------------

### Energy Detection (ED)

Request an Energy Detection (ED) measurement.  Upon success, it provides an
estimate of the receiving power within the bandwidth of the channel averaged
over a period of 8 symbols.  A level value is then returned. Section 6.9.7 of
the [IEEE 802.15.4-2006] standard provides additional information on how to
process this value.
This command can be used with the *set channel* command in order to scan for available channels.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x07</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x87 &lt;status&gt; [level/error_code]</td> </tr>
</table>

When *status* is SUCCESS, an Energy Detection *level* for the current channel is returned.
When *status* is FAILURE, an error_code is returned.

Appropriate *error_code* value can be:
* TRX_OFF: transceiver is off, i.e. the open command need to be executed first
* BUSY_TX: transceiver is busy sending, another attempt can be made at a later
  time (this can never happen if the device implements blocking TX)

### Set 64-bit (long) address

Set a 64-bit (long) address to the device. The *address* must be passed with
the Least Significant Byte first (LSB ordering).

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x08 &lt;address * 8 bytes &gt;</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x88 &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.

### Set 16-bit (short) address

Set a 16-bit (short) address to the device. The *address* must be passed with
the most Significant Byte First (LSB ordering).

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x09 &lt;address * 2 bytes &gt;</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x89 &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.

### Set PAN Identifier

Set a PAN identifier to the device. The *panid* must be passed with the most
Significant Byte First (LSB ordering).

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x0a &lt;panid * 2 bytes &gt;</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x8a &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.

### Enable/Disable promiscuous mode

*Depends on*: set 64-bit address, set 16-bit address and set PAN identifier

Enable or disable promiscuous mode in a device. *mode* indicates if the
promiscuous mode needs to be ENABLED or DISABLED. In promiscuous mode, the
device will pass any traffic it receives from the transceiver. When this mode
is disabled, only the packets that destined to the same PAN, and to the short or
the long address assigned to the device or a broadcast address will be passed to
the host.

<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x0b &lt;mode&gt;</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x8b &lt;status&gt; [error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.

### Enable/Disable hardware auto-acknowledgment

*Depends on*: set 64-bit address, set 16-bit address and set PAN identifier

Enable or disable hardware auto-acknowledgment. *mode* indicates if the
hardware auto-acknowledgment feature is ENABLED or DISABLED.TBA


<table border=1>
<tr><td>Type (entity)</td><td>Bytes sent</td></tr>
<tr><td>Command (host)</td><td>'s' '2' 0x0c &lt;mode&gt;</td> </tr>
<tr><td>Response (dongle)</td><td>'s' '2' 0x8c &lt;status&gt; [extra_info/error_code]</td> </tr>
</table>

When *status* is FAILURE, an error_code is also returned.  Enabling the
hardware auto-acknowledgment can disable the promiscuous mode on certain
devices. When that happens, the device returns *status* SUCCESS_WITH_EXTRA and
set *extra_info* to NON_PROMISC.

Handling non-implemented commands
---------------------------------

The device must respond to unknown commands with the following sequence *'s'
'2' &lt;response-id&gt; FAILURE NOT_IMPLEMENTED* (thus indicating that the
command is not supported). The response-id value is the unsupported cmdID of
the command (i.e. the request) whose higher bit has been set.

Default behavior at boot time
-----------------------------

Once the *open* command has been executed successfully, the device should be in
the following states:

* the non-beacon enabled mode and does not track any beacon
* no hardware function is started (no accelerate crypto module is initialized, no auto-ACK, etc.)

Note that the following aspects are unspecified, thus the following value
should be set on the device after a power cycle:

* channel
* 16-bit (short) address of the device
* 64-bit (long) address of the device

Because it is not known if the device support promiscuous mode, not assumption
can be made about the default mode of a device upon a power cycle and is
entirely device dependent.

Miscellaneous
-------------

* a reset of the hardware requires calling the *close* and the *open* command
  in that order. This is the current behavior of the serial driver.


Contact
-------

This document is currently a draft of what the version 2 of the serial protocol
should be and need external contribution and reviews to be improved. You
welcomed to discuss the protocol on the [linux-zigbee-ml] or send me an email
(tony.cheneau@amnesiak.org or tony.cheneau@nist.gov).

Appendices
==========

Current implementation status
-----------------------------

### Linux driver

TBA

### Redbee Econotag firmware

TBA

### Virtual IEEE 802.15.4 device

The [Virtual IEEE 802.15.4 device ][virtual-ieee802154] offers a software
solution to test 6LoWPAN/IEEE 802.15.4 code over non IEEE 802.15.4 links or
locally.

TBA.

Differences between the serial protocol version 1 and version 2
---------------------------------------------------------------

### First bytes have been changed from "zb" to "s2"

The starting bytes are now "s2" instead of "zb".

**Rational**:
* better reflect that the implementation is not connected to any ZigBee related effort
* prevent implementation of [Serial Protocol version 1][serial-v1] from working with a device implementing this document
* the "2" part can be change to a later value if the protocol even needs to be implemented

### Set Channel does not use an offset

**Rational**: The IEEE 802.15.4 standard enables a device to operate on various
frequency bands (e.g. 2450 MHz, 915 MHz, 868 MHz bands, and maybe more in the
future).  Previous version of the protocol would use an offset to map channels
11 to 26 of the 2450 MHz band to values 1 to 16. Such an offset prevent usage
of device that operates in different bands and thus is not desirable.


### Set State command has been removed

**Rational**: a device can enable the TX mode automatically when transmitting data.


### Energy Detection (ED) is optional

**Rational**: exporting the energy detection feature to the user is not crucial
for a proper functioning of the devices

### Clear Channel Assessment (CCA) command has been removed

**Rational**: this function is implemented on the device side. Furthermore, it
is not possible to execute this command on a serial like in a timely manner.

### A new NO-OP command has been introduced

**Rational**: when debugging a device, it is very convenient for the developer
to have a little probe tool that cause no changes to the device and still
inform on the device availability.

### Command values have been reassigned

**Rational**: command values has been freed by removing several obsolete
commands. It makes sense to try to use contiguous values of the available
"command" space.


<!-- list of references -->

[virtual-ieee802154]: https://github.com/tcheneau/virtual-ieee802154-serial
[serial-v1]: http://sourceforge.net/apps/trac/linux-zigbee/wiki/SerialV1
[linux-zigbee-ml]: http://sourceforge.net/apps/trac/linux-zigbee/wiki/MailingList
[serial-driver]: http://TODO
[linux-wpan-userpace-tool]: http://sourceforge.net/apps/trac/linux-zigbee/wiki/BuildingUserspaceTools
[econotag]: http://redwirellc.com/store/node/1
[IEEE 802.15.4-2006]: http://standards.ieee.org/findstds/standard/802.15.4-2006.html
