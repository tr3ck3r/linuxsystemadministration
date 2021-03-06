# linuxsystemadministration
While there are many introductions to Linux, this guide provides working lab activities to help practice Linux system administration commands.  To practice common Linux system administrative functions, tasks are prescribed and then the fastest way(s) to accomplish the goal is available for the selected Linux distribution which is Ubuntu 16.04 LTS server.   

# Table of Contents

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Setup Process
## Caveats
While this guide could make use of sudo for all super privileged commands (such as those in /sbin) to avoid the use of the root account; efficiency of time makes it optimal to use the root account for almost all functions.  In a few cases, an unprivileged account will be used such as testing file and user privileges.  It is with heavy heart that the use of root account herein is documented; but, the goal is speed and this shortcut turns into a tremendous time saver.  Do not pursue this approach with production servers with multiple administrators where you should be using sudo to provide audit trail and visudo rather than these shortcuts.  Root can be accessed via `sudo -i` or `sudo su - root` from vagrant account.
## Platform generic focus
Take careful note of the version of Ubuntu as this presumes 16.04.  These instructions are platform specific to Ubuntu 16.04 to avoid extra commands to master.  When possible, generic concepts and methods encompassing multiple distributions are covered such as iptables instead of ufw for example.  
## Setup notes
There are a multitude of possible ways to setup your learning environment.  However, given the overall benefit of standardizing your configuration to match this guide so that it works every time from scratch and is identical, vagrant is the preferred solution.  A libvirt hypervisor or manual configuration of a group of virtualbox/vm ware hosts can work but provide some complexity to keep standardized.

Port forwarding will be used on localhost so that a ssh client can be used always to the same port to access the Linux boxes.

Setup should several additional physical volume on lb40 for RAID, swap space, quota, and other file system testing.

Make vim default editor: ```update-alternatives --config editor```

To create the working folders to be consistent in /tmp; use ```mkdir /work/{1..9}.{1..9}```
### Layout of drives
Virtualbox restricts us to 15 hard drives.  The disk layout will be as follows for lb40 and is reflected in Vagrantfile supplied to avoid confusion and drive re-use between exercises as well as partitioning for exercises:
1. /dev/sda VagrantRoot
2. /dev/sdb ubuntu cloud fs
3. /dev/sdc 1000MB will be for partition practicing and other non-persistent across boot stuff
4. /dev/sdd 50MB normal_swap
5. /dev/sde 50MB encrypted_swap
6. /dev/sdf 50MB encrypted_fs
7. /dev/sdg 50MB ext4 /tmp
8. /dev/sdh 50MB ext4 /quota 
9. /dev/sdi 50MB mdadm RAID
10. /dev/sdj 50MB mdadm RAID
11. /dev/sdk 50MB mdadm RAID
12. /dev/sdl 50MB mdadm RAID
13. /dev/sdm 50MB LVM
14. /dev/sdn 50MB LVM
15. /dev/sdo 50MB LVM

### Initial setup without vagrant
```
sudo apt -y update; sudo apt -y upgrade ; echo 'datasource_list: [ None ]' | sudo -s tee /etc/cloud/cloud.cfg.d/90_dpkg.cfg; sudo apt -y purge cloud-init; sudo rm -rf /etc/cloud/; sudo rm -rf /var/lib/cloud/; sudo apt -y install tmux 
```
We are going to setup a group of computers all running Ubuntu 16.04 Server within a NAT Network of 10.20.30.0/24 with primary machine being 10.20.30.40 and other clients being .50 and up not to exceed .99. Static IPs will be used to make consistent for these examples to work especially for networking so consider starting there.  Hostnames will be lbXX with XX being the last octet such as 10.20.30.40 being lb40 and the domain name search will be linux.local.

## Caching Updates and Packages
As you will be running more than one identical Ubuntu box; it makes some sense to maintain local LAN copies of packages you are going to be using to speed up installation.  There is no one size fits all; merely different ways of approaching the problem.
Sources: [Cache APT packages with Squid proxy](http://www.rushiagr.com/blog/2015/06/05/cache-apt-packages-with-squid-proxy/)
### squid-deb-proxy
This is the easiest solution and easiest to maintain and install.  It requires no configuration on the client side other than package and no configuration on the server side after installing packages.   A bad configuration or down traditional squid server will cause updates to still work after connection to the squid-deb-proxy fails. Therefore, this style will be used by default with Vagrantfile and configuration scripts.  lb40 will automatically install this functionality and this server should be the most heavily used anyways.  The nice thing is the port will work off 8000 so it won't conflict with later configuration of a squid server on 3128 on lb40.  Default mirrors work, I prefer to use the fastest US mirror I can find and tweak appropriately in the configuration files run on on devices.

_Install on the server_
```
# sudo apt-get install -y squid-deb-proxy squid-deb-proxy-client;  sudo systemctl start squid-deb-proxy; systemctl enable squid-deb-proxy
```
_Install on clients_
```
# sudo apt-get install squid-deb-proxy-client
```
### Squid
One of the things we are practicing anyways is setting up a squid caching server.  So, if one was to skip down to configuring squid right away this could be the best learning approach to make updates apply faster locally off your LAN using lb40 as the caching server and can be up and running quickly (apt-mirror requires you to download the entire mirror which can take a long time).  However, after testing all methodologies, squid-deb-proxy appears to be better run off lb40.  
#### Squid Server
```
refresh_pattern -i \.(tar.gz|tar|deb|rpm|bz2|gz|xml)$ 129600 90% 129600 override-expire ignore-no-cache ignore-no-store ignore-private refresh-ims
```
#### Squid Clients
```
# cat /etc/apt/apt.conf.d/01proxy
Acquire::http::Proxy "http://10.20.30.40:8080/";
Acquire::https::Proxy "http://10.20.30.40:8080/";
```
The first download will be at WAN speed; but, subsequent downloads should be much faster. 
More tweaks can be found at https://wiki.ubuntu.com/SquidDebProxy
### apt-mirror
An alternative idea would be a local apt-mirror repository which will never slow you down in that all files will be local; but, it will take a significant amount of time and space to populate your repository.   While the fastest automated solution from a loading of any package once mirror is populated; the space makes this a suspect method as mirror will take in the ballpark of 129.0 GB heavily depending on configuration (use deb-amd64 to not also download i386).

## Files to support Vagrant
Install vagrant and put the supplied Vagrantfile, lb40.txt and standardize.txt file in the same folder.  Bring up the VMs (after they finish downloading) with vagrant up.   
As we are using vagrant, the default username and password to accomplish tasks for these activities is vagrant/vagrant.  Feel free to use fewer hosts; but, since there is an abundancy of memory and processing power it seems logical to crank up may VMs with 1 GB of RAM.

<details><summary>Vagrantfile</summary>

```
Vagrant.configure("2") do |config|
 config.vm.provision :shell, path: "standardize.txt", run: 'always'
#  This is the best way to set all machines to use the same proxy server; but, requires plugin and proxy server to be already setup
# consider using squid-deb-proxy instead as it is automatically optimized for this function
# if Vagrant.has_plugin?("vagrant-proxyconf")
#    config.apt_proxy.http     = "http://10.20.30.40:8080/"
#    config.apt_proxy.https    = "http://10.20.30.40:8080/"
# end
 config.vm.define "lb40" do |lb40|
    lb40.vm.provision :shell, path: "lb40.txt", run: 'always'
    lb40.vm.box = "lb40"
	lb40.vm.network "private_network", ip: "10.20.30.40", virtualbox__intnet: true, :adapter => 2
    lb40.vm.network "forwarded_port", guest: 22, host: 55540
	lb40.vm.network "forwarded_port", guest: 3128, host: 3128
	lb40.vm.network "forwarded_port", guest: 8080, host: 8080
	lb40.vm.network "forwarded_port", guest: 44480, host: 80
	lb40.vm.network "forwarded_port", guest: 44443, host: 443
	lb40.vm.network "forwarded_port", guest: 44421, host: 21
	lb40.vm.hostname = "lb40"
	lb40.vm.box = "ubuntu/xenial64"
	lb40.vm.provider :virtualbox do |lb40|
        lb40.name = "lb40"
		lb40.memory = "4000"
		# 3 drive
		unless File.exist?('./3Disk.vdi')
            lb40.customize ['createhd', '--filename', './3Disk.vdi', '--variant', 'Fixed', '--size', 1 * 1024]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 2, '--device', 0, '--type', 'hdd', '--medium', './3Disk.vdi']
        # 4 drive
		unless File.exist?('./4Disk.vdi')
             lb40.customize ['createhd', '--filename', './4Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 3, '--device', 0, '--type', 'hdd', '--medium', './4Disk.vdi']
		# 5 drive
		unless File.exist?('./5Disk.vdi')
             lb40.customize ['createhd', '--filename', './5Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 4, '--device', 0, '--type', 'hdd', '--medium', './5Disk.vdi']
		# 6 drive
		unless File.exist?('./6Disk.vdi')
             lb40.customize ['createhd', '--filename', './6Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 5, '--device', 0, '--type', 'hdd', '--medium', './6Disk.vdi']
		# 7 drive
		unless File.exist?('./7Disk.vdi')
             lb40.customize ['createhd', '--filename', './7Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 6, '--device', 0, '--type', 'hdd', '--medium', './7Disk.vdi']
		# drive 8
		unless File.exist?('./8Disk.vdi')
             lb40.customize ['createhd', '--filename', './8Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 7, '--device', 0, '--type', 'hdd', '--medium', './8Disk.vdi']
		# drive 9
		unless File.exist?('./9Disk.vdi')
             lb40.customize ['createhd', '--filename', './9Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 8, '--device', 0, '--type', 'hdd', '--medium', './9Disk.vdi']
		# drive 10
		unless File.exist?('./10Disk.vdi')
             lb40.customize ['createhd', '--filename', './10Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 9, '--device', 0, '--type', 'hdd', '--medium', './10Disk.vdi']
		# drive 11
		unless File.exist?('./11Disk.vdi')
             lb40.customize ['createhd', '--filename', './11Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 10, '--device', 0, '--type', 'hdd', '--medium', './11Disk.vdi']
		# drive 12
		unless File.exist?('./12Disk.vdi')
             lb40.customize ['createhd', '--filename', './12Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 11, '--device', 0, '--type', 'hdd', '--medium', './12Disk.vdi']
		# drive 13
		unless File.exist?('./13Disk.vdi')
             lb40.customize ['createhd', '--filename', './13Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 12, '--device', 0, '--type', 'hdd', '--medium', './13Disk.vdi']
		# drive 14
		unless File.exist?('./14Disk.vdi')
             lb40.customize ['createhd', '--filename', './14Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 13, '--device', 0, '--type', 'hdd', '--medium', './14Disk.vdi']
		# drive 15
		unless File.exist?('./15Disk.vdi')
             lb40.customize ['createhd', '--filename', './15Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 14, '--device', 0, '--type', 'hdd', '--medium', './15Disk.vdi']
		# drive 16
		unless File.exist?('./16Disk.vdi')
             lb40.customize ['createhd', '--filename', './16Disk.vdi', '--variant', 'Fixed', '--size', 50]
        end
        lb40.customize ['storageattach', :id,  '--storagectl', 'SCSI', '--port', 15, '--device', 0, '--type', 'hdd', '--medium', './16Disk.vdi']
	end
 end
 config.vm.define "lb50" do |lb50|
    lb50.vm.box = "lb50"
    lb50.vm.network "forwarded_port", guest: 22, host: 55550
	lb50.vm.network "private_network", ip: "10.20.30.50", virtualbox__intnet: true, :adapter => 2
	lb50.vm.hostname = "lb50"
	lb50.vm.box = "ubuntu/xenial64"
	lb50.vm.provider :virtualbox do |lb50|
        lb50.name = "lb50"
	end
 end
  config.vm.define "lb60" do |lb60|
    lb60.vm.box = "lb60"
    lb60.vm.network "forwarded_port", guest: 22, host: 55560
	lb60.vm.network "private_network", ip: "10.20.30.60", virtualbox__intnet: true, :adapter => 2
	lb60.vm.hostname = "lb60"
	lb60.vm.box = "ubuntu/xenial64"
	lb60.vm.provider :virtualbox do |lb60|
        lb60.name = "lb60"
	end
 end
   config.vm.define "lb90" do |lb90|
    lb90.vm.box = "iptables"
    lb90.vm.network "forwarded_port", guest: 22, host: 55590
	lb90.vm.network "private_network", ip: "10.20.30.90", virtualbox__intnet: true, :adapter => 2
	lb90.vm.hostname = "lb90"
	lb90.vm.box = "ubuntu/xenial64"
	lb90.vm.provider :virtualbox do |lb90|
        lb90.name = "lb90"
    end
 end
end
```

</details>

<details><summary>standardize.txt</summary>

```
# make sure to grab packages locally if possible
sudo apt-get install -y squid-deb-proxy-client
# enable vagrant ssh remotely with passwordsed
sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config  
service ssh restart
# disable cloud for a faster bootpin
sudo apt-get -y purge cloud-init
sudo mv /etc/cloud/ ~/; sudo mv /var/lib/cloud/ ~/cloud-lib
sudo apt remove -y open-iscsi
mkdir /work/{1..9}.{1..9}
sudo useradd -m fred
sudo useradd -m sally
sudo useradd -m george
# apt-get update; apt-get upgrade -y
# sudo apt-get -y install apache2 squid tmux ntp bind9 dnsutils quota quotatool fish
# sudo chsh -s /usr/bin/fish
# sudo apt-get autoremove
```

</details>

<details><summary>lb40.txt</summary>

```
# Let's make sure we have fastest way to run updates by storing them on lb40
sudo apt-get install squid-deb-proxy -y
sudo systemctl start squid-deb-proxy
sudo systemctl enable squid-deb-proxy
sudo apt-get install -y squid-deb-proxy-client
# enable vagrant ssh remotely with password
sed -i 's/ChallengeResponseAuthentication no/ChallengeResponseAuthentication yes/g' /etc/ssh/sshd_config  
service ssh restart
# disable cloud for a faster bootpin
sudo apt-get -y purge cloud-init
sudo mv /etc/cloud/ ~/; sudo mv /var/lib/cloud/ ~/cloud-lib
sudo apt remove -y open-iscsi
mkdir /work/{1..9}.{1..9}
sudo useradd -m fred
sudo useradd -m sally
sudo useradd -m george
# apt-get update; apt-get upgrade -y
# sudo apt-get -y install apache2 squid tmux ntp bind9 dnsutils quota quotatool fish
# sudo chsh -s /usr/bin/fish
# sudo apt-get autoremove
```

</details>

## tmux and fish and apropos and documentation
Before doing any work within a vm at the command line, the first thing which should always be run is tmux (or screen if you are so inclined).  This will allow you to look up documentation without opening a new session or stopping your existing command.  Some vendors run tmux as the first thing they do when they log into our machines on shared sessions.  The most basic thing to do is open two terminals after tmux is loaded is to use `CTRL-B` then `" `.  To switch between them, use the `CTRL-B` then `up` or `down` arrow key.  There is a lot more a person could learn about tmux; but, this is enough to get several terminals up and running as well as navigate quickly between them.

fish can also help to avoid having to remember command syntax.

If you just can't remember how to do something, try apropos "idea" to see what the man pages say.

Extremely helpful guides include:
1. /usr/share/doc/sed/sedfaq.txt

### What to do when network is down to box or proxy servers block you: put the Linux documentation to work for you:
Way too often the Internet isn't reachable because you have messed something up or the company proxy servers are just painful to detail with; so, how do you find information when you can't reach Internet or this guide?

Extract all the compressed gzip files in /usr/share
```
# find /usr/share -name "*.gz" -type f -exec ls -l "{}" \; 
# find /usr/share -name "*.gz" -type f -exec gunzip -k "{}" \;
# grep "route add.*" -R /usr/share/
/usr/share/man/fr/man8/route.8:.B route add \-net 192.57.66.0 netmask 255.255.255.0 gw ipx4
```
Sources: [tmux for noobs](https://eoinoc.net/tmux-for-noobs)
## Syntax in this guide
All shell commands will be proceeded by a # indicating root account in use or $ indicating another account is in use that isn't the root account.  Sometimes the username will be displayed first as in this case the account fred:
```
fred $ ping -c 1 8.8.8.8
ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=84.6 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 84.683/84.683/84.683/0.000 ms
fred $
```
If a specific box is being used, the prompt may also have the hostname prepended:
```
lb60 # 
```
To avoid helping out oneself too much by looking at the solution rather than working through problem from memory and documentation, the github mechanism of hidding blocks of text will be used as such:
<details><summary>Description</summary>

```
Example block of text
```

</details>

This maybe inconsistent in earlier incarnations of the guide; but, basically the example solutions should indicate enough information to understand what to do if you get stumped without being a cut and paste mechanism.
# Basic Linux Commands
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module1-essentialcommand/
## Login to Both Local and Remote Text Consoles
You may not have a GUI to use when first logging into Ubuntu on a VM or physical box and may be presented with a black screen with white text.  This is known as the terminal mode or “multi-user” mode and is desirable for servers.  There are multiple consoles accessible using the key combination of ALT+F# with the number being a specific console (example: ALT-F2 switches to access console 2 called tty2).  Enter your username and password to login.  Sometime a specific tty is dedicated to logging and this tty won’t allow you to login.

Above all else, get used to using the CLI and not the Linux GUI (Xwindows).  GUI is never to be used on production servers so learn vi and not gnotepad+.

<details><summary>System is set to a graphical environment.  Set to multi-user only on Ubuntu. </summary>

```
# systemctl get-default
graphical.target
# systemctl enable multi-user.target
# systemctl start multi-user.target
# systemctl set-default multi-user.target
```

</details>

## Finding Files
<details><summary>What commands can help out in a jam:</summary>

```
# locate ping
# whatis ping
```

</details>

<details><summary>Put the binary path for ping into file named /work/1.2/path 
Figure out where the program 'ping' is stored:</summary>

To begin this exercise; make sure you are in /work/1.2 folder which should have been created during setup.

```
# which ping > /work/1.2/path
# cat /work/1.2/path 
/bin/ping
```

</details>

<details>
<summary>Find file named Debian-PAM-MiniPolicy in /usr/share and place path into /work/1.2/filepath.</summary>

```
# find /usr/share/ -name Debian-PAM-MiniPolicy -print > /work/1.2/filepath
# cat /work/1.2/filepath
/usr/share/doc/libpam0g/Debian-PAM-MiniPolicy
```

</details>

<details><summary>Place long list of files displaying permissions in /etc which are read only for all users to a file named /work/85.21/444:</summary>

```
# find /etc/ -perm 444 -exec ls -al "{}" \; > /work/85.21/444
# cat /work/85.21/444
-r--r--r-- 1 root root 33 Nov 27 14:17 /etc/machine-id
-r--r--r-- 1 root root 2415 Nov 28 03:23 /etc/apt/apt.conf.d/01autoremove-kernels
```
</details>

<details><summary>Find all files with ownership of ntp in /var storing result in /work/42.54/ntp:</summary>

```
# find /var -type f -user ntp -print > /work/42.54/ntp
# cat /work/42.54/ntp
/var/lib/ntp/ntp.drift
```

</details>

<details><summary>Find all directories in /var with group of syslog in /work/43.11/syslogdir :</summary>

```
# find /var -type d -group syslog -print > /work/43.11/syslogdir
# cat /work/43.11/syslogdir
/var/log
```

</details>

<details><summary>Put path of first five directories in /usr/share/ into /work/43.41/dirs:  </summary>

```
# find /usr/share/* -type d -print | head -5 >  /work/43.41/dirs
# cat /work/43.41/dirs
/usr/share/aclocal
/usr/share/adduser
/usr/share/apache2
/usr/share/apache2/build
/usr/share/apache2/default-site
```

</details>

<details><summary>Put path of first five files in /usr/share/ into /work/87.56/filepaths.txt:  </summary>

```
# find /usr/share/* -type f -print | head -5 > /work/87.56/filepaths.txt
# cat /work/87.56/filepaths.txt
/usr/share/aclocal/dovecot.m4
/usr/share/aclocal/dovecot-pigeonhole.m4
/usr/share/adduser/adduser.conf
/usr/share/apache2/build/envvars-std
/usr/share/apache2/apache2-maintscript-helper
```

</details>

<details><summary>In etc, find all files and then output their type to /work/87.56/etctypes and take a look at the top of the resulting file:</summary>

```
# find /etc -type f -exec file '{}' > /work/87.56/etctypes \;
# head /work/87.56/etctypes
/etc/environment: ASCII text
/etc/apache2/magic: magic text file for file(1) cmd, ASCII text
/etc/apache2/sites-available/default-ssl.conf: ASCII text
/etc/apache2/sites-available/000-default.conf: ASCII text
/etc/apache2/conf-available/localized-error-pages.conf: ASCII text
/etc/apache2/conf-available/security.conf: ASCII text
/etc/apache2/conf-available/javascript-common.conf: ASCII text
/etc/apache2/conf-available/charset.conf: ASCII text
/etc/apache2/conf-available/serve-cgi-bin.conf: ASCII text
/etc/apache2/conf-available/other-vhosts-access-log.conf: ASCII text
```

</details>

<details><summary>Copy everything in /etc/ssh/ to /work/52.13/.  Find all files in /work/52.13/ and ask if it is ok to delete them one by one but answer no to each delete other than moduli: </summary>

```
# cp  /etc/ssh/* /work/52.13/
find /work/52.13/ -type f -ok rm '{}' \;
< rm ... /work/52.13/ssh_host_ed25519_key > ? n
< rm ... /work/52.13/ssh_import_id > ? n
< rm ... /work/52.13/ssh_host_dsa_key > ? n
< rm ... /work/52.13/ssh_host_rsa_key > ? n
< rm ... /work/52.13/sshd_config > ? n
< rm ... /work/52.13/ssh_host_rsa_key.pub > ? n
< rm ... /work/52.13/moduli > ? y
< rm ... /work/52.13/ssh_config > ? n
< rm ... /work/52.13/ssh_host_ecdsa_key.pub > ? n
< rm ... /work/52.13/ssh_host_ed25519_key.pub > ? n
< rm ... /work/52.13/ssh_host_ecdsa_key > ? n
< rm ... /work/52.13/ssh_host_dsa_key.pub > ? n
```

</details>

<details><summary>Find all files sized greater than 2k in /etc/pam.d/ and write results to  /work/16.13/bigpamfiles.txt.</summary>

```
# find /etc/pam.d/ -size +2k -print > /work/16.13/bigpamfiles.txt
# cat /work/16.13/bigpamfiles.txt
/etc/pam.d/
/etc/pam.d/login
/etc/pam.d/sshd
/etc/pam.d/su
```

</details>

<details><summary>First, make inside /work/62.21/ make dummy file named 1999 with the modify time during day of January 1st, 1999 and a current file with all timestamps for today named 2018; </summary>

```
# touch -t 199901010101 1999 /work/62.21/1999
# touch /work/62.21/2018
# stat /work/62.21/1999 | grep Modify
Modify: 1999-01-01 01:01:00.000000000 +0000
```
</details>


<details><summary>Now,  find only the 1999 file using find command and the 2018 file using find.</summary>

```
# find /work/62.21/ -mtime +6480 -print
./1999
# find /work/62.21/ -mtime 0 -type f
./2018
```

</details>

<details><summary>Sometimes it is helpful to find out the full path for a file:</summary>

```
# readlink -f /work/1.2/freds
/work/1.2/freds
```

</details>


## Evaluate and Update Text Files
### touch
<details><summary>It can be very useful to create a blank file.  Touch allows for this with bash:
Create files named file1-file9 in /work/53.34/:</summary>

```
# touch /work/53.34/file{1..9}
# ls
file1  file2  file3  file4  file5  file6  file7  file8  file9
```

</details>

<details><summary>Make a file named 1978 which was created that year: </summary>

```
# touch -t 197801010101 1978
# ls -al 1978
-rw-r--r-- 1 root root 0 Jan  1  1978 1978
```

</details>

### textfiles: vi(m)
There are many text editors.  For the serious administrator, traditional vi or one of the many newer versions which are vi improved (vim) is the one to at least be able to use with some proficiency as it is installed on systems when others are not.  When you are using a system without nano and you have no Internet connection, a knowledge of vi is critical. Unfortunately, vi requires more than a passing commitment to make it worth the effort.  It will pay huge dividends in the long run to learn vi/vim; but, requires some commitment to master. Use vimtutor to master program.

Sources: [How to Install and Use vi/vim as a Full Text Editor](https://www.tecmint.com/vi-editor-usage/)

<details><summary>Compare /usr/share/common-licenses/GPL-3 to /usr/share/common-licenses/GPL-2 placing result into /work/43.21/GPLdiff.txt: </summary>

```
# diff /usr/share/common-licenses/GPL-3 /usr/share/common-licenses/GPL-2c2
<                        Version 3, 29 June 2007
---
>                        Version 2, June 1991
4c4,5
<  Copyright (C) 2007 Free Software Foundation, Inc. <http://fsf.org/>
---
>  Copyright (C) 1989, 1991 Free Software Foundation, Inc.,
>  51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA
10,19c11,18
```

</details>

<details><summary>Make two directories using bash script `mkdir -p /work/32.42/{A_dir,B_dir}; touch /work/32.42/A_dir/{3..25}.txt; touch /work/32.42/B_dir/{5..27}.txt`   The folders have different files and sometimes similar files.  Produce a report to /work/32.42/B of files only that exist only in /work/32.42/B_dir and not at all in /work/32.42/A_dir .</summary>

```
# diff -q /work/32.42/A_dir/ /work/32.42/B_dir/ | grep B_ > /work/32.42/B
# cat /work/32.42/B
Only in B_dir/: 26.txt
Only in B_dir/: 27.txt
``` 

</details>

### textfiles: patch
### textfiles: cut
<details><summary>Let’s see how we can just display the third and forth columns for the first line in /etc/passwd:</summary>

```
# head -1 /etc/passwd
root:x:0:0:root:/root:/bin/bash
# head -1 /etc/passwd | cut -d : -f 3,4
0:0
```

</details>


<details><summary>textfiles: vimdiff</summary>

Vimdiff starts Vim on two (or three or four) files.  Each file gets its own window. The differences between the files are highlighted.  This is a nice way to inspect changes and to move changes from one version to another version of the same file.
</details>

<details><summary>textfiles: pr</summary></details>

<details><summary>textfiles: cat</summary></details>

<details><summary>textfiles: tac</summary></details>

<details><summary>textfiles: tail</summary></details>

<details><summary>Capitalize everything in /etc/passwd and send to /work/1.4/CAPITALIZED.txt</summary>

```
cat /etc/passwd | tr '[:lower:]' '[:upper:]' > /work/1.4/CAPITALIZED.txt
```

</details>

<details><summary>textfiles: sed</summary>

Don't study harder than you need to, **/usr/share/doc/sed/sedfaq.txt** contains many examples.

<details><summary>To capitalize all "a" in /etc/passwd placing result in /work/47.43/A</summary>

```
# sed 's/a/A/g' /etc/passwd > /work/47.43/A
# tail -3 /work/47.43/A
sAlly:x:1005:1005::/home/sAlly:
wAlt:x:1006:1006::/home/wAlt:
Abc123:x:1007:1008::/home/Abc123:
```

</details>

<details><summary>Escape special character </summary>

backslash in sed: 
```
# sed -e "s/'/\"/g" printer2.txt
```

</details>

<details><summary>Copy /etc/apt/sources.list to /work/43.21/.  Substitute all occurrences of "http:" with "ftp:" .</summary>

```
# cp /etc/apt/sources.list /work/43.21/
# sudo sed -i.backup 's/http:/ftp:/g' /etc/apt/sources.list
# grep -v ^# /etc/apt/sources.list | tail -2
deb-src ftp://la-mirrors.evowise.com/ubuntu xenial-security multiverse
```

</details>

<details><summary>Binary files: ls -l, cmp, md5sum</summary></details>

<details><summary>What type of file is /boot/vmlinuz*</summary>

```
# file /boot/vmlinuz-4.15.0-38-generic
/boot/vmlinuz-4.15.0-38-generic: Linux kernel x86 boot executable bzImage, version 4.15.0-38-generic (buildd@lcy01-amd64-023) #41~16.04.1-Ubuntu S, RO-rootFS, swap_dev 0x7, Normal VGA
```

</details>

<details><summary>What is /etc/passwd</summary>

```
# file /etc/passwd
/etc/passwd: ASCII text
ls -ld <filename>
```

</details>


<details><summary>Put the type of file out of long listing for /etc/rc2.d/S02ssh and /etc/passwd into /work/32.45/files?<summary>

```
# ls -al /etc/rc2.d/S02ssh > /work/32.45/files
# ls -ld /etc/passwd >> /work/32.45/files
# cat /work/32.45/files
lrwxrwxrwx 1 root root 13 Nov 27 14:24 /etc/rc2.d/S02ssh -> ../init.d/ssh
-rw-r--r-- 1 root root 2431 Dec  1 02:35 /etc/passwd
```

</details>

##	Redirecting shell and command output 
To get help: `man sh` or `man bash` and look for redirect.
This is as good of a time as any to introduce cat.  cat - concatenate files and print on the standard output. It dumps to standard output (stdout) input files.
Also, echo is useful to take text enclosed in quotes to either send to stdout or with redirect to a file.
Write output to file clobbering any existing content:
`# echo "hello" > file`
Append output to file `# command >> file`
Send output from one command to another (indefinitely stackable): `# cat file | grep A`

<details><summary>Write stdout to /work/76.43/stdout and errors /work/76.43/stderr: </summary>

```
# echo "echo this goes to stdout
echo and this goes to stderr >&2" > /work/65.43/command
# chmod 755 /work/65.43/command
# /work/65.43/command > /work/76.43/stdout 2> /work/76.43/stderr
# cat /work/76.43/stdout
this goes to stdout
# cat /work/76.43/stderr
and this goes to stderr
```
Source: https://www.linuxjournal.com/content/working-stdin-and-stdout

</details>



<details><summary>Write both stdout and errors to same file named /work/65.43/both for command : </summary>

```
# /work/65.43/command &>  /work/65.43/both
```
Source: https://www.linuxjournal.com/content/working-stdin-and-stdout

</details>

To read a file back into a command: `# wc -l < syslog.pdf`
##	Regular expression syntax
<details><summary>Find all the lines in /usr/share/common-licenses/GPL-3 that start with the word "The" but case insensitive and put result in /work/7.4/the </summary>

```
grep -i "^The " /usr/share/common-licenses/GPL-3 > /work/7.4/the
```

</details>


<details><summary>Find all the lines that end with the word "you" case sensitive within /usr/share/common-licenses/GPL-3 and put result in /work/32.43/you </summary>


```
# grep " you$" /usr/share/common-licenses/GPL-3 > /work/32.43/you
# head -4 /work/32.43/you
price.  Our General Public Licenses are designed to make sure that you
them if you wish), that you receive source code or can get it if you
  To protect your rights, we need to prevent others from denying you
of having them make modifications exclusively for you, or provide you
```

</details>



<details><summary>Find all the lines that start with the word "gnu" ignoring case within /usr/share/common-licenses/GPL-3 and put result in /work/54.24/gnu </summary>


```
# grep -i "^GNU" /usr/share/common-licenses/GPL-3 > /work/54.24/gnu
# cat /work/54.24/gnu
GNU General Public License for most of our software; it applies also to
GNU General Public License, you may choose any version ever published
```

</details>



<details><summary>Find all the lines in /usr/share/common-licenses/GPL-3 that contain word "the" followed by the word "you"  and put result in /work/43.24/theyou </summary>

```
# grep -w "the .* you" /usr/share/common-licenses/GPL-3 > /work/43.24/theyou
# head -5 /work/43.24/theyou
them if you wish), that you receive source code or can get it if you
these rights or asking you to surrender the rights.  Therefore, you have
or can get the source code.  And you must show them these terms so they
(1) assert copyright on the software, and (2) offer you this License
of having them make modifications exclusively for you, or provide you
```

</details>

##	Archive or compress files and directories
<details><summary>Place a gzipped archive in /work/3.2/ named "cron.tar.gz" which contains /etc/cron.d/ but without absolute paths</summary>

```
# cd /etc/cron.d/
# tar cfgv /work/3.2/cron.tar.gz *
```

</details>

<details><summary>Place a XZ compressed tar archive in /work/3.2/ named "ssh_config.tar.xz" which contains all of /etc/ssh but without folders from root.</summary>

```
# cd /etc/ssh/
# tar cfJ /work/3.2/ssh_config.tar.xz *

```

</details>

<details><summary>Extract  /usr/share/doc/apg/php.tar.gz into a new directory named /work/3.2/php</summary>

```
# mkdir -p /work/3.2/php
# tar xfvz /usr/share/doc/apg/php.tar.gz -C /work/3.2/php
# ls /work/3.2/php
index.php  lang/  README  themes/
```

</details>

Sources: [tecmint - Archiving Files/Directories](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
##	Affecting directories and files with mv, cp, and others
<details><summary>Think of all commands related to this topic</summary>

```
echo 'text' > file
cat
touch
vi(m)
nano
mkdir
mkdir -p
rm
cp
mv
```

</details>

<details><summary>Create directories dir1-dir8 in /work/37.24</summary>

```
mkdir /work/37.24/dir{1..8}
# ls /work/37.24/
dir1  dir2  dir3  dir4  dir5  dir6  dir7  dir8
```

</details>


<details><summary>Create dummy files file1-99 in /work/1.64 and then delete file40 through file52</summary>

```
# touch /work/1.64/file{1..99}
# rm /work/1.64/file{40..49}
# ls /work/1.64/file{40..49}
# ls /work/1.64/file* # result truncated to avoid filling this space
```

</details>


<details><summary>Copy /etc/debian_version to /work/34.21/ </summary>

```
# cp /etc/debian_version /work/34.21/
# ls /work/34.21/debi*
/work/34.21/debian_version
```

</details>

<details><summary>Place a copy of /etc/debian_version in directory /work/33.23 with the name of ubuntu </summary>

```
# cp /etc/debian_version /work/33.23/ubuntu
# ls /work/33.23/ubuntu
/work/33.23/debian_version
```

</details>


<details><summary>Make directories in /work/1.8/dirtest named 1,2,3,4, and 5 with one command; then delete them all with one command:</summary>

```
# mkdir -p /work/1.8/dirtest/{1..5}
# find /work/1.8/dirtest/ -type d -print
/work/1.8/dirtest/
/work/1.8/dirtest/4
/work/1.8/dirtest/2
/work/1.8/dirtest/3
/work/1.8/dirtest/1
/work/1.8/dirtest/5
# rm -rf /work/1.8/dirtest/
```

</details>

<details><summary>Ask for confirmation before deleting directory /work/1.8/ then cancel.</summary>

```
rm -Rfi /work/1.8/
```

</details>

<details><summary>Make the following path: /work/1.8/4/5/1/3/2/1/4/1/3/2/1/4/5/6/4 </summary>

```
mkdir -p /work/1.8/4/5/1/3/2/1/4/1/3/2/1/4/5/6/4
```

</details>

<details><summary>Change your working directory (the directory you are currently in) to /work/1.8 and print working directory:</summary>

```
# cd /work/1.8
# pwd
/work/1.8
```

</details>

##	Shortcuts to files and directories with hard and soft links
<details><summary>Create a hard link from /etc/apg.conf to /work/2.4/apglink: </summary>

```
ln /etc/apg.conf /work/2.4/apglink
```

</details>

<details><summary>Create a symbolic link between /etc/appstream.conf and /work/5.6/appstream.txt</summary>

```
# ln -s /etc/appstream.conf /work/5.6/appstream.txt
# ls -al /work/5.6/appstream.txt
lrwxrwxrwx 1 root root 19 Nov 10 01:47 /work/5.6/appstream.txt -> /etc/appstream.conf
```

</details>

<details><summary>Figure out where the symbolic link /etc/vtrgb points to and put the result in /work/3.6/vtrgb</summary>

```
# ls -al /etc/vtrgb > /work/3.6/vtrgb
# cat /work/3.6/vtrgb
lrwxrwxrwx 1 root root 19 Nov 10 01:47 /work/5.6/appstream.txt -> /etc/appstream.conf
```

</details>

<details><summary>/etc/ has a bunch of symbolic links.  Provide a report of them all in /work/3.6/etclinks.txt</summary>

```
# find /etc -type l  -exec ls -l {} \; > /work/3.6/etclinks.txt
# head /work/3.6/etclinks.txt
lrwxrwxrwx 1 root root 24 Nov 12 01:43 /etc/mysql/my.cnf -> /etc/alternatives/my.cnf
lrwxrwxrwx 1 root root 17 Nov 14 00:55 /etc/rc5.d/S02anacron -> ../init.d/anacron
lrwxrwxrwx 1 root root 17 Nov 15 20:42 /etc/rc5.d/S04dovecot -> ../init.d/dovecot
lrwxrwxrwx 1 root root 22 Nov 15 20:42 /etc/rc5.d/S04cpufrequtils -> ../init.d/cpufrequtils
```

</details>

<details><summary>Since there is an absence of hard links on default install, we need to create a bunch in /work/98.31.  Provide a report of them all after creating them in /work/98.31/etclinks.txt</summary>

```
# touch /work/98.31/{1..5}.txt; mkdir hard
# for filename in /work/98.31/*.txt ; do iterate=`shuf -i 1-99 -n 1`;  for i in $iterate; do rand=`shuf -i 1-99 -n 1`; ln "$filename" "$filename"."$rand" ;  done;    done
# find /etc -type l  -exec ls -l {} \; > /work/3.6/etclinks.txt
# head /work/3.6/etclinks.txt
lrwxrwxrwx 1 root root 24 Nov 12 01:43 /etc/mysql/my.cnf -> /etc/alternatives/my.cnf
lrwxrwxrwx 1 root root 17 Nov 14 00:55 /etc/rc5.d/S02anacron -> ../init.d/anacron
lrwxrwxrwx 1 root root 17 Nov 15 20:42 /etc/rc5.d/S04dovecot -> ../init.d/dovecot
lrwxrwxrwx 1 root root 22 Nov 15 20:42 /etc/rc5.d/S04cpufrequtils -> ../init.d/cpufrequtils
```

</details>

##	Standard file permissions
<details><summary>Create empty files in /work/8.2 with following names and permissions:
readonly r--r--r--, readwriteowner rw-------, and readwritegroup ---rw---- </summary>

```
# cd /work/8.2
# touch readonlyowner readwriteowner readwritegroup
# chmod 600 readwriteowner
# chmod 400 readonlyowner
# chmod 060 readwritegroup
# ls -al
total 8
drwxr-xr-x  2 root root 4096 Nov 10 02:20 .
drwxrwxrwt 91 root root 4096 Nov 10 02:20 ..
-r--------  1 root root    0 Nov 10 02:20 readonlyowner
----rw----  1 root root    0 Nov 10 02:19 readwritegroup
-rw-------  1 root root    0 Nov 10 02:19 readwriteowner
```

</details>

<details><summary>Set fred's account to allow group members to read only any of his new files he creates for current session.</summary>

``` 
fred $ umask -p
0002
fred $ umask -S
u=rwx,g=rwx,o=rx
# umask 0037
# apt-get install -y pam_umask
```

</details>

Other stuff:
```
ls -l file.txt
chown o+w file.txt
chown 755 file; chown 644 file; chown 700 file, chown 600 file; chown -r
Create a directory named /private. Use an acl to only allow access (rwx) to tom to the private directory.
umask
```

Sources: [tecmint - File Permissions & Attributes](https://www.tecmint.com/manage-users-and-groups-in-linux/)
## Documentation on systems
```
# man ls #### Truncated but is simple system manual
# info mount #### the links at bottom work in info
# whatis -w mount*
mount (8)            - mount a filesystem
mount.fuse (8)       - format and options for the fuse file systems
mount.lowntfs-3g (8) - Third Generation Read/Write NTFS Driver
mount.ntfs (8)       - Third Generation Read/Write NTFS Driver
mount.ntfs-3g (8)    - Third Generation Read/Write NTFS Driver
mount_namespaces (7) - overview of Linux mount namespaces
mountpoint (1)       - see if a directory or file is a mountpoint
# whereis mount
mount: /bin/mount /sbin/mount.lowntfs-3g /sbin/mount.vmhgfs /sbin/mount.ntfs /sbin/mount.ntfs-3g /sbin/mount.fuse /usr/share/man/man8/mount.8.gz

# ls -l /usr/share/doc/gzip-1.5/
```

<details><summary>Using quotatool command; get help.</summary>

```
# quotatool -h
  quotatool version 1.4.12
  Copyright (c) 1999-2012 Mike Glover / Johan Ekenberg
  Distributed under the GNU General Public License
  http://quotatool.ekenberg.se

Usage: quotatool -u uid | -g gid options [...] filesystem
       quotatool -u | -g -i | -b  -t time filesystem
Options:
  -b      : set block limits
  -i      : set inode limits
  -q n    : set soft limit to n blocks/inodes
  -l n    : set hard limit to n blocks/inodes
  -t time : set global grace period to time
  -r      : restart grace period for uid or gid
  -R      : raise-only, never lower quotas for uid/gid
  -d      : dump quota info in machine readable format (see manpage)
  -h      : show this help
  -v      : be verbose (twice or thrice for debugging)
  -V      : show version
  -n      : do nothing (useful with -v)
```

</details>

Sources: [tecmint]( https://www.tecmint.com/explore-linux-installed-help-documentation-and-tools/)

##	Root account and root-like access
Documentation: /usr/share/doc/sudo/examples/sudoers
<details><summary>Change to root account</summary>

`sudo -i`

</details>

<details><summary>Change root password: </summary>

```
# passwd root
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
```

</details>

<details><summary>Allow sally to run all the root commands without entering her password:</summary>

```
root # sudo su - sally
sally $ sudo fdisk -l /dev/sdj
[sudo] password for sally:
sally is not in the sudoers file.  This incident will be reported.
sally $ exit
root # less /usr/share/doc/sudo/examples/sudoers
root # echo "sally ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/sally
sally $ sudo fdisk -l /dev/sdj
Disk /dev/sdj: 50 MiB, 52428800 bytes, 102400 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

</details>

<details><summary>Allow fred to become sally during her vacation but no other user (ethical issues aside):</summary>

```
root # grep "joe" /usr/share/doc/sudo/examples/sudoers    # joe may su only to operator
joe             ALL = /bin/su operator 
root # echo "fred ALL = /bin/su sally" >> /etc/sudoers.d/fred
root # sudo su - fred
fred $ su - sally
Password:
sally $ whoami
sally
```

</details>

<details><summary>Allow fred to activate edit passwd file and change users passwords:</summary>

```
root # swapoff -a
root # free | grep -i swap
Swap:             0           0           0
root # sudo su - fred
fred $ /sbin/swapon -a
swapon: cannot open /dev/sdd: Permission denied
swapon: cannot open /dev/mapper/swap_encrypt: Permission denied
fred $ exit
root # echo "fred ALL = VIPW " >> /etc/sudoers.d/fred
root # echo "Cmnd_Alias      VIPW = /usr/sbin/vipw, /usr/bin/passwd" >> /etc/sudoers.d/fred
root # sudo su - fred
fred $ $ sudo passwd sally
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
```

</details>

<details><summary>Recover root password if nobody knows the root password on a machine or a sudo account:</summary>

```
#### HOLD left shift DURING BIOS POST.
#### Select a kernel that says (recovery mode) 
### Wait until you get recovery menu, then select root
### At root prompt, remount root filesystem.
#### Check for rw
# mount | grep -w /
# mount -o remount,rw /
# passwd 
# reboot
```

</details>

<details><summary>Set walt, fred, and joe to all be part of a new group called "ops" and grant all ops users full control of system as if they were root</summary>

```
root # 
```
</details>

Sources: [tecmint - Enabling sudo Access on Accounts](https://www.tecmint.com/manage-users-and-groups-in-linux/)
#	Operation of Running Systems

##	Bootloader configuration
<details><summary>/etc/grub.d/40_custom is the correct place to edit many things and sometimes /etc/default/grub.
Make any text file changes for grub settings and auto add additional found operating systems like Windows with:</summary>

```
# update-grub
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-4.15.0-29-generic
Found initrd image: /boot/initrd.img-4.15.0-29-generic
Found memtest86+ image: /memtest86+.elf
Found memtest86+ image: /memtest86+.bin
Done
$
```

</details>

<details><summary>Change boot timeout to 10 seconds</summary>

```
# grep GRUB_TIMEOUT /etc/default/grub
GRUB_TIMEOUT=10
# update-grub
```

</details>

Sources: [tecmint](https://www.tecmint.com/configure-and-troubleshoot-grub-boot-loader-linux/), [Configure GRUB2 Boot Loader Settings In Ubuntu 16.04](https://www.ostechnix.com/configure-grub-2-boot-loader-settings-ubuntu-16-04/)

##	Rebooting and halting system properly
<details><summary>Just reboot immediately. </summary>

```
# reboot
```

</details>

<details><summary>Power off system (called halting it).</summary>

```
# shutdown -h now
```

</details>

<details><summary>Shutdown in 5 minutes with explanation to users who are logged in using write all (wall).</summary>

```
# shutdown -r +5 "I need to apply new kernel, back in a few." 
```

</details>

##	Set system to boot at a different runlevel
<details><summary>Check current runlevel with below indicating multi-user (no Xwindows) and previous unknown: </summary>

```
# runlevel
N 3
```

</details>

<details><summary>Find out names of runlevels:  </summary>

```
# info runlevel # result truncated
       Table 1. Mapping between runlevels and systemd targets
       ┌─────────┬───────────────────┐
       │Runlevel │ Target            │
       ├─────────┼───────────────────┤
       │2, 3, 4  │ multi-user.target │
       ├─────────┼───────────────────┤
       │5        │ graphical.target  │
       └─────────┴───────────────────┘
```

</details>

<details><summary>Enable GUI boot with one command (unlikely this is desired as focus is CLI):  </summary>

```
# systemctl {start,enable} graphical.target
```

</details>

<details><summary>Disable GUI boot with one command:  </summary>

```
# systemctl {start,enable} multi-user.target
```

</details>

##	System log files
The main system logs are located in /var/log/

/var/log/apache2 stores apache2 logs


<details><summary>Find all occurrences of sudo and su authentication and place a report in /work/41.15 in a file named attackers.txt</summary>

```
# grep "su|sudo" /var/log/auth.log > /work/41.15/attackers.txt
```

</details>

/var/log/syslog is the kernel boot information

Make use of grep on these to find what you are looking for in previous content or use tail -f <logfile> to watch events as they happen.

##	Running commands at a scheduled date and time
Commands at, cron, crontab, sleep
Files: /etc/crontab; /etc/cron.*
How do you tell BASH to take a 1 minute break before doing something? `sleep 1m`
Use crontab tool to list and set up scheduled tasks.
<details><summary>Let’s take a look at current scheduled tasks:</summary>

```
crontab -l
```

</details>

<details><summary>Edit crontab:</summary>

```
update-alternatives --config editor # Great time to set text editor to avoid nano
crontab -e
```

</details>

<details><summary>List a specific user’s scheduled jobs:</summary>

```
crontab -u lab -l
```

</details>

<details><summary>Run `df -h > /var/log/df.log` that daily at 10:15 checks disk space and writes it to /var/log/systemstatus.log</summary>

```
# apt-get install sysstat
# grep ENA /etc/default/sysstat
ENABLED="true"
# service sysstat restart
# crontab -l
15 22 * * * /usr/bin/sar -A > /var/log/systemstatus.log
```

</details>

<details><summary>Run command `df -h > /var/log/df.log` on at 21:50 every Friday.</summary>

``` 
# 
```

</details>

<details><summary>Run command `ping -c 1 8.8.8.8 > /var/log/networkstatus.log` during work hours of 8 AM to 5 PM (8-13) at 30 minutes after the hour on weekdays only (M-F).</summary>

``` 
# 
```

</details>

Sources: [tecmint](https://www.tecmint.com/11-cron-scheduling-task-examples-in-linux/ )

##	Evaluating results of scheduled commands
cron will log output which can be useful to investigate how a job performed.  The logs will inform you as to the failure or success of your task.
##	Apply software patches from Linux vendor
### Update packages from the network, a remote repository, or from the local file system
Install a specific DEB file given URL: `# apt-get -y install `
install web server: `apt-get install -y apache2`
dump the information for package apache to a file named /work/3.2/apacheinfo `apt-cache show apache2 > /work/3.2/apacheinfo`
Remove package named finger: `apt purge -y finger`
How many files are part of finger?
```
dpkg -L finger | wc -l
13
```
Copy all files that contain the word grub from /etc/grub.d into /work/4.7: 
```
cp `grep -il "grub" /etc/grub.d/*` .
for file in /etc/grub.d/*; do if grep -q "grub" "$file";then cp "$file"
```
Sources: [tecmint - Linux Package Management](https://www.tecmint.com/linux-package-management/), [tecmint - 25 apt-get Command Examples](https://www.tecmint.com/linux-package-management/)

### Produce and deliver reports on system use (processor, memory, disk, and network), outages, and user requests
##	Monitor Linux Processes Resource Usage
ps, top and htop
##	Set Kernel Runtime Parameters in Linux
First thing is to figure out what kernel parameters are of interest.  They are active in /proc/sys as files but you should think of this as a temperary file created at boot time.  While these files can be updated, they will not persist upon reboot. Here we will look at a few options related to MySQL optimization:
```
# sysctl -a # Too much text to include so truncated
# sysctl vm.swappiness
vm.swappiness = 1
# cat /proc/sys/vm/swappiness
1
```
Changing kernel runtime parameters temperarily until next boot such as decreasing swappiness for MySQL:
```
# echo 0 > /proc/sys/vm/swappiness
# cat /proc/sys/vm/swappiness # to verify
0
```
OR
```
# sysctrl -w vm.swappiness = 0
# sysctl vm.swappiness # to verify
0
```
Changing kernel runtime parameters which will remain in effect after reboot but you must apply it with 'sysctl -p' :
```
# vi /etc/sysctl.conf
vm.swappiness=0
sysctl -p
```
Sources: [tecmint](https://www.tecmint.com/change-modify-linux-kernel-runtime-parameters/)
##	Basic Shell Scripting
Create a very basic script to perform a series of commands in a set sequence.
Make a temporary file in /work/ and display name: `mktemp`
Make a temporary directory in /work/ and display name: `mktemp -d`

Create a bash script named “infiniteloop” in /work/6.7 that does the following:
1.  Continually writes the string "Hello, world!" to the folder /dev/null
2. Has no exit loop (that is, it will continue to run until told to stop    
3.  Give fred permissions to run infinite_loop
4.  Switch user to fred and have him run infiniteloop
5.  Switch back to root
6.  As root, locate the now running infiniteloop and forcefully kill it by PID
Firstly, man bash and then look for "Compound Commands" to avoid forcing yourself to memorize syntax.
Also, take a look at /usr/share/doc/util-linux/examples/getopt-parse.bash
```
# cat infiniteloop
while true; do echo "Hello, world!" > /dev/null ; done
# ps auxf | grep fred
# kill <PID>
```
Write a BASH script called 2MB.sh in the /work/6.3/ directory that creates 17 files of 1MB each using the fallocate command. The files should have the name of: file_N where N is a number from 1 to 17.
```
# SUM=1 ; 
> until [ $SUM = 18 ]  ;
> do
>        touch file_$SUM
>        fallocate -l 1M file_$SUM
>        ((SUM++));
> done
# ls -alh file* # to verify, output removed.
```
Simpler answer:
```
# for N in `seq 1 17`; do fallocate -l 1M file_$N; done
```

Sources: [tecmint - Learning Basic Shell Scripting](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
##	Manage system services with systemd
Check specific service: 
```
$ service apache2 status
```
Check all services: 
```
# service --status-all
```
Check status on, disable, and stop ufw:
```
# systemctl status ufw
# systemctl stop ufw
# systemctl disable ufw
```
Sources: [tecmint - Managing System Startup Process and Services)](https://www.tecmint.com/linux-boot-process-and-manage-services/)

###	Configure a service to run every time after boot
Enable network service dns to start at boot time.
```
systemctl enable bind
```
Disable network service dns to start at boot time.
```
systemctl disable bind
```
Sources: [tecmint - Configuring Services Automatic Start on Boot](https://www.tecmint.com/installing-network-services-and-configuring-services-at-system-boot/)

###	Start, stop, and check the status of network services
Start network service dns for current session.
```
systemctl start bind
```
Start network service dns for current session.
```
systemctl stop bind
```
Check status of network service dns.
```
systemctl status bind
```


##	Individual processes
Getting information about running processes on a system 
```
top (1)              - display Linux processes 
htop (1)             - interactive process viewer
ps (1)               - report a snapshot of the current processes.
```
Managing running processes
```
kill (1)             - send a signal to a process
nice - run a program with modified scheduling priority 
renice (1)           - alter priority of running processes 
```
Find the NICE value of your web server on 40.  Change NICE value of web server to -15.  Confirm value has changed.
```
# ps auxf -u root | grep apache | head -1
root      1511  0.0  0.5 260352 21924 ?        Ss   Nov08   0:03 /usr/sbin/apache2 -k start
# renice -15 1511
1511 (process ID) old priority 0, new priority -15
```
Find all processes which are being run by user fred: `ps -u fred`
Display all processes: `ps -ef`
Display all processes with one line per thread: `ps -eLf`
Display all processes on a system in BSD like syntax: `ps aux`
Show the current resource limits for user: `ulimit -a`
Two commands to set a process priority: `nice` and `renice`

<details><summary>How do you find out which libraries a binary needs </summary>
 
```
ldd <binary> 
``` 

</details>

Source list: [tecmint - Monitor Linux Processes Resource Usage](https://www.tecmint.com/monitor-linux-processes-and-set-process-limits-per-user/), [Mark Grimes](https://channel9.msdn.com/events/Ignite/2016/BRK3268?term=Mark%20Grimes&lang-en=true)

## SELinux and AppArmor
Let's investigate the utilities view both file and process contexts for both SELinux and AppArmor.

Access control lists are enabled by setting the  ``acl`` directive in `/etc/fstab` file for desired filesystem.  But, you must remount the filesystem to activate them.

Syntax
Set:  `setfacl -m u:fred:rw /home/users/file`
Remove:  `setfacl -x u:fred /home/users/file`
Query:  `getfacl /home/users/file`
<details><summary>Create the directory /home/acl/fred. Create a file in this directory called “fredsfile”.  Set an ACL such that fred will have access to read and write to the file.  Verify that fred cannot create new files in this directory. Verify that fred can edit the file “fredsfile”</summary>

```
root # mkdir -p /home/acl/fred
root # touch /home/acl/fred/fredsfile
root # setfacl -m u:fred:rw /home/acl/fred/fredsfile
root # getfacl /home/acl/fred/fredsfile
root # sudo su - fred
fred $ cd /home/acl/
fred /home/acl/fred $ touch 1
touch: cannot touch '1': Permission denied
fred /home/acl/fred $ echo "1" > fredsfile
fred /home/acl/fred $ cat fredsfile
1
fred /home/acl/fred $ exit
```

</details>

<details><summary>Create the directory “/home/acl/group”. Create the secondary group “acltest”. Assign fred and sally to it. Set the group ownership of “/home/acl/group” to “acltest”.  Set permissions to enable SGID and to lock-out non-group users.  Test and verify this using root account and test with all appropriate user accounts including walt. As fred, Create a file called “test” in the “/home/acl/group” directory. As fred, add the line “Fred was here.” As sally, add the line “Sally was here.” As walt, fail to use the file named "/home/acl/group/test". As george, see both sally and fred's entries inside the file and clobber the file</summary>

```
# mkdir /home/acl/group
# addgroup acltest
Adding group `acltest' (GID 1005) ...
Done.
# grep acl /etc/group
acltest:x:1005:fred,sally
# setfacl -m g:acltest:wr /home/acl/group/
# getfacl /home/acl/group/
getfacl: Removing leading '/' from absolute path names
# file: home/acl/group/
# owner: root
# group: root
user::rwx
group::r-x
group:acltest:rw-
mask::rwx
other::r-x
# touch /home/acl/group/acltest
# setfacl -m g:acltest:rw /home/acl/group/acltest
```

</details>

<details><summary>To see more about a process or file with SE information:</summary>
```
* ls -Z
* ps -eZ
```

</details>

<details><summary>Restore default SELinux file contexts for /etc/passwd</summary>

```
#
```

</details>

<details><summary>Change SELINUX current state to disabled until next reboot?</summary>

```
# setenforce 0
```

</details>

<details><summary>What is my current SELINUX state?</summary>

```
# getenforce
```

</details>

<details><summary>Disable SELINUX permanently?</summary>

```
# grep SELINUX /etc/selinux/config
SELINUX=disabled
```

</details>

<details><summary>Install and start AppArmor and place tcpdump profile into enforce</summary>

```
# apt-get install apparmor-profiles
aa-enforce tcpdump
systemctl enable apparmor
systemctl start apparmor
```

</details>

Sources: [tecmint - Implementing Mandatory Access Control](https://www.tecmint.com/mandatory-access-control-with-selinux-or-apparmor-linux/), [Ubuntu AppArmor Wiki](https://wiki.ubuntu.com/AppArmor)
##	Manage Software
On Ubuntu you can add software with `apt-get install -y <packagename>`

<details><summary>Find out which package provides ping6 and install: </summary>

```
# apt-cache search ping6
iputils-ping - Tools to test the reachability of network hosts
oping - sends ICMP_ECHO requests to network hosts
thc-ipv6 - The Hacker Choice's IPv6 Attack Toolkit
# dpkg -S ping6
bash-completion: /usr/share/bash-completion/completions/ping6
iputils-ping: /bin/ping6
iputils-ping: /usr/share/man/man8/ping6.8.gz
# apt-get install -y iputils-ping
```

</details>

##	Identify the component of a Linux distribution that a file belongs to placing the package name only in the file /work/7.5/pingpackage
```
# dpkg -S /bin/ping | cut -d \:  -f 1 >  /work/7.5/pingpackage
# cat /work/7.5/pingpackage
iputils-ping
```
What is in a package:
```
# dpkg -L quota | head -3
/.
/sbin
/sbin/quotaon
```
Sources: [tecmint - 15 dpkg Command Examples](https://www.tecmint.com/dpkg-command-examples/)
#	User and Group Management
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module2-userandgroupmanagement/
##	Local user accounts
We are going to want several accounts; if they haven't been created already from previous activity.  Make sure you have fred, sally, and walt.  For Fred, make sure he is named "Fred Flintstone" and is using shell of tcsh and ensure the folders home directory gets created for fred.
```
# useradd -m sally
# useradd -m walt
# useradd -c "Fred Flintstone" -s /bin/tcsh -m fred
# passwd fred
# su fred
# sudo chage -l fred
# sudo usermod -g users fred
```
Set Fred's account to expire after 8 months, sally after one year after setting both passwords to t3sting.
```

```
Add account hal with UID of 4000, group named hal with GUID of 4000.
```

```

<details><summary>Add account nut with shell of tcsh, primary group of staff, password of abc123 but force user to rotate password upon login. </summary>

```
# useradd -g staff -s /usr/bin/tcsh nut
# passwd nut
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
# passwd --expire nut 
#### Alternatively use 
# chage -E -1 nut
```

</details>

<details><summary>Give fred access to all of hal's home directory: </summary>

```

```

</details>


<details><summary>Set sally to have the ksh shell: </summary>

```
# apt-get install -y csh
# usermod -s /bin/csh sally
userdel: sally mail spool (/var/mail/sally) not found
```

</details>

<details><summary>Provide all of walt's idenfiying information such as UID and GUID into a file named /work/43.13/walt: </summary>

```
# id walt > /work/43.13/walt
# cat /work/43.13/walt
uid=1006(walt) gid=1006(walt) groups=1006(walt)
```

</details>

<details><summary>Delete sally's account and all her files in her home directory: </summary>

```
# userdel -r sally
userdel: sally mail spool (/var/mail/sally) not found
```

</details>

<details><summary>Set sally to use Korn Shell, fred to use CSH, walt to use tcsh, and george to use sh.</summary>
</details>

## Local groups and group memberships
For future purposes, we will want the group named "limit":

For speed, just edit /etc/groups as the CLI tools are slow to use.
```
sudo groupadd limit
cat /etc/group | grep limit
groups command
addgroup 
groupadd
chmod
chgrp
```

<details><summary>Create a directory named /home/partners. Allow fred and hal to share documents in this directory using a group called partners. Both of them can read, write and remove documents from the other in this directory but any user not member of the group can’t.</summary>

```
# mkdir /home/partners
```

</details>


<details><summary>Set a group up called wild with a group password of halt.</summary>

```
# useradd abc123
# passwd abc123
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
# grep abc123 /etc/shadow
abc123:$6$hTjxsNcj$ncyhRhaNhVcFFEP.LimVdL1o85i0H4ZPw0h4oM/7.gKsCh6i8iXIds8/IQDAVIysCu7PbSa4CuIBwSpfybHH1/:17866:0:99999:7:::
# groupmod -p $6$hTjxsNcj$ncyhRhaNhVcFFEP.LimVdL1o85i0H4ZPw0h4oM/7.gKsCh6i8iXIds8/IQDAVIysCu7PbSa4CuIBwSpfybHH1/ wild
# grep wild /etc/gshadow
wild:.LimVdL1o85i0H4ZPw0h4oM/7.gKsCh6i8iXIds8/IQDAVIysCu7PbSa4CuIBwSpfybHH1/::
```

</details>


Sources: [tecmint - Managing Users & Groups](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
##	System-wide profiles
To control system wide environmental variables; tweak one of the following: /etc/environment, /etc/profile, /etc/profile.d/, /etc/bash.bashrc. 
##	User Environmental Variables
### Temporarily setting an environmental variable
<details><summary>Put a report of all the current user environmental variables for root into /work/41.56/environment</summary>

```
# env > /work/41.56/environment
# cat /work/41.56/environment
XDG_SESSION_ID=c1
TERM=xterm
SHELL=/bin/bash
USER=root
```

</details>

<details><summary>Remove mail variable temporarily</summary>

```
# env | grep MAIL
MAIL=/var/mail/root
# unset MAIL
# env | grep MAIL
#
```

</details>

### Permanently setting an environmental variable

<details><summary>Remove mail variable permanently for root</summary>

```
# unset MAIL >> ~/.bashrc
```

</details>


<details><summary>Remove set variable HELLO for root to WORLD for root</summary>

```
# echo "HELLO=WORLD" >> ~/.bashrc
# source ~/.bashrc
# echo $HELLO
WORLD
#
```

</details>

##	New user templates
Adding, deleting, and modifying the files in /etc/skel for new users
##	Restricting resource limits for users and groups
/etc/security/limits.conf is the persistant file; ideally we would use /etc/security/limits.d/fred.conf but there is no documentation in there so the main file is easier to use.
We can check with ulimit -a 
<details><summary>Add a limit configuration file for fred limiting him to 5 user processes. Two separate sessions can be maintained to help; one for root and one for fred:</summary>

```
fred $ ulimit -u
max user processes (-u) 60201
root # -s
root # echo -e "fred\thard\tnproc\t5" >> /etc/security/limits.conf
fred $ grep fred /etc/security/limits.conf
fred        hard    nproc           5
```

</details>

<details><summary>Test this as fred after logging out and then back in again as fred:</summary>

```
fred $ exit
root # sudo su - fred
fred $ ulimit -u
5
fred $ echo “Don’t run the following as root, run as fred!\n”
fred $ :(){ :|:& };: # This will explode some machines!
[1] 18921
fred $ -su: fork: retry: Resource temporarily unavailable
-su: fork: retry: Resource temporarily unavailable
-su: fork: retry: No child processes
-su: fork: retry: No child processes
-su: fork: retry: Resource temporarily unavailable
-su: fork: retry: Resource temporarily unavailable
-su: fork: retry: No child processes
-su: fork: retry: No child processes
```

</details>

<details><summary>Make a group named "problems" in which new users "sid" and "ed" are placed and make it so all group members are restricted to 20 processes (both hard and soft limits) as they have been taxing the server recently.  However, since sid hasn't been as big a problem, allow sid use as many processes as needed:</summary>

```
# echo -e "@problems\t-\tnproc\t20" >> /etc/security/limits.conf
# echo -e "sid\t-\tnproc\tunlimited" >> /etc/security/limits.conf
# useradd sid -m ; useradd ed -m ; groupadd problems; usermod -g problems sid; usermod -g problems ed
# sudo su - sid
$ ulimit -a | grep proce
max user processes              (-u) unlimited
$ exit
# sudo su - ed
$ ulimit -a | grep proc
max user processes              (-u) 20
$ exit
``` 

</details>

Sources: [Forkbomb explained](https://www.cyberciti.biz/faq/understanding-bash-fork-bomb/),  [Set Process Limits on a Per-User Basis](https://www.tecmint.com/monitor-linux-processes-and-set-process-limits-per-user/)

##	Updating privileges for groups and users
Let's go ahead and add fred to the sudo user group so he can manage system:
```
# cat /etc/group | grep ^sudo
sudo:x:27:lab
# usermod -a -G sudo lab
# cat /etc/group | grep ^sudo
sudo:x:27:lab
# usermod -a -G sudo fred
# cat /etc/group | grep ^sudo
sudo:x:27:lab,fred
```
##	Configure PAM
<details><summary>Make use of PAM to lock out user accounts after two failed ssh login attempts made to the system.</summary>

```
# whatis pam_tally2
pam_tally2 (8)       - The login counter (tallying) module
# head -2 /etc/pam.d/common-auth  # Settings must be at top!
auth required pam_tally2.so deny=2 unlock_time=900
# ssh fred@localhost
Password:
Password:
Account locked due to 3 failed logins
Password:
# pam_tally2 --user=fred
Login           Failures Latest failure     From
fred                3    12/03/18 14:00:14  127.0.0.1
# pam_tally2 --user=fred --reset
Login           Failures Latest failure     From
fred                0
```

</details>

https://www.tecmint.com/use-pam_tally2-to-lock-and-unlock-ssh-failed-login-attempts/ 

### Configure LDAP Server
To setup lb40 as an ldap server: ```# sudo apt -y install slapd ldap-utils migrationtools```
Fill in admin password desired when prompted, in this case let's use t3sting.
Use the following to configure: ```sudo dpkg-reconfigure slapd```
Let's match up with our DNS settings:
* domain: **ubuntu.local**
* Organization name: **Ubuntu**
* Administrator password: **Same password set during installation.**
* Database backend:  **MDB**.
* Do you want the database to be removed when slapd is purged? **No.**
* Move old database? **Yes**
* Allow LDAPv2 protocol? **No**

### Configuring the LDAP Clients

`/etc/ldap/ldap.conf`  is the configuration file for all OpenLDAP clients. Open this file.

sudo nano /etc/ldap/ldap.conf

We need to specify two parameters: the  **base DN**  and the  **URI**  of our OpenLDAP server. Copy and paste the following text at the end of the file. Replace  `your-domain`  and  `com`  as appropriate.

BASE     dc=ubuntu,dc=local
URI      ldap://10.20.30.40
### To have PAM use LDAP server:
To configure PAM within an application to use specifically LDAP:
### Setup LDAP

```
slappasswd -s t3sting -n > /etc/openldap/passwd
```

Sources: [Ubuntu LDAP Server](https://www.linuxbabe.com/ubuntu/install-configure-openldap-server-ubuntu-16-04)

#	File System
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module5-storagemanagement/
## Introduction to File System 
### Standard layout
Root mounted at /
Home mounted at /home
Temporary files go into /tmp

### Common Linux file systems
* Ext4 – 4th Extended FS: Very Large FS sizes, Individual file can be 16TB + Perf - Standard for many enterprises
* XFS – Developed by SGI: Individual file can be 8 EiB – Less performance - Default for RHEL7
* Ext3 – 3rd Extended FS: Large FS sizes, Individual file can be 2TB + Journaling
* btrfs – Pronounced “ButterFS”: Many ideals of ext4 & ZFS + Duplication
* Old file system support included for NTFS/FAT32/FAT16
### Investigating deployed file system
```
# cat /etc/fstab
# fdisk -l /dev/sda
# df -Th
# du
# e2label /dev/sda1
```
<details><summary>Make Partition 4 on /dev/sdc which starts at 400 MB and ends at 499 MB and is ext4.</summary>

```
parted --script /dev/sdc mktable gpt
parted -s /dev/sdc mkpart pri ext4 400 500 set 4 lvm on
```

</details>

### fdisk fundamentals
Blah, who wants to write this up.  Just know parted or fdisk, gdisk, or parted.

### File System Extended Attributes

Each filesystem can have various extended attributes which can be investigated via stat, chattr, lsattr

<details><summary>Make a file that is immutable and one that is mutable after creating both and then try to delete both:</summary>

```
# touch /work/43.23/{immutable,mutable}
# chattr +i /work/43.23/immutable
# lsattr /work/43.23/*mutable
----i---------e--- /work/43.23/immutable
--------------e--- /work/43.23/mutable
# rm /work/43.23/*mutable
rm: cannot remove '/work/43.23/immutable': Operation not permitted
```

</details>

<details><summary>Delete both immutable mutable created earlier</summary>

```
# chattr -i /work/43.23/immutable
# lsattr /work/43.23/*mutable
----i---------e--- /work/43.23/immutable
--------------e--- /work/43.23/mutable
# rm /work/43.23/{immutable,mutable}
# lsattr /work/43.23/*mutable
lsattr: No such file or directory while trying to stat /work/43.23/*mutable
```

</details>

<details><summary>Disable access time for a file such as would be desirable on an SSD drive and test:</summary>

```
# touch /work/1.3/{timenoatime,timeatime}
# chattr +A /work/1.3/timenoatime
# echo 1 > /work/1.3/timeatime
# echo 1 > /work/1.3/timenoatime
# stat /work/1.3/time* | egrep "time|Access: 20"
  File: '/work/1.3/timeatime'
Access: 2018-11-08 15:22:07.248033652 -0600
  File: '/work/1.3/timenoatime'
Access: 2018-11-08 15:18:12.279613170 -0600
```

</details>

##	File system fixing 
<details><summary>Fix your ext4 partition which you may need to create at /dev/sdg1 which has we will create errors on using `dd if=/dev/zero count=1 bs=4096 seek=0 of=/dev/sdg1`
</summary>

```
# fsck.ext4 -y /dev/sdg1
e2fsck 1.42.13 (17-May-2015)
ext2fs_open2: Bad magic number in super-block
fsck.ext4: Superblock invalid, trying backup blocks...
/dev/sdg1 was not cleanly unmounted, check forced.
Resize inode not valid.  Recreate? yes
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
Free blocks count wrong for group #0 (6398, counted=6399).
Fix? yes
Free blocks count wrong (43674, counted=43675).
Fix? yes
/dev/sdg1: ***** FILE SYSTEM WAS MODIFIED *****
/dev/sdg1: 11/12544 files (0.0% non-contiguous), 6481/50156 blocks
```

</details>

##	List, create, delete, and modify physical storage partitions
<details><summary>Create two partitions sized 100MiB on /dev/sdc with partition 1 labeled one and 2 labeled two</summary>

```
Command (? for help): n
Partition number (2-128, default 2):
First sector (34-2097118, default = 985088) or {+-}size{KMGTP}:
Last sector (985088-2097118, default = 2097118) or {+-}size{KMGTP}: +100M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/sdc: 2097152 sectors, 1024.0 MiB
Logical sector size: 512 bytes
Disk identifier (GUID): 23882B3E-1041-4CD5-9990-2C96829FB512
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 2097118
Partitions will be aligned on 2048-sector boundaries
Total free space is 1298365 sectors (634.0 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1          780288          985087   100.0 MiB   8300  Linux filesystem
   2          985088         1189887   100.0 MiB   8300  Linux filesystem
   3          391168          585727   95.0 MiB    8300  3
   4          585728          780287   95.0 MiB    8300  4
```

</details>

<details><summary>Format 1 as ext2 and 2 as ext4 and mount 1 as /ext2 and 2 as /ext4</summary>

```
# mkfs.ext2 /dev/sdc1
# mkfs.ext4 -q /dev/sdc2
# mkdir /ext4; mkdir /ext2
# mount /dev/sdc1 /ext2
# mount /dev/sdc2 /ext4
```

</details>

Sources: [tecmint - Formatting Filesystems](https://www.tecmint.com/create-partitions-and-filesystems-in-linux/)
##	Manage and configure LVM storage
### Practice Environment
My preferred way of handling this is to use a virtual machine inside virtualbox and then add several 50 MB hard drives.  I am assuming that configuration for the following commands with vagrant file and that sdc, sdm, sdn, and sdo are available.	This avoids having to undo LVM to do RAID and also allows for doing swapspace on the LVM drives.
13. /dev/sdm 50MB LVM
14. /dev/sdn 50MB LVM
15. /dev/sdo 50MB LVM
### lvm shell
Just get into lvm shell to make command lookup faster:
```
# lvm
lvm>
```

<details><summary>Put /dev/sdm and /dev/sdn into volume group named VG with 512k physical extends</summary>

```
# lvm
lvm> vgcreate VG --physicalextentsize 512k /dev/sdm /dev/sdn
  Volume group "VG" successfully created
lvm>
```

</details>

<details><summary>Create logical volume named "projects" of 20 MB ext4 that is mirrored raid1 and set it be persistent after reboot at /projects:</summary>

```
lvm> lvcreate --type raid1 -n backups -L 20M VG
  Logical volume "backups" created.

lvm> lvdisplay /dev/VG/projects
  --- Logical volume ---
  LV Path                /dev/VG/projects
  LV Name                projects
  VG Name                VG
  LV UUID                psfwcN-LF1y-qX3D-YPqI-6PuR-Tuqi-OjdQDI
  LV Write Access        read/write
  LV Creation host, time lb40, 2018-11-16 19:39:30 +0000
  LV Status              available
  # open                 0
  LV Size                20.00 MiB
  Current LE             2
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:2
# mkfs.ext4 -q /dev/mapper/VG-projects
# mkdir /projects
# blkid | grep /dev/mapper/VG-projects -w
/dev/mapper/VG-projects: UUID="51ec3153-272f-45c1-843c-e6d9099e3c69" TYPE="ext4"
#### Add to fstab and test with mount using directory which will test persistancy 
```

</details>

<details><summary>Create logical volume named "backups" of 5 MB mirrored ext2 across two drives and set it be persistent after reboot at /backups:</summary>

```
lvm> lvcreate --type raid1 -n backups -L 5M VG
  Rounding up size to full physical extent 8.00 MiB
  Logical volume "backups" created.
lvm> lvdisplay /dev/VG/backups
  --- Logical volume ---
  LV Path                /dev/VG/backups
  LV Name                backups
  VG Name                VG
  LV UUID                pHRwNf-6s30-FQIv-gT95-yf7l-BovG-Zw8mNi
  LV Write Access        read/write
  LV Creation host, time lb40, 2018-11-16 19:41:25 +0000
  LV Status              available
  # open                 0
  LV Size                8.00 MiB
  Current LE             2
  Mirrored volumes       2
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:7
# mkfs.ext2 -qq /dev/mapper/VG-backups
# mkdir /backups
# blkid | grep /dev/mapper/VG-backups -w
/dev/mapper/VG-backups: UUID="93918f45-77d7-4fc0-bf26-9a8256d9daa7" TYPE="ext2"
#### Add to fstab and test with mount
```

</details>

<details><summary>Make backups larger by 5 MB </summary>

```
lvm> lvextend -L +5M /dev/VG/backups
  Extending 2 mirror images.
  Size of logical volume VG/backups changed from 5.00 MiB (10 extents) to 10.00 MiB (20 extents).
  Logical volume backups successfully resized.
```

</details>

<details><summary>Add /dev/sdo drive into VG, then mirror backups to three drives instead of two</summary>

```
lvm> vgextend VG /dev/sdo
  Physical volume "/dev/sdo" successfully created
  Volume group "VG" successfully extended
lvm> lvconvert -m2 VG/backups
lvm> lvdisplay VG/backups
  --- Logical volume ---
  LV Path                /dev/VG/backups
  LV Name                backups
  VG Name                VG
  LV UUID                pHRwNf-6s30-FQIv-gT95-yf7l-BovG-Zw8mNi
  LV Write Access        read/write
  LV Creation host, time lb40, 2018-11-16 19:41:25 +0000
  LV Status              available
  # open                 0
  LV Size                4.00 MiB
  Current LE             1
  Mirrored volumes       3
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:7
```

</details>

<details><summary>Create 20 MB swap stripped across all three drives with lvm but don't active it yet:</summary>

```
lvm> lvcreate --stripes 3 -L 20M -n swap_lvm VG
  Using default stripesize 64.00 KiB.
  Rounding size 20.00 MiB (40 extents) up to stripe boundary size 21.00 MiB (42 extents).
  Logical volume "swap_lvm" created.
lvm> lvdisplay VG/swap_lvm -m
  Logical extents 0 to 41:
    Type                striped
    Stripes             3
    Stripe size         64.00 KiB
    Stripe 0:
      Physical volume   /dev/sdm
      Physical extents  61 to 74
    Stripe 1:
      Physical volume   /dev/sdn
      Physical extents  21 to 34
    Stripe 2:
      Physical volume   /dev/sdo
      Physical extents  21 to 34
```

</details>

<details><summary>Shrink swap "lvm_swap"to 10 MB, format it, then set it to mount upon boot</summary>

```
lvm> lvreduce -L 10M VG/swap_lvm
  WARNING: Reducing active logical volume to 10.00 MiB
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce swap_lvm? [y/n]: y
  Size of logical volume VG/swap_lvm changed from 20.00 MiB (80 extents) to 10.00 MiB (40 extents).
  Logical volume swap_lvm successfully resized.
# mkswap /dev/mapper/VG-swap_lvm
Setting up swapspace version 1, size = 10 MiB (10481664 bytes)
no label, UUID=7e2f501c-37a1-4358-af23-3d5082bf9e47
# grep swap_lvm /etc/fstab
/dev/mapper/VG-swap_lvm swap    swap    defaults        0       0
```

</details>

<details><summary>Clean this stuff out out of fstab using text editor</summary>

```
# grep mapper /etc/fstab
#
```

</details>

Sources: [tecmint](https://www.tecmint.com/manage-and-create-lvm-parition-using-vgcreate-lvcreate-and-lvextend/)
##	Create and configure encrypted storage
<details><summary>Setup /dev/sde as an encrypted 50 MB swap partition active after every boot
Make a swap partition that fills the /dev/sde which is 50MB with label of swap_encrypted and passphrase of "swap" after destroying all partitions on the drive.  Make sure it comes up after boot.
Make sure you open the document file and use it as your reference both in practicing as well as production.  Look for word urandom</summary>

```
# swapoff -a
# free | grep -i swap
Swap:             0           0           0
# wipefs -a /dev/sde
# cryptsetup luksFormat /dev/sde1
# cryptsetup luksOpen /dev/sde1 swap_encrypt
# mkswap /dev/mapper/swap_encrypt
Setting up swapspace version 1, size = 48 MiB (50290688 bytes)
no label, UUID=528e3f7e-8174-4adb-98a7-2276efde32b9
# swapon /dev/mapper/swap_encrypt
# free | grep -i swap
Swap:         49112           0       49112
# grep swap /etc/fstab 
/dev/mapper/swap_encrypted       none            swap    sw                              0 0
# grep swap /etc/crypttab
swap_encrypted               /dev/sde1         /dev/urandom swap
```
Now remove all the work you just did by destroying partition table and removing the mount on reboot:
```
# swapoff -a
# free | grep -i swap
Swap:             0           0           0
# dd if=/dev/zero of=/dev/sde bs=1024 count=2 
# grep -H swap_encrypt /etc/{fstab,crypttab}
#
```

</details>

<details><summary>Encrypt a ext4 file system on /dev/sdf 
For all of /dev/sdf make file system that is ext4. Make it active on boot under mount point of /crypt.  No password should be required during boot.  The passphrase of "passphrase" shall be used and a decryption key of /root/luks.key such that on every bootup the partition gets mounted to /crypt on lb40 with no prompt for passphrase.</summary>

```
# umount /crypt
# wipefs -a /dev/sdf
# parted /dev/sdf --script -- mklabel gpt
# parted /dev/sdf --script mkpart primary ext4 0% 100%
# parted /dev/sdf --script name 1 crypt
# dd if=/dev/random of=/root/luks.key bs=1024 count=2
# apt-get install cryptsetup
# lsmod dm_crypt
# modprobe dm_crypt
# lsmod | grep dm_crypt
dm_crypt               40960  0
# cryptsetup -y luksFormat -d /root/luks.key /dev/sdf1
WARNING!
========
This will overwrite data on /dev/sdf1 irrevocably.
Are you sure? (Type uppercase yes): YES
# cryptsetup --verbose luksOpen -d /root/luks.key /dev/sdf1  crypt
Enter passphrase for /dev/mapper/VG-crypt:
Key slot 0 unlocked.
Command successful.
# mkfs.ext4 /dev/mapper/crypt
# mkdir /crypt
# mount -v /dev/mapper/crypt /crypt
# cat /etc/fstab
/dev/mapper/crypt       /crypt  ext4    defaults        0       1
# cat /etc/crypttab
crypt   /dev/sdf1       /root/luks.key  none
```

</details>

Sources: [tecmint - Disk Encryption](https://www.tecmint.com/disk-encryption-in-linux/), [chousensha - encryption](http://chousensha.github.io/blog/2018/02/12/lfcs-prep-managing-encrypted-partitions/)
##	Configure systems to mount file systems at or during boot
Look at system as it currently stands
```
cat /etc/fstab
fdisk -l /dev/sda
mount -t type device dir
umount dir
mkfs.____ or mkfs -t ____	
df -ht
```

<details><summary>Create a new ext4 partition 10 MB in size on lb40 using one of the spare drives and mount it every boot using UUID to /mnt/10MBext4.</summary>

```
# fdisk /dev/sdc
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-2097151, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-2097151, default 2097151): +10M

Created a new partition 1 of type 'Linux' and of size 10 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
# mkfs.ext4 /dev/sdc1
# blkid | grep sdc1
/dev/sdc1: UUID="f0f91b53-f542-4ab6-92e8-064b63211b8e" TYPE="ext4" PARTUUID="2fa99c97-01"
# grep UUID /etc/fstab
UUID=f0f91b53-f542-4ab6-92e8-064b63211b8e       /mnt/10MBext4   ext4    defaults        0 2
# mkdir /mnt/10MBext4
# mount /mnt/10MBext4
# df -h /mnt/10MBext4
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc1       8.7M   92K  7.9M   2% /mnt/10MBext4
```

</details>

<details><summary>Create a 1 MB ext4 partition that doesn't mount at bootup, copy /etc/group to it,  then resize it to 2 MB mounting it temporarily at /mnt/2MB. (In this case, we are using partition /dev/sdc1 but any valid location will do.  It is just easier to illustrate with an empty drive).</summary>

```
# fdisk /dev/sdc # details removed herein
# partprobe
# mkfs.ext4 /dev/sdc1
# mount /dev/sdc1 /mnt/2MB/
# cp /etc/group /mnt/2MB
# umount /mnt/2MB
# parted
(parted) resizepart 1 3MiB # syntax will vary heavily depending on start point
# ls -al /mnt/2MB/group # this verifies we didn't lose data
-rw-r--r-- 1 root root 1048 Nov 13 20:58 /mnt/2MB/group
```

</details>

Create a new ext3 partition 20 MB in size on lb40 using one of the spare drives using lvm and mount it every boot using /dev/mapper syntax to /mnt/20MBext3.

Do a filesystem check on any unmounted partition you have created:  `fsck /dev/sdc1`
Force filesystem check next mount for root: `touch /forcefsck` which is discussed and documented in /etc/init.d/checkfs.sh
Disable filesystem check for a partition: `touch /mnt/1MB/fastboot` documented in /etc/init.d/checkfs.sh

##	Configure and manage swap space
<details><summary> Setting up a swap partition for activation after boot
Make a swap partition that fills the /dev/sdd which is 50MB with label of normal_swap after destroying all partitions on the drive.</summary>

```
# free | grep -i swap
Swap:             0           0           0
# parted /dev/sdd --script -- mklabel gpt
# parted /dev/sdd --script mkpart primary Linux-Swap 0% 100%
# parted /dev/sdd --script name 1 normal_swap
# mkswap /dev/sdd 
# swapon /dev/sdd
# free | grep -i swap
Swap:         51196           0       51196
# grep swap /etc/fstab 
/dev/sdd       none            swap    sw                              0 0
```
</details>

<details><summary>Make an LVM swap partition but don't set it up to work after boot
Create a new swap partition 10 MB in size stripping on lb40 using several LVM partitions and make active for current session but do not configure for after reboot to be active.  </summary>

```
lvcreate --size 10M --name lv_swap VG
mkswap /dev/vg/lv_swap
swapon /dev/vg/lv_swap
swapon -s
# grep lv_swap fstab
/dev/mapper/vg-lv_swap swap swap defaults 0 0
```

</details>

<details><summary>Make a simple 40MB swap partition on /dev/sdc and activate it every boot:</summary>

```
# 
```
 
 </details>
<details><summary>Make a 3 MB swap file at /work/3MBswap and temporarily use it. </summary>

```
#### pick between dd and fallocate
# fallocate  -l 3M /swap
# mkswap /swap
Setting up swapspace version 1, size = 3 MiB (3141632 bytes)
no label, UUID=f0d9560a-47b0-4781-bb7d-4b1998be62ed
# free
Swap:             0           0           0
# swapon /swap
swapon: /swap: insecure permissions 0644, 0600 suggested.
# chmod 0600 /swap
# swapon /swap
swapon: /swap: swapon failed: Device or resource busy
# free | grep swap
Swap:          3068           0        3068
```
</details>

<details><summary>Now remove all the work you just did by destroying partition table and removing the mount on reboot:</summary>

```
# swapoff /dev/sdd
# free | grep -i swap
Swap:             0           0           0
# dd if=/dev/zero of=/dev/sdd bs=1024 count=2 
# grep -w /dev/sdd /etc/fstab
```

</details>

Sources: [tecmint - Configuring Swap Partition](https://www.tecmint.com/create-partitions-and-filesystems-in-linux/)
##	Create and manage RAID devices
#### Reminder of layout of available drives
9. /dev/sdi 50MB mdadm RAID
10. /dev/sdj 50MB mdadm RAID
11. /dev/sdk 50MB mdadm RAID
12. /dev/sdl 50MB mdadm RAID

For starters, don't try to memorize all the options.  Start with the information provided by and prepare drives in case of previous attempts using: 
```
/usr/share/doc/mdadm/README.recipes
/usr/share/doc/mdadm/examples/mdadm.conf-example 
# umount /dev/md0
# mdadm --stop /dev/md0
# wipefs -a /dev/sdi /dev/sdj /dev/sdk /dev/sdl

``` 
#### Create stripped RAID drive
<details><summary>Make a striped RAID array on two hard drives (/dev/sdi AND /dev/sdj) and place a ext4 file system on it, mount it at /raid, then dismount it and disable it.</summary>

```
# less /usr/share/doc/mdadm/README.recipes
# mdadm --create /dev/md0 -n 2 -l raid1 /dev/sdi /dev/sdj
# mdadm --detail /dev/md0
# mkfs.ext4 /dev/md0 
# mkdir /raid
# mount /dev/md0 /raid
#  df -h /raid
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         45M  810K   41M   2% /raid
# umount /dev/md0
# mdadm --stop /dev/md0
# wipefs -a /dev/sdi /dev/sdj /dev/sdk /dev/sdl
```

</details>

#### Create RAID5 out of three drives, add a drive, fail a drive, bring back in failed drive, then save for reboot a three drive RAID 5 with spare.
<details><summary>On three drives (/dev/sdi /dev/sdj /dev/sdk) as /dev/md0 and format as ext4, mount to /raid, check file system.</summary>

```
# mdadm --create -l5 -n3  /dev/md0 /dev/sdi /dev/sdj /dev/sdk
# mkfs.ext4 /dev/md0 
# mkdir /raid
# mount /dev/md0 /raid
#  df -h /raid
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         45M  810K   41M   2% /raid
```
</details>

<details><summary>Add fourth drive as spare /dev/sdl</summary>

```
# mdadm --add /dev/md0 /dev/sdl
```

</details>

<details><summary>Fail drive /dev/sdi then remove</summary>

```
# mdadm --fail /dev/md0 /dev/sdi
mdadm: set /dev/sdi faulty in /dev/md0
# mdadm --remove /dev/md0 /dev/sdi
```

</details>

<details><summary>Bring back in /dev/sdi into array</summary>

```
# mdadm --add /dev/md0 /dev/sdi
```

</details>

<details><summary>Verify that there are three drives as RAID 5 and a spare</summary>

```
# mdadm  --detail /dev/md0
/dev/md0:
        Version : 1.2
  Creation Time : Tue Nov 27 17:03:59 2018
     Raid Level : raid5
     Array Size : 100352 (98.02 MiB 102.76 MB)
  Used Dev Size : 50176 (49.01 MiB 51.38 MB)
   Raid Devices : 3
  Total Devices : 4
    Persistence : Superblock is persistent

    Update Time : Tue Nov 27 17:05:30 2018
          State : clean
 Active Devices : 3
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 1

         Layout : left-symmetric
     Chunk Size : 512K

           Name : lb40:0  (local to host lb40)
           UUID : f31b37d5:c45f6f19:bf142fd1:99c0295b
         Events : 40

    Number   Major   Minor   RaidDevice State
       0       8      128        0      active sync   /dev/sdi
       4       8      176        1      active sync   /dev/sdl
       3       8      160        2      active sync   /dev/sdk

       5       8      144        -      spare   /dev/sdj
``` 

</details>

<details><summary>Make this current status persistent after boot</summary>

```
# mdadm --detail --scan >> /etc/mdadm/mdadm.conf
# echo "/dev/md0        /raid   ext4    defaults        0       2" >> /etc/fstab
```

</details>

Sources: [tecmint - Assembling Partitions as RAID Devices](https://www.tecmint.com/creating-and-managing-raid-backups-in-linux/), [mdadm cheat sheet](http://www.ducea.com/2009/03/08/mdadm-cheat-sheet/), [MSDN blog](https://blogs.msdn.microsoft.com/maheshk/2018/06/11/lfcs-managing-software-raid/)
##	Configure systems to mount file systems on demand
##	Create, manage and diagnose advanced file system permissions
```
sticky bit, setgid, and setuid
chmod 
find files with those permissions set on them
```

<details><summary>Enable suid and user's executable bit for new file called /work/7.4/suid</summary>

```
# touch /work/7.4/suid
# chmod u+sx  /work/7.4/suid 
# ls -lt  /work/7.4/suid 
-rwsr--r-- 1 root root 0 Nov 18 16:36 /work/7.4/suid
```

</details>

<details><summary>Enable guid and group's executable bit for new file called /work/7.4/guid</summary>

```
# touch /work/7.4/guid
# chmod g+sx  /work/7.4/guid
# ls -lt  /work/7.4/guid
-rwsr--r-- 1 root root 0 Nov 18 16:36 /work/7.4/guid
```

</details>

<details><summary>Using octal, set /work/7.4/octal to GUID and read only for everyone.</summary>

First, lookup octal values with man 2 stat
```
# touch  /work/7.4/octal
# chmod 02444   /work/7.4/octal
# ls -al /work/7.4/octal
---x--s--x 1 root root 0 Nov 18 20:01 /work/7.4/octal
```

</details>

<details><summary>Using octal, set /work/7.4/permissive to SUID and readwrite for everyone.</summary>

First, lookup octal values with `man 2 stat`
```
# touch  /work/7.4/permissive
# chmod 04777   /work/7.4/permissive
# ls -al /work/7.4/permissive
-rwsrwxrwx 1 root root 0 Nov 18 20:04 /work/7.4/permissive
```

</details>

<details><summary>Find all SUID files in /work and make a record in /work/5.2/suid</summary>

```
# grep -i "suid" -R /usr/share/ | grep find
/usr/share/man/man1/find.1:.B find / \e( \-perm \-4000 \-fprintf /root/suid.txt \(aq%#m %u %p\en\(aq \e) , \e
# man find 
# find /tmp -perm 4000 -print > /work/5.2/suid
```

</details>

<details><summary>Find all GUID files in /work and make a record in /work/5.2/guid</summary>

```
# grep -i "suid" -R /usr/share/ | grep find
/usr/share/man/man1/find.1:.B find / \e( \-perm \-4000 \-fprintf /root/suid.txt \(aq%#m %u %p\en\(aq \e) , \e
# man find 
# find /work -perm 4000 -print > /work/5.2/suid
```

</details>


<details><summary>Find all SUID and GUID marked  files in /work and make a record in /work/5.2/suidandguid</summary>

```
# grep -i "suid" -R /usr/share/ | grep find
/usr/share/man/man1/find.1:.B find / \e( \-perm \-4000 \-fprintf /root/suid.txt \(aq%#m %u %p\en\(aq \e) , \e
# man find 
# find /work -perm 6000 -print > /work/5.2/suidandguid
```

</details>


##	Setup user and group disk quotas for filesystems
<details><summary>Assumption is that /quota is a separate partition on /dev/sdh1 taking up the entire 50 MB drive. If not, make one for home somewhere and somehow.</summary>

```
# apt-get install quota quotatool
# mkdir /quota
/etc/fstab needs to have ,usrquota,grpquota added as options
man mount # helps to point out the syntax for ,usrquota,grpquota
mount -o remount /quota
# /etc/crontab needs to have a quotacheck added to it
# quotacheck -acug
# quotaon -a
```

</details>

<details><summary>Limiting particular users or groups block storage using "quota"</summary></details>

<details><summary>Hard and soft limits and set grace periods</summary></details>

<details><summary>Set quotas with a soft limit of 3 files and a hard limit of 7 for user fred using 1 day grace limit and verify both as root and as fred: </summary>

```
# mkdir /quota/fred; chown fred /quota/fred
# quotatool -i -q 3 -u fred /quota
# quotatool -i -l 7 -u fred /quota
# quotatool -i -u fred -t 2 /quota
# sudo su – fred
$ cd /quota/fred/
touch {1..100} # this should fail after 7 files
```

</details>


<details><summary>We can test groups now with a  user sally and fred who both belongs to a group named limit which for the group can only store 1MB hard limit:</summary>

```
# useradd -m sally -N
# groupadd limit
# usermod sally -a -G limit
# usermod fred -a -G limit
# quotatool -b -l 1025 -g limit /quota
# sudo su – sally
# mkdir /quota/limit
# chgrp limit /quota/limit
# chmod 770 /quota/limit
# sudo su - sally
$ cd /quota/limit
$ dd if=/dev/zero of=big bs=1024 count=60000
dd: error writing 'big': Disk quota exceeded
1025+0 records in
1024+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 0.00722797 s, 145 MB/s
$ ls -alh big
-rw-r--r-- 1 sally limit 1.0M Nov 15 22:31 big
```

</details>

<details><summary>While we are at it; let's make it so a new user named "walt" is restricted to a permanent limit of 4 MB and a temporary allowance to have up to 7 MB and make the length of time allowed to exceed temporary limit 1 day. </summary>

```
root # useradd walt -m
root # quotatool -u walt -b -q 4M /quota
root # quotatool -u walt -b -l 7M /quota
root # quotatool -u walt -b -t "1 day"  /quota
# repquota -a
*** Report for user quotas on device /dev/sdh1
Block grace time: 24:00; Inode grace time: 7days
                        Block limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------

walt      --      28    4096    7168              5     0     0
root # sudo su – walt
walt $ dd if=/dev/urandom of=~/bigfile bs=1M count=1000
dd: error writing '/quota/walt/bigfile': Disk quota exceeded
7+0 records in
6+0 records out
7311360 bytes (7.3 MB, 7.0 MiB) copied, 0.196509 s, 37.2 MB/s
rm ~/bigfile
```

</details>

Sources: [tecmint - Disk Quotas for Users and Groups](https://www.tecmint.com/set-access-control-lists-acls-and-disk-quotas-for-users-groups/)

## Create and configure file systems
<details><summary>Create physical volumes on top of /dev/sdn and /dev/sdo and display all volumes:</summary>

```
lvm> pvcreate /dev/sdn /dev/sdo
  Physical volume "/dev/sdn" successfully created
  Physical volume "/dev/sdo" successfully created
lvm> pvs
  PV         VG   Fmt  Attr PSize  PFree
  /dev/sdn   VG   lvm2 a--  48.00m 48.00m
  /dev/sdo   VG   lvm2 a--  48.00m 48.00m
```

</details>

<details><summary>Create the volume group named VG with extends sized 256k:</summary>

```
lvm> vgcreate --physicalextentsize 256k VG /dev/sdn /dev/sdo
Volume group "vg" successfully created
lvm> vgs
  VG   #PV #LV #SN Attr   VSize  VFree
  VG     2   0   0 wz--n- 96.00m 96.00m
```

</details>

<details><summary>You can lookup UUID with “blkid” anytime for all drives to configure a lvm into fstab</summary></details>

Sources: [tecmint - Filesystem Troubleshooting](https://www.tecmint.com/linux-basic-shell-scripting-and-linux-filesystem-troubleshooting/)
#	Network Configuration
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module3-networking/
##	Hostname resolution

```
ip addr sh
nmtui
```

<details><summary>Add nameserver 8.8.8.8 and 4.2.2.2 as additional backup DNS servers to existing network configuration</summary>

```
# echo "nameserver 8.8.8.8" >> /etc/resolv.conf
# echo "nameserver 4.2.2.2" >> /etc/resolv.conf
```

</details>

<details><summary>Add a hostname entry in lb40 for static mapping of  lb50 to 10.20.30.50</summary>

```
# echo -e "10.20.30.50\tlb50" >> /etc/hosts
```

</details>

##	Host-based firewalling with iptables
Although Ubuntu often uses ufw as an iptables front end and CentOS uses firewalld, generic iptables is better to learn as it translates across all Linux distributions as well as it is more powerful.  

All filtering tasks to be performed on lb90 only to avoid creating difficulty of troubleshooting a broken service on a machine also running another service such as bind or squid. 
<details><summary>Disable ufw if configured on lb90,  and setup iptables to accept ssh as Rule 1.</summary>

```
# apt install iptables iptables-persistent
# systemctl disable ufw
# systemctl stop ufw
# systemctl enable iptables
# iptables -A INPUT -p tcp --dport ssh -j ACCEPT
```

</details>

<details><summary>Rule 2 shall drop port 80 from 10.20.30.60 but then rule 3 allow the rest of network 10.20.30.0/24 to connect to port 80.</summary>

```
# iptables -A INPUT -p tcp --source 10.20.30.60 --dport 80 -j DROP
# iptables -A INPUT -p tcp --source 10.20.30.0/24 --dport 80 -j ACCEPT
```

</details>

<details><summary>Save current iptable rules to /etc/iptables/rules.v4</summary>

```
# iptables-save > /etc/iptables/rules.v4
```

</details>

Sources: [tecmint - iptables](https://www.tecmint.com/basic-guide-on-iptables-linux-firewall-tips-commands/)

##	Route traffic statically
Documentation : `/usr/share/doc/ifupdown/examples/network-interfaces` which is at bottom of the `man interfaces` page.
<details><summary>Place a copy of the  current routing table using numeric values only in /work/3.2/routes  </summary>

```
# route -nr > /work/3.2/routes  
```

</details>

<details><summary>Statically route traffic destined for 1.2.3.0/24 through IP 10.20.40.20 persistent across reboot</summary>

```
# grep route /etc/network/interfaces
      up route add -net 1.2.3.0 netmask 255.255.255.0 gw 10.20.40.20
# /etc/init.d/networking restart
```

</details>

<details><summary>Temporarily for this session route traffic destined for host 2.3.4.5 through IP 10.20.40.20</summary>

```
# route add 2.3.4.5 gw 10.20.40.20
```

</details>

##	Establishing and maintaining correct time 
Traditional NTP server is best; when installing ntp package the new Ubuntu mechanism gets disabled which is what you want at this time as the new Ubuntu method drifts slightly and is just not as good.

<details><summary>Install ntp and setup ntp client to using several pool servers on lb40 and host service to other lb50-lb60 devices:
</summary>

```
# apt-get install ntp -y -qqq
# cat /etc/ntp.conf
pool 0.ubuntu.pool.ntp.org iburst
pool 1.ubuntu.pool.ntp.org iburst
pool 2.ubuntu.pool.ntp.org iburst
pool 3.ubuntu.pool.ntp.org iburst
restrict 10.20.30.0 netmask 255.255.255.0 nomodify notrap
# systemctl restart ntp
```

</details>

<details><summary>Check NTP status of peers with only numeric IP entries to speed up checking:</summary>

```
# ntpq -p -n 
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
+69.197.188.178  204.9.54.119     2 u   31   64    1  126.150  -53.825  16.771
+193.225.126.76  121.131.112.137  2 u   32   64    1  188.002  -56.683  16.314
``` 

Results above are truncated.  Look for entries that aren't 16 under st column
</details>

<details><summary>To setup lb40 as server for lb50:</summary>

```
# cat /etc/ntp.conf
server 10.20.30.40
# systemctl restart ntp
# ntpq -p -n
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 10.20.30.40     184.105.182.16   3 u   10   64    1    0.361   93.312   0.000
```

</details>

<details><summary>To change timezone to CST:</summary>

```
# timedatectl set-timezone America/Chicago
# timedatectl status
      Local time: Mon 2018-11-12 11:18:47 CST
  Universal time: Mon 2018-11-12 17:18:47 UTC
        RTC time: Mon 2018-11-12 17:18:48
       Time zone: America/Chicago (CST, -0600)
 Network time on: yes
NTP synchronized: yes
 RTC in local TZ: no
```

</details>

Sources: [certdepo](https://www.certdepot.net/rhel7-set-ntp-service/)
#	Configurating Services
https://www.cheatography.com/nhatlong0605/cheat-sheets/lfcs-module6-serviceconfiguration/
## Virtual Machines
**Ok, so, honestly this is one limitation of the VirtualBox approach with Vagrant.  You are either going to need to setup another machine really running Linux or something like that.**
<details><summary>Configure libvirt hypervisor to host virtual guests (may require extra steps like enabling VT in BIOS). </summary>

```
# apt-get install qemu libvirt-bin ebtables dnsmasq-base libxslt-dev libxml2-dev libvirt-dev  -y
# systemctl start libvirtd
# systemctl enable libvirtd
```

</details>

<details><summary>Configure a virtual guest using Vagrant </summary>

```
# apt-get install qemu libvirt-bin ebtables dnsmasq-base libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev -y
# systemctl start libvirtd
# wget https://releases.hashicorp.com/vagrant/2.2.2/vagrant_2.2.2_x86_64.deb
# dpkg -i vagrant_2.2.2_x86_64.deb
# vagrant plugin install vagrant-libvirt
# vagrant init generic/ubuntu1604 
# vagrant up --provider=libvirt
```

</details>

<details><summary>Find running VMs, stop, and start it on bootup.</summary>

```
# virsh list
 Id    Name                           State
----------------------------------------------------
 1     root_default                   running

# virsh autostart root_default
Domain root_default marked as autostarted
# virsh destroy root_default
# virsh start root_default
Domain root_default started
```

</details>

<details><summary>To access VM created with Vagrant</summary>

```
# vagrant ssh
vagrant@ubuntu1604:~$ ifconfig
# exit
```

</details>

<details><summary>Access a VM console</summary>

Guests need to have serial enabled

```
# sudo systemctl enable serial-getty@ttyS0.service
# sudo systemctl start serial-getty@ttyS0.service
```

Then, console will work on the sever (use ^] to escape).
 
```
# virsh console root_default
Connected to domain root_default
Escape character is ^]

Ubuntu 16.04.5 LTS ubuntu1604.localdomain ttyS0

ubuntu1604 login: vagrant
Password:
Last login: Mon Dec  3 07:17:57 PST 2018 from 192.168.121.1 on pts/0
vagrant@ubuntu1604:~$ exit
logout

```

</details>

### Evaluate memory usage of virtual machines
### Create light-weight virtualized guests via namespaces
<details><summary>Change RAM usage to be 1500 MB and set processor count to 1.</summary>

```
# virsh edit root_default
  <memory unit='KiB'>1500152</memory>
  <currentMemory unit='KiB'>1500152</currentMemory>
  <vcpu placement='static'>2</vcpu>
```

</details>

### Cloning and replicating VMs using images or snapshots

<details><summary>Stop VM and disable startup of this VM on reboot</summary>

```
#  virsh destroy root_default
# virsh autostart --disable root_default
```

</details>

##	DNS server
###	Simple local DNS server for lookup and caching speed
A local DNS server can speedup lookups for network.  This is often called a caching server. 
Before we can do anything, we need our IP address and hostname
```
# hostnamectl
# ifconfig | grep inet
```
<details><summary>Install DNS stuff and switch to configuration folder:</summary>

```
# apt-get install bind9 dnsutils -y
cd /etc/bind/
```

</details>

<details><summary>Setup DNS caching server to use Google DNS at 8.8.8.8</summary>

```
grep "forwarders {" -A 2 /etc/bind/named.conf.options
        forwarders {
                8.8.8.8
        };
```

</details>

<details><summary>Start service, enable for reboot, and test:</summary>

```
systemctl status bind9
systemctl enable bind9
dig www.google.com @localhost
```

</details>

Sources: [tecmint - Caching DNS Server](https://www.tecmint.com/setup-recursive-caching-dns-server-and-configure-dns-zones/)
###	DNS zone management
<details><summary>Using domainname of linux.local; create master DNS server on lb40</summary>

```
# cat /etc/bind/named.conf.local
zone "linux.local" {
        type master;
        file "/etc/bind/db.linux.local";
        };
cp /etc/bind/db.local /etc/bind/db.linux.local
```

</details>

<details><summary>Edit values for zone to add a test.linux.local at 10.20.30.40 and test:</summary>
```
# tail -1  /etc/bind/db.linux.local
test    IN      A       1.2.3.4
# systemctl restart bind9
#dig test.linux.local @localhost | grep 10.20.30.40
test.linux.local.      604800  IN      A       1.2.3.4
```

</details>

<details><summary>Set machine to use local DNS server only:</summary>
```
# echo -e "nameserver 10.20.30.40\nsearch linux.local" > /etc/resolv.conf
```

</details>

<details><summary>Test to make sure server is using 127.0.0.1 as name server coming from root servers.</summary>

```
dig google.com | grep ANSWER -A 5
;; ANSWER SECTION:
google.com.             58      IN      A       172.217.14.238

;; AUTHORITY SECTION:
.                       195835  IN      NS      m.root-servers.net.
.                       195835  IN      NS      f.root-servers.net.
```

</details>

Sources: [tecmint - Configure Zones for Domain](https://www.tecmint.com/setup-recursive-caching-dns-server-and-configure-dns-zones/)
##	Email
<details><summary>Configure an IMAP and IMAPS service</summary>

```
# apt-get -y install postfix dovecot-core
```

</details>

Sources: [Installing Postfix and Dovecot](https://www.tecmint.com/installing-network-services-and-configuring-services-at-system-boot/)

<details><summary>Configure email aliases using postfix</summary>

```
# grep /etc/postfix/aliases: 
sysadmin: fred, lab
# postalias /etc/postfix/aliases
```

</details>

<details><summary>Configure email aliases using sendmail</summary>

```
# grep /etc/aliases: 
sysadmin: fred, lab
# newaliases
```

</details>

<details><summary>Restrict access to SMTP server</summary>

```
# postconf _smtpd_helo_required = yes_
# postconf  _smtpd_helo_restrictions = permit_mynetworks, reject_invalid_helo_hostname_
# postconf  _smtpd_sender_restrictions = permit_mynetworks, reject_unknown_sender_domain_
# postconf  _smtpd_recipient_restrictions = permit_mynetworks, reject_unauth_destination_
# systemctl restart  _postfix_
```

</details>

<details><summary>Enable SSL encryption in Dovecot </summary>

```
# cd  /etc/pki/tls/certs/
# make /etc/dovecot/**dovecot.pem
# grep /etc/dovecot/conf.d/10-ssl.conf
ssl = required
ssl_cert = </etc/dovecot/dovecot.pem
ssl_key = </etc/dovecot/private/dovecot.pem
```

</details>


Sources: [certdepo](https://www.certdepot.net/smtp-configure-alias/), [random](https://websphereissues.wordpress.com/2017/11/15/configure-postfix-and-dovecot-in-centos7/)
##	Secure Shell (SSH) 
```
systemctl status sshd
ssh -l username ipaddress
ssh ciphers
```
#### TASK: Set a password for the account larry on hosts 40 & 50
#### TASK: Generate an ssh-keypair for the larry account on hosts 40 & 50
#### TASK: Copy the ssh-id from 40 to 50 to enable passwordless logins to 50
#### TASK: Copy the ssh-id from 50 to 40 to enable passwordless logins to 40
#### TASK: SSH as the user larry from 40 to 50 to test passwordless authentication
#### TASK: SSH as the user larry from 50 to 40 to test passwordless authentication

<details><summary>For SSH server on lb90, disable  TCP keepalives, enable password authentication, and disable remote root login </summary>

```
# cat /etc/ssh/sshd_config
TCPKeepAlive no
PermitRootLogin no
PasswordAuthentication yes
```

</details>

<details><summary>Allow users luke and vagrant to ssh in</summary>

```
# grep Users /etc/ssh/sshd_config
AllowUsers vagrant luke
```

</details>

##	Proxy server
<details><summary>Install and enable http proxy server:</summary>

```
apt -y install squid
systemctl restart squid.service
systemctl enable squid.service
```

</details>

<details><summary>Allow subnet 10.20.30.0/24 but block ip address 10.20.30.50: </summary>

```
acl localnet src 10.20.30.0/24 
http_access allow localnet 
acl badguy src 10.20.30.50
http_access allow localhost !badguy
```

</details>

<details><summary>Testing from client devices: </summary>

```
apt -y install wget
export http_proxy='http://10.20.30.40:8080/'
URL='www.google.com'
wget -q --proxy-user=test --proxy-password=test --spider $URL
if [ $? = 1 ]
then
    STATUS= echo "Proxy isn't working"
else
  STATUS="Proxy is working."
fi
echo $STATUS
```
</details>

<details><summary>Set lb50 to use lb40 as proxy server for updates</summary>

```
# echo "Acquire::http::Proxy \"http://10.20.30.40:8080\"; " >  /etc/apt/apt.conf
```
</details>

## Web server
<details><summary> Configure an HTTP server</summary>

```
# apt-get install apache2
# systemctl enable apache2
# systemctl restart apache2
```

</details>

<details><summary> Deploy a basic web application and get rid of welcome page</summary>

```
# echo "<html><body>Hello, world.</body></html>" > /var/www/html/index.html
# curl localhost
<html><body>Hello, world.</body></html>
```
</details>

<details><summary>Configure web server to use as document root /var/www/html</summary>

```
# grep -i DOC  /etc/apache2/sites-enabled/000-default.conf
        DocumentRoot /var/www/html
```

</details>

<details><summary>Customize Web server log files</summary>

```
# 
```

</details>


<details><summary>Restrict access to a private web directory /var/www/html/nel to user nel </summary>

```
# mkdir -p /var/www/nel
# echo "<Directory /var/www/html/nel>
AuthType Basic
AuthName \"Password protected area\"
AuthUserFile /etc/apache2/passwd
Require user nel
</Directory>" > /etc/apache2/sites-enabled/010-nel.conf
# htpasswd -c /etc/apache2/passwd nel
# systemctl restart apache2
# echo "<html><body>Welcome, Nel.</body></html>" > /var/www/nel/index.html
# curl http://localhost/nel
<title>401 Unauthorized</title>
# curl http://nel:abc123@localhost/nel/
<html><body>Welcome, Nel.</body></html>
```
https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-apache-on-ubuntu-14-04

</details>

<details><summary>Disable directory indexing for /var/www/html/ including a new directory called /newdir/</summary>

```
# systemctl restart apache2
# curl http://localhost/newdir/

```
https://www.digitalocean.com/community/tutorials/how-to-set-up-password-authentication-with-apache-on-ubuntu-14-04

</details>


<details><summary>More Fun</summary>

Configure HTTP server log files
Configure SSL with HTTP server
Set up name-based virtual web hosts

</details>

##	Database server
MariaDB is the most common database server and is what we use.
<details><summary>Setup MariaDB and test</summary>

```
# apt-get install mariadb-server mariadb-client -y
# systemctl start mysql.service
# systemctl enable mysql.service
# mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 32
Server version: 10.0.36-MariaDB-0ubuntu0.16.04.1 Ubuntu 16.04
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
MariaDB [(none)]> exit
Bye
#
```

</details>

<details><summary>Secure MariaDB</summary>

```
# mysql_secure_installation
```

</details>

##	Containers
<details><summary>Setup Linux container ubuntu and start it</summary>

```
# apt-get install lxc
# systemctl status lxc.service
# systemctl enable lxc
# lxc-ls
# lxc-start -n mylxcontainer
# lxc-info -n mylxcontainer { should see the state as RUNNING }  
# lxc-console -n mylxcontainer { console login to the container, username -ubuntu, pass-ubuntu }  
# ubuntu@mylxcontainer:~$ { upon login your console prompt changes takes you to ubuntu }  
# uname -a or hostname { to confirm you are within the container }  
# Type <cntrl+a q> to exit the console  
# lxc-stop -n mylxcontainer  
# lxc-destroy -n mylxcontainer
```

</details>

<details><summary>Use Docker to setup an Apache web server on port 4321 before destroying it.</summary>

```
# apt update  
# apt install docker.io  
# systemctl enable docker  
# systemctl start docker  
# systemctl status docker  
# docker info  
# docker version  
# docker search apache
# docker images
# docker pull httpd:2.4
# docker run -dit -p 4321:80 httpd:2.4  
# docker start httpd:2.4
# # curl localhost:4321
<html><body><h1>It works!</h1></body></html>
# docker ps -la
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                     PORTS               NAMES
0fb2e7e98577        httpd:2.4           "bash"              3 minutes ago       Exited (0) 2 minutes ago                       reverent_wozniak
# docker stop httpd:2.4
# docker kill 0fb2e7e98577
# docker rm 0fb2e7e98577
0fb2e7e98577
# docker rm 0fb2e7e98577
Error response from daemon: No such container: 0fb2e7e98577
#
```

</details>

<details><summary>Use Docker to setup an Ubuntu machine, test connectivity to google.com, check stats, and look at logs.</summary>

```
# docker run -it ubuntu bash  
$ root@efe34sdsdsds:/# { takes to container bash }  
_<type cntrl p + cntrl q> to switch back to terminal_  
# apt-get install
# docker ps -al
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS                      PORTS               NAMES
e6239fd17cd9        ubuntu              "bash"              About a minute ago   Exited (0) 49 seconds ago                       angry_fermat
# docker logs containername or id  
# docker logs -f containername or id { live logs or streaming logs }  
# docker stats  
```

</details>

Sources: https://blogs.msdn.microsoft.com/maheshk/2018/05/27/lfcs-commands-to-manage-and-configure-containers-in-linux/
## File Sharing
#### NFS  

<details><summary>On lb80, share /nfstemp with dummy file named test with lb70 machines on network as /nfstemp.</summary>

```
lb80 # apt-get install -y nfswatch nfs-common nfs-kernel-server
lb80 # tail -2 /etc/hosts
10.20.30.70     lb70    lb70
10.20.30.80     lb80    lb80
lb80 # mkdir /nfstemp
lb80 # touch /nfstemp/test
lb80 # cat /etc/exports
/nfstemp       lb70(rw,no_root_squash)
# exportfs -avr
lb80 # systemctl restart nfs-server.service rpcbind
lb70 # tail -2 /etc/hosts
10.20.30.70     lb70    lb70
10.20.30.80     lb80    lb80
lb70 # mkdir /nfstemp
lb70 # mount /nfs
lb70 # df -h /nfs/
Filesystem      Size  Used Avail Use% Mounted on
lb80:/nfs       9.7G  1.4G  8.3G  14% /nfs
lb70 # ls /nfs/test
/nfs/test
```

</details>

<details><summary>On lb80, share /nfs with dummy file named test with lb70 machines on network as /nfs and set to mount persistently</summary>

```
lb80 # apt-get install -y nfswatch nfs-common nfs-kernel-server
lb80 # tail -2 /etc/hosts
10.20.30.70     lb70    lb70
10.20.30.80     lb80    lb80
lb80 # mkdir /nfs
lb80 # touch /nfs/test
lb80 # cat /etc/exports
/nfs       lb70(rw,no_root_squash)
# exportfs -avr
lb80 # systemctl restart nfs-server.service rpcbind
lb70 # tail -2 /etc/hosts
10.20.30.70     lb70    lb70
10.20.30.80     lb80    lb80
lb70 # mkdir /nfs
lb70 # grep nfs /etc/fstab
lb80:/nfs       /nfs    nfs     defaults        0       2
lb70 # mount /nfs
lb70 # df -h /nfs/
Filesystem      Size  Used Avail Use% Mounted on
lb80:/nfs       9.7G  1.4G  8.3G  14% /nfs
lb70 # ls /nfs/test
/nfs/test
```

</details>
Sources: [certdepo](https://www.certdepot.net/rhel7-provide-nfs-network-shares-specific-clients/)
#### SAMBA File Sharing 
<details><summary>On lb80 to share /smbpublic RO with lb70 machines on network as /smbpublic </summary>

```
lb80 # apt-get install -y samba samba-client samba-common
lb80 # cp /etc/samba/smb.conf /etc/samba/smb.conf.orig 
# mkdir -p /srv/samba/anonymous
# chmod -R 0775 /srv/samba/anonymous
# chown -R nobody:nobody /srv/samba/anonymous
# vi /etc/samba/smb.conf
[global]
	workgroup = WORKGROUP
	netbios name = centos
	security = user
[Anonymous]
	comment = Anonymous File Server Share
	path = /srv/samba/anonymous
	browsable =yes
	writable = yes
	guest ok = yes
	read only = no
	force user = nobody
# testparm
# systemctl enable smb.service
# systemctl enable nmb.service
# systemctl start smb.service
# systemctl start nmb.service
```

</details>

<details><summary>On lb80 to share /smbprivate RO with lb70 machines on network as /smbprivate</summary>

```
lb80 # apt-get install -y samba samba-client samba-common
lb80 # cp /etc/samba/smb.conf /etc/samba/smb.conf.orig 
# # groupadd smbgrp
# usermod tecmint -aG smbgrp
# smbpasswd -a tecmint
# mkdir -p /srv/samba/secure
# chmod -R 0770 /srv/samba/secure
# chown -R root:smbgrp /srv/samba/secure
# vi /etc/samba/smb.conf
[Secure]
	comment = Secure File Server Share
	path =  /srv/samba/secure
	valid users = @smbgrp
	guest ok = no
	writable = yes
	browsable = yes
# testparm
```

</details>

Sources: [tecmint - SMB/NFS](https://www.tecmint.com/mount-filesystem-in-linux/), [certdepo](https://www.certdepot.net/rhel7-provide-smb-network-shares/)
# Other resources
* [tecmint series](https://www.tecmint.com/sed-command-to-create-edit-and-manipulate-files-in-linux/)
* [older but still good guide](http://blog.leonelatencio.com/wp-content/uploads/2014/11/Linux-Foundation-Certified-System-Administrator-LFCS-v1.3.pdf)
* [chousensha notes](http://chousensha.github.io/blog/categories/sysadmin/)
* [Awesome start on basic CLI](https://github.com/learnbyexample/Linux_command_line/blob/master/README.md)
* [Presentations](https://github.com/guruskill/LinuxAzure/tree/master/Presentations)
* [Random notes](https://github.com/ravexina/linux-notes)
* [Cert Exam Prep: Linux Foundation Certified System Administrator (LFCS) by  Mark Grimes](https://channel9.msdn.com/events/Ignite/2016/BRK3268?term=Mark%20Grimes&lang-en=true)
* [TOC Generator](https://ecotrust-canada.github.io/markdown-toc/)
* [nhatlong0605](https://www.cheatography.com/nhatlong0605/cheat-sheets/)
* [Udemy video course of possible benefit](https://www.udemy.com/linux-foundation-ceritifed-system-administrator-lfcs-2018/learn/v4/content)
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDgxMjk1Mzk2LC00NjI2Njc1MDIsLTE0OD
I5NjAxNjMsLTQ5NTYxMjQyNywtMzg0MjMwNDg1LC0xMDM0MDIz
MDc2LDEwOTA0MzY2OTUsNTk0ODA1MTQ5LC0xMjg0ODAzMDgxLC
0xNjk2NDQ1NTQ0LDUxMzg4NDEzOCwxNzQyNTY1MDc0LC00OTgz
ODA2MzMsMTY1ODQzMTI4Miw3MTk2NDYwNzAsLTQwNDU2NzY0Mi
wtMTEwMjIzODMxMywxOTcwNzMzNDkxLC02NjExMTUwMTksLTI0
NTQ2MjA3M119
-->