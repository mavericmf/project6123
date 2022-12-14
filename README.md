# Project6123 | SF Signalling For Asterisk
A project for adding full SF Signalling support to Asterisk.

## What is Project6123 / SF Signalling?
Project6123 adds full SF trunking support to Asterisk (an open source VoIP telephony engine). SF signalling has been used since the early days of metallic telephone trunking, essentially as a way to send signalling over a standard voice band only line. Unlike your regular 2 wire analog lines, SF signalling was mainly used for long distance circuits, microwave relays, and tie-lines. It was made so that signalling could be sent in-band, without requiring extra signalling wires, like E&M signalling, or high voltages, like FXS ringing.

SF Signalling uses a 2600Hz tone (in the U.S.) to signify an on-hook status. When the line is idle, a 2600Hz tone is being sent in both directions.

Example Setup:
```
============               ============
| SF Trunk |-TX---------RX-| SF Trunk |
|    A     |-RX---------TX-|    B     |
============               ============
```

NOTE: SF SIGNALLING ACTS LIKE A RINGDOWN TRUNK
WHEN 2600Hz IS LOST, IT "RINGSDOWN" THE OTHER SIDE.

If either side goes off-hook, aka, removing the 2600Hz tone, the other side can answer by also removing it's 2600Hz tone.
When either side hangs up, it will play a 2600Hz tone, causing the other side to hangup, and play the 2600Hz tone again.

Because filters are not perfect, and the 2600Hz detection circuit doesn't act immediately before dropping the call, that causes a very short 2600Hz pulse to be heard on the call audio when someone goes on-hook, or when the call is first placed. This is called a "wink".

## Method A
Project6123 has two methods of operation. Method A "fakes" SF signalling. Instead of a trunk that always has a 2600Hz tone playing, method A just adds support for SF dropping. Operation: it will call the technology specified in the arguments, and when a 2600Hz tone is detected, it will hangup the called party, and return from the subroutine.

```n,GoSub(sf_method_a_scan,s,1(ARG1,ARG2))```

Arguments:

`ARG1` = Winkback? Input options, 0 or 1, this is wheather or not you would like a 2600Hz "wink" to be played if a 2600Hz tone is detected.

`ARG2` = Technology to call? Input options, dial technology (ex. Local/2311000@main), this is the number to call via an "SF" trunk.

Example using SF Method A Scanning:
```
[main]
exten => 1234,1,Answer()
  same => n,GoSub(sf_method_a_scan,s,1(1,SIP/380))
  same => n,Hangup
```

## Method B
The other option, Method B, emulates a real SF trunk. Operation: it generates a 2600Hz tone, it expects a 2600Hz tone, if no tone is heard, or the tone stops, it will dial the technology supplied in the arguments, when a 2600Hz tone is heard again, it will hangup the called party, and resend a 2600Hz tone until the tone drops.
TLDR: emulates a 2600Hz ringdown trunk.

```n,GoSub(sf_method_b_scan,s,1(ARG1))```

Arguments:
`ARG1` = Technology to call? Input options, dial technology (ex. Local/2311000@main), this is the technology that will be called when the 2600Hz tone is dropped.

Example using SF Method B as a SIP context, connected to a real SF to FXS card:
extensions.conf
```
[sf_sip]
exten => _X!,1,Answer()
  same => n,Gosub(sf_method_b_scan,s,1(Local/8@dt))
  same => n,Hangup
```
sip.conf
```
[2311000](ATAs)
defaultuser = 2311000
secret = password
authid = 2311000
context = sf_sip
```

When a call is placed by the SIP user, it will go directly to the SF receiver, when the 2600Hz tone is dropped, it will call the number specified in the arguments, which in the example above is extension 8, at context "dt".
