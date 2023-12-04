# My pi setup

These are the steps to setup my pi.

I guess more than half of the content are copied from the offical website...

<https://www.raspberrypi.com/documentation/computers/os.html>

## Download the OS to the SD card

Using the default OS (Raspberry Pi OS with Desktop?) is probably fine but I would prefer the Raspberry Pi OS Lite since it is without additional software and I only need a headless server.

I choose the 64-bit version.

<https://www.raspberrypi.com/software/operating-systems/>

Also Etcher is needed to write the image to the SD card.

<https://etcher.balena.io/>

## Headless setup

Prepare an SD card and a password. I use wired connection so I won't setup Wi-Fi.

In the boot partition of the SD card which comes together with the kit, create an empty `ssh` file and need to create another `userconf.txt` to create a new user.

In my Linux notebook, the path of the boot partition in SD card is `/media/duck/bootfs`.

```bash
cd /media/duck/bootfs
touch ssh
touch userconf.txt
```

We have to provide the user name and encrypted password for that user.

```bash
$ openssl passwd -6 # SHA512
Password:
Verifying - Password:
$6$04wstsomethinglong...
```

Then you enter the user username and encrypted password into the `userconf.txt`.

```txt
duck:$6$04wstsomethinglong...
```

Boot the pi and we can now SSH into it!

**In your router remember to reserve an address for your Pi as it will be your main DNS and DHCP server.**

## Hardening

Now we have the basic setup. Time for configuration/customization...

### Update OS and reboot

Manual step recommendation from offical guide - check if we have enough space beforehand.

`df -Ph`

To update packages:

```bash
# Update package list
sudo apt update

# Install them
sudo apt full-upgrade # -y  # if you dare or for automation # otherwise manually enter y to continue

# Remove unwanted packages
sudo apt autoremove # -y  # if you dare or for automation # otherwise manually enter y to continue

sudo reboot
```

To update firmware:

Note the warning message from the program...

```txt
#############################################################
WARNING: This update bumps to rpi-6.1.y linux tree
See: https://forums.raspberrypi.com/viewtopic.php?t=344246

'rpi-update' should only be used if there is a specific
reason to do so - for example, a request by a Raspberry Pi
engineer or if you want to help the testing effort
and are comfortable with restoring if there are regressions.

DO NOT use 'rpi-update' as part of a regular update process.
##############################################################
```

So we are not supposed to upgrade the firmware automatically...

```bash
sudo rpi-update
sudo reboot
```

### User password

If you were using insecure password at the beginning (with the not so convenient copy and paste of encrypted password), it is time to change it.

```bash
passwd

# Check related policies
$ chage -l $USER
Last password change:                              May 03, 2023
Password expires:                                  never
Password inactive:                                 never
Account expires:                                   never
Minimum number of days between password change:    0
Maximum number of days between password change:    99999
Number of days of warning before password expires: 7
```

I am not a fan of changing password regularly so I like the current setup. (And I suppose this is the only active user on my pi as well)

Just in case the user settings is something not the same as above... (Note: related to `/etc/login.defs`)

```bash
sudo chage -m 0 -M 99999 -E -1 -I -1 $USER

chage -l $USER # verify, should be same as above
```

### SSH setup - no root login, only SSH public key pair

Some reference <https://www.raspberrypi.com/documentation/computers/configuration.html#improving-ssh-security>

First step, generate ssh key pair in another machine.

```bash
ssh-keygen -t ed25519  # ed25519 is personal preference

# Update own ~/.ssh/config as well
vi ~/.ssh/config

Host my_pi
  Hostname your_pi_hostname
  User username
  Port 22
  IdentityFile ~/.ssh/private_key_file
```

Remember `ssh my_pi` won't work until we added the SSH key in the pi server.

We can try `ssh-copy-id` to add our public key to pi server. At this stage we need to enter the password (the last time)

`ssh-copy-id -i ~/.ssh/public_key_location.pub USERNAME@pi-hostname`

Update the SSH config in pi server.

Note that Pi is using Debian and some default conifgs are slightly different from what you know...

From `man 5 sshd_config`:

```txt
    Note that the Debian openssh-server package sets several options as standard in /etc/ssh/sshd_config which are not the default in sshd(8):

           o   Include /etc/ssh/sshd_config.d/*.conf
           o   ChallengeResponseAuthentication no
           o   X11Forwarding yes
           o   PrintMotd no
           o   AcceptEnv LANG LC_*
           o   Subsystem sftp /usr/lib/openssh/sftp-server
           o   UsePAM yes

     /etc/ssh/sshd_config.d/*.conf files are included at the start of the configuration file, so op-
     tions set there will override those in /etc/ssh/sshd_config.
```

```bash
$ grep -v '#' /etc/ssh/sshd_config | grep .

Include /etc/ssh/sshd_config.d/*.conf
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem sftp /usr/lib/openssh/sftp-server
```

Acutal setup:

```bash
# Setup to accept public key if not done via ssh-copy-id
mkdir -p ~/.ssh/
touch ~/.ssh/authorized_keys

# Fix permission
chmod 700 $HOME/.ssh/
chmod 600 $HOME/.ssh/authorized_keys

vi ~/.ssh/authorized_keys  # add the content of public key here
```

Then hardening SSH server config - /etc/ssh/sshd_config

```bash
sudo vi /etc/ssh/sshd_config
# Comment out the line AcceptEnv LANG LC_*
# This would fix the problem of "man: can't set the locale; make sure $LC_* and $LANG are correct"
# From https://forums.raspberrypi.com//viewtopic.php?f=50&t=11870

# Follow the Include settings and use a customized .conf file
sudo vi /etc/ssh/sshd_config.d/pi_custom.conf  # want a 644 config file, same as sshd_config

cat /etc/ssh/sshd_config.d/pi_custom.conf

PasswordAuthentication no
PermitRootLogin no
UsePAM no
```

Not really sure about the PAM option... PAM seems won't do much on this home pi server setup.

Finally, test if our public key pair works and password would fail or not.

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password username@pi-hostname  # should fail - Permission denied (publickey).

ssh my_pi  # the alias in ~/.ssh/config.
# Should get into the server directly
```

### Other minor stuff

#### Setup vi

```bash
echo 'set number' >> ~/.vimrc
echo 'set nocompatible' >> ~/.vimrc
echo 'set backspace=2' >> ~/.vimrc
```

#### Update hostname

```bash
sudo hostnamectl set-hostname (new hostname)

vi /etc/hosts  # the old hostname still remains in host file. Update it

# Now sudo whatever command should not show error like
# sudo: unable to resolve host (hostname): Name or service not known
```
