# Ankuoo Wireless Switch / Plug Notes
The following are notes from reverse engineering of the protocol used by the NEO app to interact with the Ankuoo Neo (not Neo Power) WiFi swithces/plugs.
## Models Used
 - Ankuoo Neo Switch - A wall switch replacement
 - Ankuoo Neo Plug - The version that does not feature power monitoring

Other models may use different values.
## Similar Models
The Ankuoo switch is a Broadlink variant, however, it is not compatible with Broadlink software and will not work with any other broadlink based device.  There are several devices that appear to be Broadlink clones, however, for the Ankuoo, the models map in the following manner.

| Model     | Broadlink Model | Feature Set                                                                              |
|-----------|-----------------|------------------------------------------------------------------------------------------|
| Neo       | Broadlink SP1   | Internet/Local Power On/Off, Schedule per Day of Week On/Off, Anti-Theft, Invisible Mode |
| Neo Power | Broadlink SP2   | All of the features of Neo plus Power Usage Monitoring by Day/Week/Month/Year            |

### Unconfirmed Similar Models

 - SmartLink WiFi - Appears to use the same UDP packet based communication with similar format except has a different "key" component (5aa5aa55... from below).  These devices may or may not have a web service component as well
 - Maginon (was available at many Aldi stores over Christmas 2015) - Same as above.
 
## Internet Communication
All devices communicate regularly with a cloud host (likely in lieu of having a "hub" device on the network).  
#### Protocol
The communication is done over UDP.

__*TODO:*__ Determine if TCP is a fall-back should UDP be impossible to use.

#### Internet Hosts Contacted:
The devices perform DNS queries for the following hosts
 - us.broadlink.com.cn -> 54.227.239.229
 - eu.broadlink.com.cn -> 54.217.219.157

It's possible the second host was queried as a result of my sniffer having blocked communication to all external hosts from the device (I had DHCP configured to point the gateway at a unix host that I was using to MITM SSL in case the device used SSL in some way).

The application appears to contact the following IPs without looking up by DNS
 - 112.124.42.42 (only ever seen on port 80)
 - 112.124.35.104 (only ever seen on port 8080)

In theory, if one wanted to prevent the device from communicating with the internet host, it'd simply be a manner of pointing the DNS entries at a local host and providing the correct responses to packets.

__*TODO:*__ Determine what happens when both hosts are allowed and blocked.

#### Ports

 - Port 80 (http if it were TCP)
 - Port 8080 (http_alt if it were TCP)
 - Port 1812 (radius)
 - Port 16384 - Not Assigned but used by Cisco RTP according to Wikipedia

Port choices appear to be to to get around restrictive firewalls

## Packets Sent/Received Directly From Devices

These include command packets, and their responses.  For some reason, the app and the device reply to everything with two identical packets.  This might be a hack to help cover dropped UDP packets or there might be some other reason, however, in my tests, the devices respond fine with just a single packet sent.

### Common fields for All Packets

###### 32-bit Key
All communication of this kind starts with a 32-bit key that's identical regardless of the device
```
00: 5aa5 aa55 5aa5 aa55
08: 0000 0000 0000 0000
16: 0000 0000 0000 0000
24: 0000 0000 0000 0000
```

###### Timestamp
The next field is a 32-bit little-endian time-stamp value of some kind in seconds.  When one device's packets are monitored, the distance between this value and a future value sent from the device is the same as the length of time recorded in Wireshark between packets.  It appears to be a time-span of some kind, though whatever value it's counting forward from is unknown.  The two devices use wildly different values for this.

__*TODO:*__ Determine if packet type matters.
```
32: xxxx 0000 .... ....
```

##### Device Type
The device type appears on all command packets, but not on some communication to/from the internet.

```
32: .... .... xxxx yy00
```

 - __x__ - Represents the device type: 0x2717 represents all "non-power" Ankuoo devices.  The "switch" and "plug" do not have separate identifiers.  When it is not set, it's all zeros.  This is *always* set on command packets.
 - __y__ - Unknown '01' and '02' have been observed - this might be the "sub-type" of the device or something else

###### MAC Address

```
40: 0000 xxxx xxxx xxxx
```
 - __x__ - Little-endian MAC address

### Internet Packets To/From Device
#### Heartbeat Packet
The device appears to send a periodic (rather frequent) request to the internet, possibly a Heartbeat.  The outbound packet contains no fields beyond what is in the Common Fields for All Packets.

#### Heartbeat Reply
The response to the packet is identical to the outbound, except the value after the device ID is a different value (observed 1 in the first byte) and at the end of the packet, a value indicating the current time is returned.

###### Time Encoding
The encoding of the time is ... odd.  The time is returned in the local time of the device, which I assume is set by the phone's local time when the device is joined to the network.

```
48: yyyy ssnn hhpp ddmm
```

 - __yyyy__ - Little-endian uint of the year
 - __ss__ - Current Seconds
 - __nn__ - Current Minutes
 - __hh__ - Current Hours on a 12-hour clock, set to "0" means "12", otherwise subtract 1 from the hours to get the current hour
 - __pp__ - AM/PM
 - __dd__ - Current Day of the month
 - __mm__ - Current Month of the year

### Query Response Packet
