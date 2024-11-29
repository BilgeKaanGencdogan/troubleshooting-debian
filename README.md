# TROUBLESHOOTING OF APACHE WEB SERVER

- [USERS AND PASSWORDS](#users-and-passwords)
- [NETWORK ADAPTER CONFIGURATION OF WM](#network-adapter-configuration-of-wm)
- [LOG IN TO VM THAT LOG-IN CREDENTIAL IS NOT GIVEN](#log-in-to-vm-that-log-in-credential-is-not-given)
- [SWITCHING EMERGENCY MODE TO NORMAL MODE](#switching-emergency-mode-to-normal-mode)
- [PATH VARIABLE IS MISSING](#path-variable-is-missing)
- [ADDING A USER FOR SSH CONNECTIONG](#adding-a-user-for-ssh-connection)
- [DNS AND DNS RESOLVER CONFIGURATION](#dns-and-dns-resolver-configuration)
- [APACHE SERVER TROUBLESHOOTING](#apache-server-troubleshooting)

### USERS AND PASSWORDS
```
User: bilgekaan
Password: 0123456789
User : root
Password: 123456

```

### NETWORK ADAPTER CONFIGURATION OF WM

  Before starting my VM, I want to look at the network settings of the VM. Two network adapters are configured for that VM, One is Host-only Adapter, the other one is NAT. First of all, I removed Host-only because Host-only is used like a private network with VM to VM, VM to Host, but it does not allow the outside  network to communicate with the VM, since we want an outside network to communicate with our web server, I removed that adapter. When It comes to NAT, outside the internet and the host can communicate through port with that VM, we do not want to deal with that either. The best option for us is the Bridged-adapter. When the VM is configured with that adapter, it behaves like a real computer on the same network. Outside the network can access directly, vice versa.

### LOG IN TO VM THAT LOG-IN CREDENTIAL IS NOT GIVEN
After starting the VM, some Spanish text welcomes you, it is possible that the environment variable related to language is set to Spanish. We’ll deal with that in a later section 3, and also before that Spanish text, it says you are in the emergency mode. We’ll deal with that too in a later section 3. 

There is a way to log-in that system without entering the password by modifying the GRUB Boot Parameters. Let’us walk through the process;


1. First, the VM should be powered-off, then started. 
2. When the screen starts to show itself, ESC should be pressed to enter GRUB Menu.
3. The default boot entry should be selected, then E to edit it.
4. Line that starts with `linux /vmlinuz..` should be found to edit. This boot parameter is responsible to set-up the Linux kernel and is a critical part of the boot process in a Linux system. Because we do not have any credentials to log-in, we can configure that line to change the behavior of the operating system during startup to bypass the log-in, however this process is temporary, if we want it to be permanent, we should configure the `/etc/default/grub` file.
5. We should append `init=/bin/bash` at the end of that line to boot the system into a minimal shell as root, bypassing the login prompt. After CTRL+X to save that configuration and start VM
6. After the VM starts, we should set a new password with passwd command, 
but it throws an __error__ which is;
```
   “passwd: authentication token manipulation error, passwd: password unchanged”.
``` 
  This error typically occurs when trying to change passwords in a read-only filesystem, meaning we should go back to that parameter and change the read-only to read-write mode. We changed `linux /vmlinuz.. ro quiet` to `linux /vmlinuz.. rw init=/bin/bash`.
  
7. After setting the password, we should reboot the system to log-in. However when we type reboot and press enter, it throws an error which is;
```
   “System has not been booted with systemd as init system (PID 1). Can’t operate. Failed to connect bus: Host is down. Failed to talk init deamon” 
```
This error occurs because we are in a minimal shell environment without systemd (since we booted with `init=/bin/bash`). In this situation, we can't use the regular reboot command. However, there are two ways to reboot it: 1)We can force the system to reboot with `sync; echo b > /proc/sysrq-trigger` command. After that, the system reboots and we can use that password to login. 2) with the `systemctl reboot` command as suggested in the warning line that says, you are in the emergency mode.

### SWITCHING EMERGENCY MODE TO NORMAL MODE
At the section, we discussed that our system is in emergency mode. And the system itself suggests that we should run that command `journalctl -xb` to look at the system logs for the reason. And there is an __error__ which is;
```
“The unit systemd-timesyncd.service has entered the “failed” state with result ‘result’ resources  . Failed to start network time synchronization. Subject: A start job for unit systemd-timesyncd.service has finished with a failure. ”
```
We should get more detail with the log of that service by looking at command `journalctl -u systemd-timesyncd.service`, and the output is;
```
“systemd-timesyncd.service: Failed to start 'run' task: No space left on device. systemd-timesyncd.service: Failed with result 'resources'.failed to start network time synchronization. systemd-timesyncd.service: Service has no hold-off time (RestartSec=0). scheduling restart. systemd-timesyncd.service: Scheduled restart job, restart counter is at 2. stopped Network Time Synchronization”
```
It looks like "No space left on device" means your disk space is full, which is causing the time synchronization service (and possibly other services) to fail.  We should look at disk usage of the system with `df -h`. The output looks like we have enough space, however we can check one more thing which is inode. We can monitor that with the `df -i` command. According to the output, in the disk on which root sits ,only 3 inode is free, that’s why the error occurs. We should either delete some files or add a partition to the VM to transfer some of the files in root to that partition. Let’s add a disk and make partition configuration. 
1. First of all, we need to power-off the VM to add a disk.
2. Go to storage -> and add hard disks on Controller:SATA -> add VirtualBox Disk Image 
3. Then Start the VM.
4. After start, we should check that If the newly added disk is added successfully with `fdisk -l`, and disk name with `/dev/sdb` is added successfully.
5. Then we should create a partition with that disk with `fdisk <disk_name>`.
6. n # New partition, p # Primary partition, 1  # Partition number ,# Press Enter for default first sector ,# Press Enter for default last sector (or specify size like +10G), w # Write changes and exit.
7. We successfully create a partition, we can again check `fdisk -l` and see the partition.
8. Then we should format that partition with the `mkfs.ext4 <partition name>`.
9. Create a mount point and mount with the commands which are `mkdir /mnt/newdisk` and `mount /dev/sdb1 /mnt/newdisk`.
10. We can check if our mounting processes are successful or not with `df -h` again.
11. Finally we can mv files under /home/ to that partition to free inodes. But this home directory is too large to move all files, we should adapt different approach, let’s mv part by part with the `sudo find /home -name '*9*' -exec mv {} /mnt/newdisk/ \;` command by the way, we should do that 10 times because 99k files are there. That’s why '*9*' part will change every run('*9*','*8*'....etc). We did.
12. We successfully free lots of inode to operate.
13. Let’s start the systemd-timesyncd.service again. And We successfully started the service.
14. Let’s reboot the system to see if we get out of emergency mode after we successfully start `systemd-timesyncd.service`.
15. We did not get out of the emergency mode, let look again but this time, just look at the errors with `journalctl -xb -p err`, and also there is one more error and it says;
```
“console-setup.service failed”. Let’s run journalctl -u console-setup.service to see more detail. And the log says “/usr/bin/setupcon: 870: /usr/bin/setupcon: cannot open /tmp/tpmkd.A2ns2n: No such file. Console-setup.service: Main process exited, code: exited, status=1/FAILURE. Console-setup.service: Failed with result ‘exit-code’. Failed to start Set console font and keymap”.
```
16. I fixed that issue with sudo dpkg-reconfigure console-setup to reconfigure the console-setup package to fix any broken configuration and  restart the service with `sudo systemctl restart console-setup.service`. It works.
17. Let’s reboot the system again and run `journalctl -xb -p err` to check if there is another error.
18. There is another error but I think it is not that important that’s why I will skip `“[drm:vmw_host_log [vmwgfx] ] *ERROR* Failed to send host log message”`.
19. I am still in `emergency mode`. When I researched it, I found that When our system boots into emergency mode, it means a critical issue prevents the system from mounting essential filesystems or starting necessary services. We should troubleshoot with `journalctl -xb` but it is too long that is why we can filter with `journalctl -xb | grep -i “..”`. So I am going to write respectively `mount, fsck, corrupt, file,failed, error` instead of .. in that command to find the reason.  When I try to do `journalctl -xb | grep -i "mount"`, log says that;
```
“mount: /dev/sda9: no es un dispositivo de bloques(It is not a block device)”
```
This can be reason for us not to get out of emergency mode because this error means that the system is attempting to mount the partition `/dev/sda9`, but it is failing because the device either doesn't exist, isn't properly recognized, or is not in the correct format for mounting. We should first verify that whether `/dev/sda9` actually exists and is a valid block device. We should first run the `lsblk` command to list all the block devices and their mount points. Check if /dev/sda9 is listed here. No it is not there. And also we can ceheck it with one more command which is sudo fdisk -l to list all disks and partitions to ensure that `/dev/sda9` is shown. No it is not there either. Then finally, since it is not added to either disk or block devices, we should delete its entry from `/etc/fstab` for our system to mount essential filesystems or start necessary services. Then reboot the system to switch to normal mode . Okay, we successfully switched our system to normal mode.

### PATH VARIABLE IS MISSING
After I successfully switched my system from emergency mode to normal mode, then rebooted it, basic commands like ls, printenv are not working. This is because the `PATH` variable is missing even in the root directory. We should add it to `~/.profile` or `~/.bashrc` file but also nano does not work. The only choice to overcome this issue, we first should set that variable temporarily with `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` then add that related files. I have added export `export PATH=$PATH:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` to both `~/.bashrc` and `~/.profile` then reboot the system to see if it works or not. It works. Finally, I am going to change the language to English by adding export `LANGUAGE=en_US.UTF-8` to `~/.bashrc` file.

### ADDING A USER FOR SSH CONNECTION
I have created this user but even though its home directory is `/home/kaan`, I could not navigate it with cd /home/kaan. In my host system, if I created the user, its home directory is automatically created under /home/ and also while creation of that user, I should set its password immediately, but in that distro, after I added new user, its home directory is not created automatically and also I should run extra `sudo passwd bilgekaan` to set its password. I do not know why. And I created `/home/bilgekaan` directory then try to `su - bilgekaan`, it throws an error which is;
```
“su: atencian: no se puede cambiar el directorio a /home/kaan : no existe el fichero o el directorio (The home directory for the user kaan does not exist)
-sh: 1: id: not found 
-sh: 18: id: [illegal number: 
$”
```

The shell is unable to find the id command (possibly due to the absence of necessary binaries in the PATH). We should create the home directory manually with `sudo mkdir /home/kaan` and set the correct ownership: The home directory should be owned by the user kaan and have the correct permissions with `sudo chown kaan:kaan /home/kaan` and `sudo chmod 755 /home/kaan`. `The -sh: id: [illegal number]` __error__ suggests there might be an issue with the shell or the environment for the user. If `/bin/bash` or another valid shell isn’t set, or if there’s a configuration issue, it might prevent the user from properly switching. In `/etc/passwd`, we should verify that the user's shell is set to a valid shell like `/bin/bash`, but it is `/bin/sh`, that’s why we should configure that with `sudo usermod -s /bin/bash kaan`. We did that, let’s try it with `su - kaan again`. It worked.

Even though, I write PATH variable to `~/.bashrc`, `~/.profile` and `/etc/environment` under root user, new user’s PATH variable is not setup, I thought that I configured it globally but no, let’s write it `/home/kaan/.bashrc` and `/home/kaan/.profile` from root. It worked.

When I try to `sudo apt update`, it throws an __error__;
```
“kaan is not the sudoers file, incident will be reported”.
```
That means that the kaan user does not have the required permissions to use the sudo command. To fix this, you need to add the kaan user to the sudo group. We first need to login as root with `su -`. Then run `usermod -aG sudo kaan` to add kaan sudo group.

SSH is not working because my host pc and my vm are not in the same network, my host machine’s ip is 139.179.203.225/32 and my vm 1.1.1.1/24 with static ip. My vm should have ip like 139.179.203.x/24 to communicate with my host, so that’s why I should configure `/etc/network/interfaces` to make ssh connection. I set my vm to 139.179.203.15. Let’s reboot then make an ssh connection.

Even though I have repeatedly entered the correct password for my VM user, permission denied __error__ is thrown. Ssh is active in vm also sshd. Last option is to configure the SSH server configuration file by setting `PasswordAuthentication yes` on the VM `/etc/ssh/sshd_config` to make sure that password authentication is allowed, then `systemctl reload ssh` to take this change effect to the ssh service.

### DNS AND DNS RESOLVER CONFIGURATION
I have deleted ssh service because when I configure `/etc/ssh/sshd_sonfig`, I accidentally delete the all content, then I made dummy decision to delete ssh then install ssh from scratch rather than writing that file content from scratch manually, but I could not install ssh because my vm is closed for communication to internet. It seems there is an issue about the DNS resolver. DNS resolver is a service or tool that your computer uses to resolve domain names (like google.com) into IP addresses (like 142.250.190.14) so it can communicate over the internet.When we try to visit a website, our computer needs to know the IP address associated with the domain name we've entered. That's where a DNS resolver comes in—it queries DNS servers to get the right IP. On Linux systems, the DNS resolver settings are usually stored in `/etc/resolv.conf`. This file tells your computer which DNS servers to use.Why we are facing that because our `/etc/resolv.conf` shows 127.0.0.1 which is the `loopback address` (also called localhost). It points to our own machine. It indicates that our system is trying to use a DNS server running locally (on the same machine) to resolve domain names. If the system is set to use 127.0.0.1 but there is no DNS service running on the machine (for example, no DNS resolver like systemd-resolved or dnsmasq is active), the machine won't be able to resolve domain names. Solution is to point to an `external DNS server` (like Google's 8.8.8.8 or Cloudflare’s 1.1.1.1) to enable proper internet connectivity.But I can not write anything to `/etc/resolv.conf` file even under the root folder, because this file is immutable meaning that it is not changeable no matter what. We can check if the file is immutable by running `lsattr <filename>`. If the output is `“----i—----------- filename”`, it indicates that the file is immutable. So this is the case for my situation. Why this file is immutable is because some administrators set `/etc/resolv.conf` as immutable to stop dynamic tools like `NetworkManager` or `systemd-resolved` from overwriting it. We can remove that immutability by running `sudo chattr -i /etc/resolv.conf` for a moment to setting it to public DNS servers like Google's 8.8.8.8 and 8.8.4.4 or default gateway IP address which also serves as an DNS SERVER so that our system can resolve domain names and successfully connect to the repositories to fetch updates, and then reapply it to the same file `sudo chattr +i /etc/resolv.conf`. Then restart the related service to take these changes to effect by running `sudo systemctl restart systemd-networkd`. Okay, the connection is not yet established. Because our `ip route` is not configured correctly. Ip route’s purpose is used to view, add, or modify routing table entries on a Linux system. Routing tables determine how network traffic is forwarded between devices within a local network or to remote destinations (like the internet). The purpose of the ip route command is to manage this routing information. There are two entries in that table which are `139.179.203.0/24 dev enp0s3 proto kernel scope link src 139.179.203.15` and `169.254.0.0/16 dev enp0s3 scope link metric 1000`. First entry specifies that any traffic destined for the 139.179.203.0/24 network (the local network) will be sent directly to `enp0s3`, and the source address used for the traffic will be 139.179.203.15. Second entries’ purpose is to allow communication with other devices that have `169.254.x.x` IP addresses (link-local), but only within the local network segment. If no other route is available (i.e., no valid routes to a more specific network), this route could be used, though it has a high metric. One is missing which is the default route. That route, often called the `gateway`. It tells the system where to send packets that are not destined for any of the local subnets directly connected to the machine, which is the internet. Let’s add it to that with `sudo ip route add default via 139.179.203.1 dev enp0s3` command. After adding that we should also add this gateway to the `/etc/network/interfaces` with the format like gateway 139.179.203.1. However, we have not finished yet because we should change the content of /etc/resolv.conf to nameserver <default gateway IP address>. Then, we can restart the systemd-resolved to take these changes to effect. But sudo apt update is working, meaning we succedded. I have adjust `/etc/network/interfaces` like above;

`/etc/network/interfaces`
```
#BEGIN ANSIBLE MANAGED BLOCK
iface enp0s3 inet static
      address 139.179.203.179
      netmask 255.255.255.0
      network 139.179.203.0
      gateway 139.179.203.1
                dns-nameservers 8.8.8.8 8.8.4.4
#END ANSIBLE MANAGED BLOCK
```
And my `/etc/resolv.conf` is changed,
Before it was;
              `Nameserver 127.0.0.1`
Right now;
          ```
              domain dormnet.bilkent.edu.tr
              domain dormnet.bilkent.edu.tr
		          Search bcc.bilkent.edu.tr
              Nameserver 139.179.30.24
		          Nameserver 139.179.10.13
          ```
However, I want to point out that I did not configure the `/etc/resolv.conf` manually, at the time when dealing with the `/etc/network/interfaces`, I changed `enp0s3` type from static to dynamic with dhcp. The DHCP client (e.g., dhclient) fetches the IP address, gateway, and DNS information from the DHCP server. And then change it from dynamic, then configure enp0s3 interface manually, `/etc/resolv.conf` stays the same. Then I tried to do `sudo rm /etc/resolv.conf` and `sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf` for `systemd-networkd service` to handle all dns server and dns resolution staff. We create a symlink above to show that whatever in that `/resolve/resolve.conf` will automatically be used in `/etc/resolv`.

When I restart the machine, I cannot connect to the internet again, ping does not work. I have changed the network interface from static to dynamic, meaning that all  dns or ip related configuration is done by the dhcp server. I give up trying static configuration. However, my machine is unable to resolve domain names for the repositories `(deb.debian.org and security.debian.org)` when I do sudo apt update. However when I add nameserver 8.8.8.8 and nameserver 1.1.1.1, my host machine can not connect to vm with ssh. Interesting.

`sudo iptables -L -v -n `
```
sudo iptables -A OUTPUT -p udp --dport 53 -m conntrack --ctstate NEW -j ACCEPT 
sudo iptables -A OUTPUT -p tcp --dport 53 -m conntrack --ctstate NEW -j ACCEPT
```

I have done those and still Did not figure it out why I am getting that __errors__ when I do `sudo apt update`;
```
Err:1 http://security.debian.org/debian-security buster/updates InRelease
  Temporary failure resolving 'security.debian.org'
Err:2 http://deb.debian.org/debian buster InRelease
  Temporary failure resolving 'deb.debian.org'
Err:3 http://deb.debian.org/debian buster-updates InRelease
  Temporary failure resolving 'deb.debian.org'
Reading package lists... Done
Building dependency tree       
Reading state information... Done
All packages are up to date.
W: Failed to fetch http://deb.debian.org/debian/dists/buster/InRelease  Temporary failure resolving 'deb.debian.org'
W: Failed to fetch http://security.debian.org/debian-security/dists/buster/updates/InRelease  Temporary failure resolving 'security.debian.org'
W: Failed to fetch http://deb.debian.org/debian/dists/buster-updates/InRelease  Temporary failure resolving 'deb.debian.org'
W: Some index files failed to download. They have been ignored, or old ones used instead.
```

### APACHE SERVER TROUBLESHOOTING
```
root@debian:~# sudo systemctl list-units --type=service
  		UNIT                               LOAD    ACTIVE SUB     DESCRIPTION                                         
● apache2.service                        loaded  failed failed  The Apache HTTP Server  
```
```
root@debian:~# journalctl -u apache2.service  
		          Nov 23 21:43:39 debian apachectl[547]: AH00526: Syntax error on line 5 of /etc/apache2/ports.conf:
              Nov 23 21:43:39 debian apachectl[547]: Invalid command 'Lisen', perhaps misspelled or defined by a module not in
              Nov 23 21:43:39 debian apachectl[547]: Action 'start' failed.
```
I have corrected this spelling mistake but I could not start the apache2.servce because another error shows up.
```
root@debian:~# journalctl -u apache2.service 
               Nov 23 21:57:07 debian apachectl[1040]: AH00526: Syntax error on line 7 of /etc/apache2/conf-enabled/charset.conf:
               Nov 23 21:57:07 debian apachectl[1040]: Invalid command 'AdDefaultCharset', perhaps misspelled or defined by a module not incl
```

Then restart it, It worked. Lastly when I try to connect with vm ip, the site is redirected it `https://10.0.3.15/`, we do not want to do that, first this ip is NAT ADAPTER’s IP which we closed, even it is open we did not port forwarding configuration, so we would not had used it. Let’s change that configuration by commenting or deleting related line in the `/etc/apache2/sites-available/000-default.conf` , and then we should do `systemctl reload apache2.service` for this change to take effect. And then, we connect to apache2 server by typing bridged adapter IP. It works.
PS; in bridged adapter, Ip may change, so we should check the vm ip every time we want to connect to apache.














