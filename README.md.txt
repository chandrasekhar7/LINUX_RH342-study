# Note:  I have not taken this exam yet so there's absolutely nothing in this document that has anything to do with the exam.  I am bound by a non-disclosure agreement so please don't ask me anything about the exam.


# Things to Remember:

Ensure services are set to restart on reboot.
Check those SELinux labels!  Ensure SELinux booleans are set permanently with "-P".
Ensure the firewall rules are permanant and firewalld is set to restart on reboot (if applicable).



TODO:  man page that shows how to initiate crash
       man page that has password,keyboard-interactive for SSH

Things I don't have a full grasp on:

pcp, pmlogger, pmval


# Chapter 1:  Using the Scientific Method

student can't log in

check:  ssh, 'last log' (lastlog -u student), user shell
getent passwd studentexit


## Collecting Information

journactl -ef (enable and follow)
journalctl -u sshd.service (just sshd service)
journalctl -p emerg..err (just syslog priority stuff)

journalctl -b -1 (show messages from last boot)

'man journalctl' will lead you to 'man systemd-journald.service' page that gives you clues on how to make the storage persistent.

Instead of manually creating the /var/log/journal directory, one can also change the systemd-journald configuration file:

Raw

    # sed -i 's/#Storage=auto/Storage=persistent/' /etc/systemd/journald.conf
    
    # systemctl restart systemd-journald.service

    ausearch -i -m avc -ts today (search audit logs for AVC)

redhat-support-tool



    sos-report -l ( list available modules)
    sosreport -e xfs -k xfs.logprint (enables xfs module and xfs.logprint option enabled)
    redhat-support-tool (menu driven, you don't need to know the command line)


[root@demo ~]# yum -y install redhat-access-insights


Free Labs:

https://access.redhat.com/labs/


## What is troubleshooting?

    firewall-cmd --list-all


# Chapter 2:  Being Proactive


    yum -y install cockpit
    yum -y install pcp; systemctl enable pmcd; systemctl start pmcd
    (rpm -qil pcp | grep systemd gives hints to the service name)
    firewall-cmd --add-service=cockpit --permanent

When copying log files to another system, ensure you also copy the .meta files.


pmlogger
pmatop
pmstat
pmval -a <log file> kernel.all.load -T 1minute

## Remote Logging

install rsyslog-doc so that you can get the stuff that's not in the man pages.  Look in the configuration -> actions section for a few examples.


TCP and UDP server run settings are already in the /etc/rsyslog.conf file but may not be enabled (ie: they're commented out).

Ensure firewall is open for TCP or UDP or both.


Templates:  

$template DynamicFile,"/var/log/loghost/%HOSTNAME%/cron.log"
cron.* ?DynamicFile

The DynamicFile is just any name for the template.  You must reference the template by prepending ?  This is not in any man page!

## Being Proactive

yum install -y aide

/etc/aide.conf:  Note the use of @@ to define macros.  SImilar to C.

AIDE needs to be run first when the system is *clean*.  Aide checks changes so if the file is already corrupt, aide won't find it.

auditd rules work on first-match wins.  Order is important.

use 'man auditd' to get hints to the other audit binaries and auditd.conf syntax.

auditctl -w /etc/passwd -p wa -k user-edit
        -w add watch
        -p write/attribute changes
        -k give a tag/keyword of user-edit



auditctl -W <rule to remove>
auditctl -D <deletes all rules>

USE 'service auditd restart' and not 'systemctl restart auditd'
Also use 'systemctl daemon-reload'

# Chapter 3:  Troubleshooting Boot Issues

Grub2 is in /boot/grub2
/boot/grub2/grub.cfg << do not edit.
/etc/grub.d/  - config files
/etc/default/grub - this is where edits are made.

lsblk

grub2-install /dev/vda
ctrl-d, ctrl-d to exit,reboot
grub2-mkconfig -o /boot/grub2/grub.cfg



UEFI boot issues.  To boot 2TB o more, it needs to be  GUID Partition Table (GPT) and a EFI System Partition.

/boot/efi
/boot/efi/EFI/redhat

grub2-install will overwrite the grubx64.efi.  If the system was set up for secure boot, this will cause the boot to fail.

yum reinstall grub2-efi shim
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg

never edit grub.cfg manually.  make changes in /etc/default/grub2.

if shim.efi is not registered with the firmware, a default boot will occur.

## Dealing with Failing Services
/etc/systemd/system/<unitname>.d/

systemctl list-dependencies
man systemd.unit
man systemctl -> man systemd.target => man systemd.unit

Conflicts=  stops the offending service if it's running.

systemctl list-dependencies nfs-server.service

basic.target is first

Don't forget systemctl daemon-reload!!



yum install graphviz eog

systemctl list-dependencies rsyslog.service
systemctl list-jobs

man systemctl - shows dependency tasks

Breaking into the system to debug: debug-shell.service on /dev/tty9


## Recovering a Root Password

Remove the console sections that refer to TTYs in the grub2 boot menu.

load_policy -i; restorecon -Rv /etc (after recovering password)

grub2-mkconfig -o /boot/grub2/grub.cfg

# Identifying Hardware Issues

## Identifying Hardware Issues and Practice



dmidecode
hdparm
lscsci -v  # shows scsi devcies and where they are in the /proc system
hdparm -I
dmdecode -t memory
lscpu
lsblk
memtest

mcelog package
rasdaemon package


## Managing kernel modules


Install 'kernel-doc' package for documentation on kernel modules and such.  kernel-parameters.txt.  Look for usb-storage.quirks and at the end of the section there's an example

lsmod

ll /sys/module
modinfo -p <module> (shows parameters the module has)
modprobe -r <module> (unload)


## Handling Virtualization Issues

virsh

/usr/share/libvirt/schemas
virt-xml-validate

## Identifying Hardware Issues - LAB

yum install kernel-doc
/usr/share/doc/../../../
usbstoragedriver.quirks

# Troubleshooting Storage Issues

dmsetup ls

When xfs_repair-ing a file system, pay attention to the XFS errors.  If you have a corrupted file (that ends up in lost+found), look for the parent directory inode number.  Then:  find . -inum <number> to find the inode number of the directory that contains the lost+found file.  Then deduce what file belongs there.

dumpe2fs <block device> | grep superbl
e2fsck -n -b <superblock> <block device>

## Recovering from LVM accidents

ensure archive=1 is on in /etc/lvm/lvm.conf
retain_days
retain_min

vgcfgrestore -l <volume group>

## Dealing with LUKS Issues
/etc/crypttab  # part of INIT
yum -y install cryptsetup


  name /dev/vdaN /path/to/key/file


Then /etc/fstab

/dev/mapper/name /secret ext4 defaults 0 0 

UUID=<uuid> /secret ext4 defults 0 0 

cryptsetup luksDump /dev/vdb1

cryptsetup lukheaderBackup /dev/vdb1

dmsetup ls --target crypt


If you have trouble closing the luks secure file system, try: cryptsetup luksClose <friendly name>

The password key file must NOT contain any trailing characters:

echo -n <password> > /root/luks_key; chmod 400 /root/luks_key

## Troubleshooting iSCSI Storage Issues

The target is something to attach to...something to login to.  That target can be one or more disks.  Inside a target is LUNs.
A target can present more than on LUN/disk.

After logging in to a target, use "lsblk" to show the new LUNs/disks.

Discovery is the process of going to the server to find what the targets are configured.  Restrictions can be in place to show only targets for a particular host (by IP address in an ACL)

Ensure the IQN i n/etc/iscsi/initiaatorname.isci is setup up properly and that same IQN needs to be used when setting up ACLs on the target.

Any change to iscsid.conf:  restart iscsi service.

CHAP protocol:  Password authentication for target connection.  By default, RH iscsi uses no authentication.  CHAP must be configured on the target and initiatior.  Authentication settings are in /etc/iscsi/iscsid.conf.  For one-way authentication, use 'username' and 'password' credentials.  For two-way authentication, use 'username_in' and 'password_in'.

discovery.sendtargets.auth.authmethod = CHAP
discovery.sendtargets.auth.username = <username>
discovery.sendtargets.auth.password = <password>


man iscsiadm | grep iscsiadm | grep discover
man iscsiadm | grep iscsiadm | grep login

iscsiadm -m node -d8 # debug 8 is required to see authentication info

iscsiadm -m node -u # Unlogin
iscsiadm -m node -T <iqdn> -o delete # delete client side cache/information

iscsiadm --mode node --logoutall=all

Connection information/parameters are in /var/lib/iscsi/nodes (session information)

# Troubleshooting RPM Issues

## Resolving Dependency Issues 

yum deplist yum

rpm -q --requires yum

yum versionlock [list | add | delete | clear]

yum list --showduplicates

## Recovering a Corrupt RPM Database

The utilities you need are not in $PATH but in /usr/lib/rpm/.  Use 'rpm -qil rpm' to give you hints where the utilities are and where the RPM database is.

rpmdb_dump

rpmdb_verify

    cd /var/lib/rpm
    rm /var/lib/rpm/__db.*
    /usr/lib/rpm/rpmdb_verify Packages
    mv Packages Packages.broken
    /usr/lib/rpm/rpmdb_dump Packages.broken | /usr/lib/rpm/rpmdb_load Packages 
    rpm -qa > /dev/null # shows errors in process
    rpm --rebuilddb

## Identifying and Recovering Files that have Changed

yum install yum-plugin-verify

yum verify-rpm <package>

## Subscribing Systems to Red Hat Updates

[root@demo ~]# rct cat-cert /etc/pki/entitlement/7868709839063398548.pem | > egrep 'ID|Serial'

## Resolving issues with network device naming




# Resolving Network Issues

biosdevname=1 kernel parameter
biosdevname package must be installed
nmcli conn
nmcli conn reload

# Kerberos and LDAP

The authconfig RPM has the cacertdir_rehash binary.

cacertdir_rehash /etc/openldap/cacerts

# Troubleshooting Application Issues 
objdump -p /path/to/so | grep SONAME

ldconfig -p # Gives hints as to where it looks for libraries.

'man valgrind'....you're looking for memory leaks so search for 'leak'
ldd /path/to/binary

valgrind will give you the hint of --leak-check=full when you run it.

valgrind --tool=<toolname>

search for "tool" in man valgrind

strace

strace -o /tmp/mytrace -e open,stat mycommand

strace -p <PID>

strace -f (follow child processes)

ltrace


auditd
ausearch -a <event>
ausearch -i <event>  (shows errors in human readable)
ausearch -i -f <filename> (looks for audit errors by file name

# Chapter 9:  Dealing with Security Issues

ausearch -m avc -ts recent

auditd

audit2allow

journalctl -u vsftpd.service

Check /etc/pam.d/ files for changes.  Man pages on (for example) pam_ftp can be helpful.

If no /etc/pam.d/<servicname> is found, then /etc/pam.d/other file is used to determine authentication.


Recommended to do authconfig --savebackup <name> before making any changes.

PAM errors are logged to /var/log/secure.


## Fixing SELinux Issues

semanage fcontext -a -t <type> /path/to/file

restorecon -Rv /path/to/file

semanage boolean --list

setsebool -P # Permanent

getsebool

sesearch --allow -b httpd_can_connect_ldap
seinfo --portcon=443



autofs # check man 5 autofs for the auto-home directory syntax

## Resolving Kerberos and LDAP Issues


ssh -o PreferredAuthentications=keyboard-interactive,password  # TAB completion works here!


systemctl status nfs-secure
klist -ek /etc/krb5.keytab


Check that nfs-secure has credentials to kerberos destination.

check security of nfs in /etc/exports.d
check KRB5 ticket for service nfs-secure


# Dealing with Securitiy Issues

ssh -o PreferredAuthentications=keyboard-interactive,password # Forces the use of password instead of SSH keys.


kinit <user>  # get a TGT for the <user>

check /etc/krb5.conf and /etc/sssd/sssd.conf for configuration issues.


exportfs # checks NFS exports
         # check /etc/exports and /etc/exports.d/


# Troubleshooting Kernel Issues
## Kernel Crash Dumps

kdump installed by default

systemctl enable kdump
systemctl start kdump

Configure kdump
 # ensure you have the correct core collector uncommented when using SSH copy
kdumpctl propagate
restart kdump

echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger  (triggers a crash)

core_collector makedump file

## Kernel Debugging With SystemTap 

you need kernel-debuginfo and kernel-devel packages installed to use system tap.
man stap | grep debug   # gives you hint to the 'stap-prep' program that installs everything it needs.


non-root users must be in stapusr or stapdev group  (man stap | grep group)
