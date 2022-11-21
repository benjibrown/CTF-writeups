# Kioptrix Level 4

Almost done....Kioptrix 4 here we come!

# Enumeration

We will now perform a routine nmap scan on the kioptrix 4 machine.

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_12-01-33.png?raw=true)

```bash
nmap -sV -sC [ip]
```

It seems some a couple of **Samba** servers are running along with a webserver and ssh server.
Upon navigating to the webserver, we are greeted with a login page. A simple SQL injection payload we could attempt on this server is:
```sql
' or 1=1 or '
```

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_12-05-58.png?raw=true)

After entering this into both the username and password field we are told that our login is invalid. After a bit of digging through the source code, I've come to the conclusion that until we find some more details about this login page, SQL injection seems to be a dead end. So lets try investigating the samba server.
Using a quick nmap scan, we can identify the users on the Samba server as follows:

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_12-18-20.png?raw=true)

```bash
nmap --script smb-enum-users.nse [ip]
```

From this command we can see a few users of potential importance, them being `robert`, `john` and `loneferret`. 
We could maybe log in to the website using one of these usernames and the SQL payload
`' or 1=1#`  as the password.

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_12-19-50.png?raw=true)

It worked! After logging in with both `john` and `robert`, we have managed to get their passwords!

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_12-20-14.png?raw=true)
> If you still cannot login try clearing your cookies.

From here, we can login into the ssh server with the credentials we have found. In this example, I'll be using john's credentials.

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_12-22-52.png?raw=true)

```bash
ssh -oHostKeyAlgorithms=+ssh-dss john@192.168.109.7
```

It seems, we are in some sort of restricted shell. To check what shell we are in run `echo $SHELL`. This command tells us that the shell is `/bin/kshell`. After some research, I've found out that an easy way we can break out of this is by using the command `echo os.system("/bin/bash")`. After running this, we successfully have a shell!

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_12-25-35.png?raw=true)

Now, we can do some digging around. After a while, I came across a file called `checklogin.php` in the `/var/www/` directory. Upon reading this file I noticed that it actually contains credentials for the MySQL database. To login to it run `mysql -u root`.


From here, we ca![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_13-01-24.png?raw=true)n see if we can execute system commands inside MySQL.

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_13-10-38.png?raw=true)

```sql
SELECT * FROM mysql.func;
```

Now that we know we can execute system commands via MySQL, we can upgrade our current users' permissions to allow us to execute commands as root.
To do so, we'll need to use the `usermod` command.

```sql
select sys_exec('usermod -a -G admin john')
```

After running this command, exit MySQL and run `sudo su` to login as root! From here, you could perform some post exploitation but I'll leave that up to you ;)

![enter image description here](https://github.com/benjibrown/CTF-writeups/blob/main/VulnHub/images/kioptrix4/Screenshot_2022-11-17_13-15-38.png?raw=true)

As per the image, you can find the congratulations file in the `/root` directory.
Anyways, thats it! You have successfully pwned Kioptrix Level 4!

