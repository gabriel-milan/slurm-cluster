
# Configuring a SLURM cluster

## Node configuration

### Requirements
* nfs-common

### Setup
* Setup this on every node: https://www.howtoforge.com/set-up-openldap-client-on-debian-10/
* Setup mounts on /etc/fstab, example:
```
10.1.1.11:/mnt/Pool01/storage /storage nfs rsize=32768,wsize=32768,intr,bg,noatime,sync,nolock,umask=066 0 0
10.1.1.11:/mnt/Pool01/home /home nfs rsize=32768,wsize=32768,intr,bg,noatime,sync,nolock,umask=066 0 0
```
* Create directory /etc/pam_scripts:
```
mkdir /etc/pam_scripts
chmod -R 700 /etc/pam_scripts
chown -R root:root /etc/pam_scripts
```
* Create SSH Logger script (`/etc/pam_scripts/login-logger.sh`):
```
#!/bin/sh

LOG_FILE="/var/log/ssh-auth"

DATE_ISO=`date --iso-8601="seconds"`
LOG_ENTRY="[${DATE_ISO}] ${PAM_TYPE}: ${PAM_USER} from ${PAM_RHOST}"

if [ ! -f ${LOG_FILE} ]; then
	touch ${LOG_FILE}
	chown root:adm ${LOG_FILE}
	chmod 0640 ${LOG_FILE}
fi

echo ${LOG_ENTRY} >> ${LOG_FILE}

exit 0
```
* Create storage creation script  (`/etc/pam_scripts/storage.sh`):
```
#!/bin/sh

if [ ! -d "/storage/${PAM_USER}" ]; then
	mkdir -p /storage/${PAM_USER}
	chown -R ${PAM_USER}:storage /storage/${PAM_USER}
	chmod -R 0700 /storage/${PAM_USER}
fi

exit 0
```
* Append this to /etc/pam.d/sshd:
```
# Post Login Scripts
session required pam_exec.so /etc/pam_scripts/login-logger.sh
session required pam_exec.so /etc/pam_scripts/storage.sh
```

## LDAP

For account management.

Setup a single machine for this (can be a VM)
https://computingforgeeks.com/how-to-install-and-configure-openldap-server-on-debian/
After it is set, you should be able to manage accounts on http://ldap-machine-ip/lam/

## FreeNAS

For shared storage

### Configuring
* Create a Pool(s)
* Create dataset for /home
* Create dataset for /storage
* Edit ACL for both datasets doing Owner User=root, Owner Group=<your-custom>, Owner=Write, Group=Read, Everyone=Read, Group(wheel)=Full control
* Share both of these datasets using NFS and set Maproot User to root and Maproot Group to wheel
