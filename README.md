# SSH Hardening a Local Server

## Objective

The SSH Hardening project was my first project after installing Ubuntu server on an old PC tower. I decided that SSH was the first service I wanted to setup so I could have remote access to the server. I knew that threat actors are constantly scanning ports and services, so having SSH locked down was incredibly important for me. As well, I chose to have fun with this project and went overboard implementing as many tactics as I could learn. This hands-on experience helped me understand network architecture, network security, and defensive strategies.

### Skills Learned

- SSH functions
- SSH MFA setup
- Configuring SSH through sshd_config file
- SSH key generation
- Enabling Google Authentication PAM
- Port knocking
- iptables
### Tools Used

- openSSH 
- Google Authentication PAM
- knockd 
- iptables rules
- Online documentation

### Steps

1. Edit SSH config file on server for basic hardening
2. Enable SSH public key authentication and disable password authentication
3. Install Google Authenticator PAM & set to required for authentication
4. Setup knockd & add iptables rules to block SSH port

## Process
### Configure sshd_config file 

Before disabling the root login, we need to ensure that we can SSH into the server as the new user and test sudo. After confirming both, open sshd_config file at `/etc/ssh/sshd_config` and make the following changes: 

* Disable root login
* Change SSH Port
* Add allowed and denied users
* Reduce the max sessions allowed
* Auto-terminate idle sessions

We use the command `SSH user@host` to get into the server. We can use the IP address after the `@` or we can use the name of the machine if it is visible on the local area network. To SSH into the server we use the following command: 
```
naomi@local-machine:~$ ssh naomi@local-server
```

We can use the simple command `sudo -l` to check if we are able to access sudo. If we have access to sudo it will tell us. If not, you will need to login to another account with sudo or login as root to grant yourself sudo access. 

```
naomi@local-server:~$ sudo -l
Matching Defaults entries for naomi on local-server:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User naomi may run the following commands on local-server:
    (ALL : ALL) ALL
```

Once we confirm we have sudo, we will edit the `/etc/ssh/sshd_config file` as noted above. 

```console
"/etc/ssh/sshd_config"
...
Port 57625
...
# Authentication:

#LoginGraceTime 2m
PermitRootLogin no <change to no>
#StrictModes yes
#MaxAuthTries 6 
MaxSessions 2 <change to 1 or 2>

#Users Allowed <add this line and the two below, using your own username>
DenyUsers root 
AllowUsers naomi
...
TCPKeepAlive yes  <make sure this is set to yes>
#PermitUserEnvironment no
#Compression delayed
ClientAliveInterval 60 <This is how often a CAI is sent in seconds>
ClientAliveCountMax 5 <This is the number of times the CAI can be sent before the session is terminated>
...
```

With all of this changed, we can now SSH into your device to test that we still have access. In order to SSH onto our new port we use the following command (using whatever port you chose):

```console
naomi@local-machine:~$ SSH naomi@local-server -p 57625
```
### Create Public/Private Key Pair

#### Check PubkeyAuthentication and AuthenticationMethods

First, ensure that the public key authentication is set to yes and, if authentication methods are listed, that they include password and public key. 
```console
"/etc/ssh/sshd_config"
---
PubkeyAuthentication yes
AuthenticationMethods password,publickey
```

#### Create the public/private key pair

To create the public/private key pair, we have to be on the machine we want to access the server from. The public and private keys will be generated using `ssh-keygen` and then copied to the server using `ssh-copy-id`. Once that is done, we SSH back into the system and the key pair should let us in without our password. Once that is confirmed, disable password login so that SSH cannot be brute forced. 

In this instance, the ed25519 key is used and a passphrase is setup. 
```console
naomi@local-server:~$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/naomi/.ssh/id_ed25519): local-server
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in server_name
Your public key has been saved in local-server.pub
The key fingerprint is:
SHA256:DNGw5rA4aqavRD2swwHvxrSSkF93YOM2byUske7W2Cc naomi@local-server
The key's randomart image is:
+--[ED25519 256]--+
|      oo         |
|       +.        |
|.   . X          |
|.oo. O B         |
|oo=+o O S .      |
|+Bo+.+ O o       |
|+B*   + E .      |
|=o.  . . o       |
|oo.              |
+----[SHA256]-----+
```


#### Send Public Key to Server via SSH

After going through the prompts setting up the key on the local machine, use `ssh-copy-id` to send the key to the server. 
```console
naomi@local-server:~$ ssh-copy-id -i home/naomi/.ssh/local-server naomi@local-server
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/naomi/server-name.pub"
The authenticity of host 'local-server (172.0.0.1)' can't be established.
ED25519 key fingerprint is SHA256:1uKM1vcR5tujYWTFMnMoHUylNGzbmvAOsE6jSxHnTAc.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
naomi@local-server's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'naomi@local-server'"
and check to make sure that only the key(s) you wanted were added.
```

After a successful login using the key pair, the password login needed to be disabled so that ssh cannot be brute forced. Opening up the sshd_config file again, we want to set the password login to no and remove password from authentication methods.
```console
"/etc/ssh/sshd_config"
---
PasswordAuthentication no
```

After this, SSH is set up to only accept key access for authentication. 

### Install Google Authentication PAM (TOTP)

#### Install Google Authenticator PAM

For this, because the server is running Ubuntu, I used the <a href=https://ubuntu.com/tutorials/configure-ssh-2fa#1-overview>Ubuntu 2 factor authentication tutorial. </a>

To get started, we want to make sure we are up to date by running apt-update. Then use install the libpam-google-authenticator library. 
```console
naomi@local-server:~$ sudo apt update
naomi@local-server:~$ sudo apt install libpam-google-authenticator
```

After that we will add Google PAM authentication requirement. To do this, we go to `'etc/pam.d/sshd` and add the following line: 
```console 
"/etc/pam.d/sshd"
---
auth required pam_google_authenticator.so
```

Then restart the `sshd` daemon with: 
```
naomi@local-server:~$ sudo systemctl restart sshd.service
```

Lastly, we will change `KbdInteractiveAuthentication` in `etc/ssh/sshd_config` to yes. (this differs from the Ubuntu tutorial, which uses a deprecated `ChanllengeResponseAuthentication` key value pair). 

#### Configure Google Authenticator

To start the configuration we run the command `google-authenticator` and proceed through the prompts. 

In order to progress, you need to have an authenticator app like Google Authenticator or Yubikey Authenticator available. After running the initial command and QR code will pop up (if the image is too large for your terminal, use ctrl + "-" to zoom out and ctrl + shift + "+" to zoom back in). Scan the QR code to add it to your authenticator app or use the secret key listed below the QR code. 

Once added to your authenticator app, enter the 6-digit code that is provided where it says `Enter code from app`. 

```
naomi@local-server:~$ google-authenticator

Do you want authentication tokens to be time-based (y/n) y
Warning: pasting the following URL into your browser exposes the OTP secret to Google:
  https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://totp/naomi@plocal-server%3Fsecret%3DSJO2ZGQXUZ45HAZXCUQ6OEGGZM%26issuer%3Dlocal-server
---
[Google Authenticator QR Code]
---
Your new secret key is: ****************
Enter code from app (-1 to skip): ******
Code confirmed
Your emergency scratch codes are:
  ********
  ********
  ********
  ********
  ********
```

Once you have added the authenticator you'll be walked through several questions. I chose to have the file updated (although it should already be the latest version), to disallow multiple uses of the same code, and to introduce rate limiting. However, I chose not to increase the window of permitted codes because I am frequently using all of my devices from the same network so everything should be properly synced. 

```
Do you want me to update your "/home/naomi/.google_authenticator" file? (y/n) y

Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y

By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server, we allow an extra token before and after the current time. This allows for a time skew of up to 30 seconds between authentication server and client. If you experience problems with poor time synchronization, you can increase the window from its default size of 3 permitted codes (one previous code, the current code, the next code) to 17 permitted codes (the 8 previous codes, the current code, and the 8 next codes). This will permit for a time skew of up to 4 minutes between client and server.
Do you want to do so? (y/n) n

If the computer that you are logging into isn't hardened against brute-force login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y

```

At this point, the Google Authenticator PAM is installed and setup for use with SSH login. Test that the login is working by using the `exit` command in the terminal and then SSH back into the system. It should automatically prompt you to provide the code from your authenticator app. 

### knockd

**Note:** you must have physical access to the server OR the ability to remote into the server to troubleshoot this. Because of the iptables configuration, if there is an issue with the knockd configuration you will end up locked out of your server. 

Knockd works by monitoring what ports a device attempts to access. When a device attempts to access the correct ports in the correct order, knockd runs a shell script that adds an iptables rule to allow internet traffic from the ip address of the device that sent the knock on the port specified. We also need to install iptables-persistent in order for our changes to be consistent. Keep in mind are that the SSH port was changed at the very beginning of this process. In my case, I changed it to port 57625. 
#### Configure iptables

We will make the following changes to iptables: 
* Allow all incoming established and related traffic to accept
* Drop all incoming traffic directed to our SSH port

First, ensure that everything is up to date with apt update
``` console
naomi@local-server:~$ sudo apt update
```

Next, install the iptables-save package
```console
naomi@local-server:~$ sudo apt-get install iptables-save
```

We set our iptables rules using the following commands: 
```console
naomi@local-server:~$ sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
naomi@local-server:~$  sudo iptables -A INPUT -p tcp --destination-port 57625 -j DROP
```

To save our tables we simply invoke the `iptables-save` command
```console
naomi@local-server:~$ sudo iptables-save
```
#### Install & Configure knockd

Using our package manager, we install knockd easily: 
```
naomi@local-server:~$ sudo apt install knockd
```

Once the installation is done we will edit the knockd.conf file at `/etc/knockd.conf`. We will be changing the shell script that knockd runs so that it inserts instead of appends the new rule and the port that it opens. As well, we will set custom ports for the knocking sequence so that they aren't the default ports. 

Open the knockd configuration file with your editor of choice: 
```console
sudo vim /etc/knockd.conf
```

Then, we edit the file by replacing the command tabbed inside `[openSSH]` after `/sbin/iptables` to `-I INPUT -S %ip% -P TCP --dport 57625 -j ACCEPT`. Note: the number after `--dport` has to be the port you use for SSH. Under `[closeSSH]` change the number after `--dport` to your SSH port as well. Lastly, change the sequence to any ports you want. It does not have to be 3 ports, but a 

You should end up with a configuration file that looks similar to this. 
```
[options]
        logfile = /var/log/knockd.log
        
[openSSH]
        sequence    = 8650,7932,9471
        seq_timeout = 5
        command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 57625 -j NFQUEUE
        tcpflags    = syn

[closeSSH]
        sequence    = 9643,6398,9246
        seq_timeout = 5
        command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 57625 -j NFQUEUE
        tcpflags    = syn
```

Now we have to start the knockd service so that it can listen for the knock and enable it so that it starts every time the server reboots. If not, we are locked out of the server if it goes down or has to reboot until we can physically access it. 
```
naomi@local-server:~$ sudo systemctl start knockd
naomi@local-server:~$ sudo systemctl enable knockd
```

Check that the startup was successful with the following command: 
```
naomi@local-server:~$ systemctl status knockd
```

Knockd is now installed, configured, and ready 
#### Configure the client for knocking

On our local machine we will install knockd just like we did on the server. 
```
naomi@local-machine:~$ sudo apt install knockd
```

Once it's done installing we use the command `knock` in order to open up the port. I use -v so that it outputs all of the knocks. We can use the IP address of the server, the local name, or we can use a domain name if one is setup. Then, we indicate what ports we are knocking on. This has to match the ports you setup in the `knockd.conf` file earlier. 
```
naomi@local-machine:~$ knock -v local-server 8650 7932 9471
```

There will not be any output in the terminal as to whether your knock was successful. To test if your knock was successful, simply ssh into the server like normal. 

```
naomi@local-machine:~$ ssh naomi@local-server -p 57625
```

If the terminal prompts you to login, it was successful. 

---

Conclusion: 

This was an extremely fun and informative exercise that all provided me real world benefit. I felt more comfortable having my server connected to the internet. Given my lack of experience with Linux, it also forced me to become comfortable navigating and working within the terminal. Lastly, it taught me a lot about networking and how everything works together. 

I want to recognize that having all of these tactics can be detrimental to workflow if SSH is used frequently in short spurts. Having to use the port knocking and the authenticator can be a pain when using SSH to get in and out of the machine repeatedly. I mitigated this issue by creating a bash alias that goes through the port knock and the SSH without any arguments. As well, I set up a host profile for my local server so SSH didn't require options like changing the port. 

I'm really glad that I took on this exercise and stuck it through, as I was doing this project alongside completing my Google Cybersecurity Specialist Certificate. 