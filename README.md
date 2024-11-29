# TROUBLESHOOTING OF APACHE WEB SERVER

- [NETWORK ADAPTER CONFIGURATION OF WM](#network-adapter-configuration-of-wm)
- [LOG IN TO VM THAT LOG-IN CREDENTIAL IS NOT GIVEN](#log-in-to-vm-that-log-in-credential-is-not-given)

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





