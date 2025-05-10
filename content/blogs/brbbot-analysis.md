---
title: "Analysing Brbbot"
date: 2025-05-10
draft: false
---

In the previous article we unpacked the sample using various methods. In this post we will do static and dynamic analysis on the given sample.

‍

**Static analysis**

![image](/blogs/first-malware/image-20250509150843-q80g296.png)

The sample communicates over the **internet**, possibly to a C2 server, using **DNS resolution**, **HTTP GET/POST**, and **low-level sockets**. It builds and sends HTTP requests, reads headers, and handles responses.

‍

The malware uses **encryption** to protect its data or payload, likely for **obfuscation** or **C2 encryption**.

![image](/blogs/first-malware/image-20250507093619-kblvd2b.png)

‍

**Dynamic Analysis**

We are directly going to execute the malware and see its behaviour.

If you look at Process Hacker, you should notice the malicious process running on the now-infected system called  
brbbot.exe. After letting the process run for about one-half a minute,  I terminate it using Process Hacker.

we will use regshot for taking  **snapshot** of the system's **registry and optionally the file system**

‍

**How to use regshot**

```mathematica
+--------------------------+
|      Start Regshot       |
+--------------------------+
            |
            v
+--------------------------+
|  Take 1st Snapshot       |
|  (Before execution)      |
+--------------------------+
            |
            v
+--------------------------+
|  Run Suspicious Binary   |
|  (Malware or Installer)  |
+--------------------------+
            |
            v
+--------------------------+
|  Return to Regshot       |
+--------------------------+
            |
            v
+--------------------------+
|  Take 2nd Snapshot       |
|  (After execution)       |
+--------------------------+
            |
            v
+--------------------------+
|       Compare Shots      |
+--------------------------+
            |
            v
+--------------------------+
| View Registry/File System|
|      Changes Report      |
+--------------------------+

```

‍

![Screenshot 2025-05-09 151703](/blogs/first-malware/Screenshot%202025-05-09%20151703-20250509151740-4jyae3d.png)

**comparing**

![image](/blogs/first-malware/image-20250509152146-qggvvoj.png)

‍

![image](/blogs/first-malware/image-20250509152557-w65k07s.png)

‍

**Network activities done by the malware**

 I have already turned on wireshark, before executing the malware and set the default gateway to remnux machine on my windows system   and here is the output i got.

![image](/blogs/first-malware/image-20241020115414-3yrfrr7.png)

‍

**DNS Query (Frame 7)** 

The source IP `192.168.44.129`​ sends a DNS query to `192.168.44.128`​ (likely a DNS server) asking for the IP address corresponding to the domain `brb.3dtuts.by`​

* Query ID: `0xf0a8`​.

  **Purpose of Query ID (0xf0a8) ?** 

  **Uniqueness**: ***Each DNS query has a unique Query ID to differentiate it from other queries sent to the DNS server***​ **.**  This helps the server and the client identify which response corresponds to which query, especially when multiple queries are sent simultaneously.

  **Matching Responses**: When a client sends a DNS query to a server, it includes this Query ID in the message. The server then includes the same Query ID in its response. This way, the client can match the response to its original query.

  ‍
* DNS Query Type: `A`​ (which is a request for the IPv4 address of the domain).

	**ICMP Destination Unreachable**  **(Frame 8)** 

The client receives an ICMP message from the DNS server indicating that it cannot reach the requested port (likely port 80 for HTTP).

Specifically, the message states  **"Destination unreachable (Port unreachable),"**  meaning that the client attempted to communicate with a port on the server that is not open or does not have a service listening.

‍

```
1.The first packet is a DNS query from the client trying to resolve a domain name.
```

```
2.The second packet is an ICMP message from the server indicating that a request to a specific port on the server cannot be fulfilled because it is unreachable
```

<span data-type="text" style="font-size: 19px;">So, i started a fake dns server on my remnux vm , and then again captured the network traffic using wireshark</span>.

‍

![image](/blogs/first-malware/image-20250507091800-kz41muv.png)

**nslookup**

![image](/blogs/first-malware/image-20250507091830-7e4pzsh.png)

![image](/blogs/first-malware/image-20250507091920-yilhkac.png)

‍

**let's see the wireshark traffic now**

![image](/blogs/first-malware/image-20241020121549-ea3rtot.png)

‍

* The client (`192.168.44.129`​) successfully resolves the domain `brb.3dtuts.by`​ to the IP `192.168.44.128`​ via DNS.
* After resolving the IP, the client tries to initiate a TCP connection to the server on port 80 (HTTP).
* However, the server immediately resets the connection (RST) both times, preventing any successful connection from being established.

  *Because theres no http server running on port 80.*

<span data-type="text" style="font-size: 22px; color: var(--b3-font-color3);">so let,s start out http server running on port 80.</span>

‍

![image](/blogs/first-malware/image-20241020122013-59980dl.png)

‍

* The client (`192.168.44.129`​) successfully resolves the domain `brb.3dtuts.by`​ via DNS.
* After that, the client initiates a TCP connection to the server at `192.168.44.128`​ on port 80, and the TCP three-way handshake is completed successfully.
* The client then sends an HTTP GET request for a resource, but the server responds with a 404 Not Found, indicating that the resource does not exist.
* After responding, the server and client go through the TCP connection teardown process, with the server closing the connection first, followed by the client.

The GET request is typically transmitted by the web browser to request that the web server provide the designated  
web page or file. In our capture, the resource that's being requested is the output of the /ads.php script that the bot  
expects to find on the web server. The bot seems to provide data to this script in the form of parameters separated  
by ampersands (&), which is a common way of submitting data as part of a GET request.

**HTTP GET Request (Frame 80)** 

* The client sends an HTTP GET request to the server, requesting a resource (`/ads.php?...`​).
* This GET request contains a long query string with various parameters.

![image](/blogs/first-malware/image-20241020122525-kfwyiin.png)

The /ads.php page is not present on the REMnux web server. That's why the server responded with 404 Not Found.  
However, we still accomplished the goal of this experiment, which was determining the purpose of the HTTP  
connection. Based on the data we could see, we can tell that the specimen seems to be sending information about  
the infected system to the attacker.

‍

**using speakeasy for binary emulation**

![image](/blogs/first-malware/image-20250507093316-f27d2t4.png)

here we can see that it created a file(brbconfig.tmp) using CreatefileA .

![image](/blogs/first-malware/image-20250507093340-4lzutym.png)

‍

so i loaded the malware in x64dbg, looked for intermodular calls and found crypt decrypt

![image](/blogs/first-malware/image-20250507092437-yd1d6yo.png)

‍

![image](/blogs/first-malware/image-20250507092540-rulxsfx.png)

![image](/blogs/first-malware/image-20250507092630-ag8964c.png)

5b - gives an hint of a  xor key being used.

![image](/blogs/first-malware/image-20250507093126-3l4d24i.png)

‍

using it in cybercheff

![image](/blogs/first-malware/image-20250507092024-9ixjn41.png)

So the malware is sending the process list to the c2.

‍

‍

‍

‍

‍

‍

‍

‍

‍
