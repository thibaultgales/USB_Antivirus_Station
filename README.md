# USB_Decontamination_Station

Thibault Galès 2023/10

A hardened Debian machine using open source software to automatically detect viruses and threats on USB flash drives.

## Information and recommendations

I used a Debian 12 ISO

## Warning: Security and Trust

The configuration of my Debian 12 machine has been specially designed to guarantee the integrity of your USB flash drive analyses and the security of your data. In order to maintain an optimal level of trust, specific precautions have been taken to ensure that this machine remains a reliable and secure source.

### SSH deactivation:

The SSH protocol, although useful for remote management, has been deactivated after initial configuration. This limits any attempt at unauthorized access to the machine through this channel.

### Restricted Automatic User Login:

The user automatically logged in at machine start-up has limited rights. This ensures that no commands or actions can be carried out without your explicit consent.

### Command Blocking:

Any possibility of entering commands directly on the machine has been disabled. This prevents any unauthorized manipulation of the system, thus preserving the integrity of the analyses carried out.

### Warning:

**This machine is a strategic point for any potential attacker. By corrupting this system, an attacker could not only prevent the decontamination of each USB key, but also contaminate all the keys. This makes it a major entry point for any attempt at intrusion or malicious manipulation.**

**The aim of these measures is to make the machine autonomous and secure in its updates and USB key scans. However, it's crucial to stress that even with these precautions, no computer system is totally invulnerable. It is therefore imperative to take additional measures to physically secure access to this machine, and to always keep its software and security protocols up to date.**

**Your trust in this machine is fundamental. Make sure you always keep the system up to date, carefully monitor its use, and apply security best practices to ensure continued protection of your data and analyses.**

## 1 - Installation procedure for Debian machine

`Boot on the Debian 12 iso`

`Install`

`English`

`Hostname : DS`

`Domain name :`

`Root password : root`

`name account : usb`

`password : usb`

`Partitioning method : Guided - use entire disk`

`Partitioning scheme : All file in one partition`

`Choose software to install : just "ssh" & "Standard system utilities"`

`Install the GRUB boot loader to your primary drive ? yes /dev/sd`


## 2 - Configuration

`$| su root`

`#| ip a`

 Retrieve the machine's IP and connect via SSH to set it up.

`#| su root`

`#| apt-get update`

`#| apt-get install clamav`

`#| systemctl stop clamav-freshclam`

`#| freshclam`

`#| systemctl start clamav-freshclam`

USB error detection :

`#| cd /home/usb`

`#| nano check_error.sh`

	#!/bin/bash
	if [ -e "/dev/sdc" ]; then
    		systemctl reboot
	fi

Save & exit

Modify Cron to run automatic antivirus updates :

`#| crontab -e`

Added these two lines at the end of the file :

	0 0 * * * systemctl stop clamav-freshclam && freshclam
	*/5 * * * * /home/usb/check_error.sh >/dev/null 2>&1

Save & exit

Image script installation

`#| apt install jp2a`

Copy ".png" images from the "img" folder on the Debian to the /home/usb/ directory

You need to copy "HOME.png SCAN.png NOTHING.png VIRUS.png EXE.png" to the "/home/usb" folder

`#| cd /home/usb`

`#| ls`

You must have: check_error.sh EXE.png HOME.png NOTHING.png SCAN.png VIRUS.png

Create a txt file of each output of the jp2a command for each image :

`#| jp2a /home/usb/VIRUS.png > VIRUS.txt`

`#| jp2a /home/usb/HOME.png > HOME.txt`

`#| jp2a /home/usb/SCAN.png > SCAN.txt`

`#| jp2a /home/usb/NOTHING.png > NOTHING.txt`

`#| jp2a /home/usb/EXE.png > EXE.txt`

## 3 - Scripting

`#| nano check_usb.sh`

```bash
#!/bin/bash
cat /home/usb/SCAN.txt | write usb
#Initialize virus variable to 0
exe=0
virus=0
#Check if a USB key is inserted
if [ -b "/dev/sdb1" ]; then
    #Analyzes the number of partitions on /dev/sdb
    partition_count=$(lsblk -n -o NAME /dev/sdb | wc -l)
    #Checks if partitions are present
    if [[ $partition_count -eq 0 ]]; then
        exit 1
    fi
    #Browse each score
    for ((i=1; i<=partition_count; i++)); do
        partition="/dev/sdb$i"
        mount "$partition" /mnt/usb
        #Checks if files have been found
        clamscan -r --log=/tmp/clamscan.txt /mnt/usb/ | write usb
        if grep -qE "\.(exe|dll|bat|sh|pl|py|rb|c|cpp|java|jar|class)" /tmp/clamscan.txt; then
                exe=1
        fi
        if grep -q "Infected files: [1-9]" /tmp/clamscan.txt; then
            virus=1
        fi
        rm /tmp/clamscan.txt
        umount /mnt/usb
    done
    #if no infected files found
    if [[ $virus -eq 0 ]]; then
        if [[ $exe -eq 1 ]]; then
            cat /home/usb/EXE.txt | write usb
            sleep 15
            echo "Attention! The session will end in 15 seconds..." | write usb
            sleep 15
            echo -e "\033[2J\033[H" > /dev/tty1
            sleep 5
            cat /home/usb/HOME.txt | write usb
            exit 1
        fi
        sleep 2
        cat /home/usb/NOTHING.txt | write usb
        sleep 5
        echo "Attention! The session will end in 15 seconds..." | write usb
        sleep 15
        echo -e "\033[2J\033[H" > /dev/tty1
        sleep 5
        cat /home/usb/HOME.txt | write usb
        exit 1
    else
        cat /home/usb/VIRUS.txt | write usb
        exit 1
    fi
else
	echo "No USB key is inserted." | write usb
	sleep 5
	echo "Attention! The session will end in 15 seconds..." | write usb
	sleep 15
	echo -e "\033[2J\033[H" > /dev/tty1
	sleep 5
	cat /home/usb/HOME.txt | write usb
	exit 1
fi
```

Save and make it executable :

`#| chmod +x check_usb.sh`

Create an udev rule to automatically trigger the script when the sb key is inserted. Create an udev rule file using the following command :

`#| nano /etc/udev/rules.d/99-usb.rules`

Add the following line to the udev rules file :

	ACTION=="add", KERNEL=="sd[b-z][0-9]", RUN+="/home/usb/check_usb.sh"

save & exit

`#| systemctl restart systemd-udevd.service`

`#| mkdir /mnt/usb`

WARNING after that the file /home/usb/check_usb.sh will no longer be editable :

`#| chattr +i /home/usb/check_usb.sh`

`#| lsattr /home/usb/check_usb.sh`

WARNING: From here on, it's the big leagues, so log in as root and don't make a single mistake.
WARNING: this part will make the machine immutable, impossible to modify after this:

`#| PATH="/sbin:$PATH"`

`#| command -v usermod`

`#| usermod -s /bin/rbash usb`

`#| cp /etc/skel/.bashrc /home/usb/`

`#| nano /home/usb/.bashrc`

	clear
	cat /home/usb/HOME.txt
	unset PATH
	unset SHELL
	unset TERM

Connection party:
This part is used to connect the user (here: usb) automatically when the physical machine is launched.

`#|nano /etc/systemd/system/getty.target.wants/getty@tty1.service`

    ExecStart=-/sbin/agetty --autologin usb --noclear %I $TERM
    
`#|systemctl daemon-reload`

After a short reboot, the USB decontamination machine is ready.



