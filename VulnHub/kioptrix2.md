# Kioptrix Level 1

Potentially one of the easiest CTF's ever...Kioptrix
However, it is a very good CTF nonetheless.

# Setup/Prequisities
- A Brain
- Able to host Kioptrix virtual machine
- Nmap

Once you have setup your Kioptrix vm, find its IP using
 `netdiscover -i eth0` and the run `export IP=[machine-ip]`, this will simply create a variable called IP defined as your machine's IP so you do not need to remeber it.

# Enumeration

First, we'll run a routine nmap scan and save the output to a file for fututre reference.

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_26_10_2022_15_38_10.png?raw=true)

```bash
nmap -sV -sC -oN scan.txt $IP
```
As you can see, we have 6 services running on the machine them being ssh,  http, rpcbind, netbios-ssn, https and filenet-tms. There is definitely a webserver running on this machine, upon visiting it in our browser we see a default apache test page. 
Our nmap scan shows us that it is running Apache version 1.3.20, after a quick google search, I've found that there is a known exploit for this version of Apache (called OpenFuck).

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/Screenshot%202022-10-26%20154801.png?raw=true)

However, this exploit looks a little outdated, so in this case we'll be using a version of the OpenFuck by Helton Wernik [here](https://github.com/heltonWernik/OpenLuck).
So first, lets set this exploit up. As per the github repo's installation instructions, we'll run the following:
```bash
git clone https://github.com/heltonWernik/OpenFuck.git
apt-get install libssl-dev
gcc -o OpenFuck OpenFuck.c -lcrypto
```

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_26_10_2022_15_54_02.png?raw=true)

Almost there! Now, we are going to execute the exploit to do this, we'll need to use the exploit `0x6a`.

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_26_10_2022_15_58_12.png?raw=true)

```bash
./OpenFuck 0x6a [machine-ip] 443 -c 40
```
If using the option `0x6a` fails, try `0x6b`. If that also fails then restart Kioptrix and restart Kali. In addition, try running tcpdump to troubleshoot.
If executed correctly, you should get a shell. Congrats!
From here, we need to make our shell a bit more useable. So run `/bin/bash -i`. Now we need to do some more digging. What i always do first is to run the `history` command to view the commands that have been executed in that machine. 
There aren't too many interesting commands however the command `mail` does seem to stand out. 

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_26_10_2022_16_02_00.png?raw=true)

After running it, we are greeted with 2 emails, enter 1 to view the first and 2 to view the second. From there, we can see we have successfully pwned Kioptrix!
