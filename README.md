

# Configuring a SLURM cluster

Here you'll find a guide I've wrote to myself while experimenting on building a SLURM cluster using VMs on my computer.
I hope it doesn't happen, but eventually some of the stuff here may get deprecated as things update so fast.

On this tutorial, you won't find a basic tutorial for installing SLURM by making assumptions about your infrastructure, **we'll deploy everything from scratch**. If you already have things setup and just need to install SLURM, you can skip everything to the [Installing SLURM](#installing-slurm) section, although I'd recommend reading what's been done before so one can understand what's set and what's not.

So here's the setup I've built for the initial deploy:

* 1x DNS server machine
  * Distro: Debian 10.3.0
  * CPU: 1
  * RAM: 1 GB
  * Disk: 8 GB
  * Hostname: ns.cluster
  * IP: 10.1.1.20
  * Main softwares:
	  * Bind9
	 
* 1x Authentication machine
	* Distro: Debian 10.3.0
	* CPU: 1
	* RAM: 1 GB
	* Disk: 8 GB
	* Hostname: auth.cluster
	* Main softwares:
		* Kerberos
		* LDAP
		* LAM (LDAP Authentication Manager)
 
* 1x NAS machine: 
	* Distro: FreeNAS-11.3-U3.2
	* CPU: 1
	* RAM: 6 GB
	* Disk: 1x 8 GB + 1x 16 GB
	* Hostname: nas.cluster

* 1x Host machine (slurmctld/login):
	* Distro: Debian 10.3.0
	* CPU: 1
	* RAM: 2 GB
	* Disk: 8 GB
	* Hostname: host.cluster

* 2x Node machine (slurmd):
	* Distro: Debian 10.3.0
	* CPU: 1
	* RAM: 2 GB
	* Disk: 8 GB
	* Hostname[s]: node0[1-2].cluster

## Let's deploy!

### Installing the OS
So, first,  you have to install the OS for every machine. For that, here's the most important things on each of them:

* Debian machines:
	* Single partition on disk, no swap
	* Install only SSH server and system utilities when choosing packages

* FreeNAS:
	* Use smaller disk to install OS (outside VMs, a flash storage is preferred), you won't be able to use it for storage

### Configuring the DNS server

I've followed [this](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-18-04-pt) tutorial for start, but I'll describe it here for easing my future reading.

Install packages:

```
su -
apt update
apt install -y vim bind9 bind9utils bind9-doc
```

Here I'll use IPv4 only, so

```
vim /etc/default/bind9
```

Add `-4` to the end of `OPTIONS`. It should look like this:

```
OPTIONS="-u bind -4"
```

Restart BIND

```
systemctl restart bind9
```

Edit `/etc/bind/named.conf.options` file so it will look like this:

```
acl "trusted" {
	10.1.1.0/24;   # This should fit your network
};

options {
        directory "/var/cache/bind";

        recursion yes;                 # enables resursive queries
        allow-recursion { trusted; };  # allows recursive queries from "trusted" clients
        listen-on { 10.1.1.20; };      # This machine's IP
        allow-transfer { none; };      # disable zone transfers by default

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };

        . . .
};
```

Edit `/etc/bind/named.conf.local` file so it will look like this (replace <mark>cluster</mark> with your domain and <mark>1.1.10</mark> should fit the reverse of your network identifier (10.1.1 in my case):

```
zone "cluster" {
    type master;
    file "/etc/bind/zones/db.cluster";          # zone file path
    allow-transfer { none; };
};

zone "1.1.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.1.1.10";  # 10.1.1.0/24 subnet
    allow-transfer { none; };
};

```

Now you have to create your zone files

```
mkdir /etc/bind/zones
cp /etc/bind/db.local /etc/bind/zones/db.cluster
cp /etc/bind/db.127 /etc/bind/zones/db.10.1.1
```

And edit them to look like this:

* `/etc/bind/zones/db.cluster`
```
$TTL    604800
@       IN      SOA     ns.cluster. admin.cluster. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; NS records
@       IN      NS      ns.cluster.

; A records
ns.cluster.     IN      A		10.1.1.20
host.cluster.   IN		A		10.1.1.XX
node01.cluster.	IN		A		10.1.1.YY

...
```

* `/etc/bind/zones/db.10.1.1`

```
$TTL    604800
@       IN      SOA     ns.cluster. admin.cluster. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; NS records
@       IN      NS      ns.cluster.

; PTR records
20		IN		PTR		ns.cluster.
XX		IN		PTR		host.cluster.
YY		IN		PTR		node01.cluster.
```

Check if your files are OK

```
named-checkconf
named-checkzone cluster /etc/bind/zones/db.cluster
named-checkzone 1.1.10.in-addr.arpa /etc/bind/zones/db.10.1.1
```

Restart BIND, install firewall and allow BIND

```
systemctl restart bind9
apt install -y ufw
ufw allow Bind9
```
 And you should be done for this guy (for now).

### Configuring DNS resolving on clients

This step should be done for every machine on your network, except for FreeNAS, as we'll configure it later through the web interface.

Begin by installing the `resolvconf` package. This way we'll have more control of what's going on in the `/etc/resolv.conf` file.

```
su -
apt install -y vim resolvconf
```

Next we'll edit the `head` file for our nameserver gains priority. Open `/etc/resolvconf/resolv.conf.d/head` and append this to the end, replacing the IP with your own DNS server:

```
nameserver 10.1.1.20
```

Then just restart the `resolvconf` service and you should be fine

```
service resolvconf restart
```

### Configuring the authentication server

#### Kerberos

I've based this configuration on three tutorial: [this for Kerberos](http://techpubs.spinlocksolutions.com/dklar/kerberos.html), [this for LDAP](https://computingforgeeks.com/how-to-install-and-configure-openldap-server-on-debian/) and [this for the integration](https://wiki.debian.org/LDAP/Kerberos). As this step is a mix of these links, I'd recommend following from here.

Start by configuring debconf to low priority, so we can have more control of what's happening.

```
su -
dpkg-reconfigure debconf
```

When asked, go to interface=Dialog and set priority=low.

After that, we'll install some packages (it will ask some stuff on the process, so pay attention)

```
apt install krb5-{admin-server,kdc}
```

#### Q&A

* Default Kerberos version 5 realm
	* CLUSTER
* Add locations of default Kerberos servers to /etc/krb5.conf?
	* Yes
* Kerberos servers for your realm
	* auth.cluster
* Administrative server for your Kerberos realm
	* auth.cluster
* Create the Kerberos KDC configuration automatically?
	* Yes
* Run the Kerberos V5 administration daemon (kadmind)?
	* Yes

Don't worry if it fails to start, since your server  isn't set yet. So let's set this!

```
krb5_newrealm
```

When asked for password, choose yours (recommended a really strong password as this is the master one). On the `/etc/krb5.conf` section, look for `[domain_realm]` not `[realms]` and append your definitions:

```
.cluster = CLUSTER
cluster = CLUSTER
```

[Optional] Add the logging section at the bottom of the file

```
[logging]
	kdc = FILE:/var/log/kerberos/krb5kdc.log
	admin_server = FILE:/var/log/kerberos/kadmin.log
	default = FILE:/var/log/kerberos/krb5lib.log

```

If you've set the `logging` section, create the directory and set correct permissions

```
mkdir /var/log/kerberos
touch /var/log/kerberos/{krb5kdc,kadmin,krb5lib}.log
chmod -R 750  /var/log/kerberos
```

Edit the `/etc/krb5kdc/kadm5.acl` file and make sure the following line is there and NOT commented

```
*/admin *
```

Apply changes

```
invoke-rc.d krb5-kdc restart
invoke-rc.d krb5-admin-server restart
```

[Optional] Add password policies for new principals, a privileged principal and an unprivileged principal

```
kadmin.local

add_policy -minlength 8 -minclasses 3 admin
add_policy -minlength 8 -minclasses 4 host
add_policy -minlength 8 -minclasses 4 service
add_policy -minlength 8 -minclasses 2 user
addprinc -policy admin root/admin
addprinc -policy user unprivileged_user
quit
```

#### LDAP

First you have to install some packages (note that you can skip everything asked NOT THE PASSWORD, leaving the default options)

```
apt install -y slapd ldap-utils
```

You can show your server's details by running

```
slapcat
```

Next we're going to install LAM, the LDAP Account Manager. It's a web app and it eases our life SO MUCH

```
wget http://prdownloads.sourceforge.net/lam/ldap-account-manager_7.2-1_all.deb
sudo dpkg -i ldap-account-manager_7.2-1_all.deb
apt -f install
```

Then you can access it through `http://auth.cluster/lam`. We'll now configure it through web.

First of all, click `[LAM configuration]` at the upper right corner, then click `Edit server profiles`. It will ask for your password, the default is `lam`.

At this point, it's safer to change your password, do this on the `Profile password` section.

After that, on the `Server settings` section, set `Server address` and `Tree suffix` to match your own.

On `Security settings`, set `List of valid users` to something like `cn=admin,dc=cluster`, always matching your setup.

Do few further modifications on the `Account types` tab changing from `dc=example,dc=com` to match your stuff.

Next we'll save the configurations, returning to the homepage, and will login with the admin password. If LAM asks to create anything that's missing, you can safely allow that.

[Optional] Add a group (the first one will be the default group for every new user) and add new user(s). It's really simple to use the interface.

#### LDAP/Kerberos integration

**This step should be done on each one of the following machines:**

* Host machine (where slurmctld will run)
* Computing nodes (where slurmd will run)

Install few packages (the setup is pretty straightforward, just fill things to match your own)

```
su -
apt install krb5-config krb5-user
```

[Optional] Test Kerberos setup by doing

```
kinit -p <your-username>
klist
kdestroy
```

For this optional step, you need to have at least one principal added on the server.

Next, we'll need to use PAM for Kerberos authentication. For that, install

```
apt install libpam-krb5
```

The default configurations after installation should work fine. Next, we'll set LDAP with NSS

```
apt install libnss-ldap
```

Again, fill things to match your own. In order to ease configuration, I'll install another package where we can choose which services we'll enable. In my case, I just checked `passwd`, `group` and `shadow`

```
apt install libnss-ldapd
```

Once that's done, do

```
/etc/init.d/nscd restart
/etc/init.d/ssh restart
```

Next, we'll setup PAM for creating home directories for users that don't have it yet. For that, edit the `/etc/pam.d/common-session` file by adding this line to the end of it

```
session    required    pam_mkhomedir.so skel=/etc/skel/ umask=022
```

And you're done! Remember that, for this to work, **you have to create users both on LAM and on Kerberos server**. The password set on LAM can be anything, since the authentication is made through Kerberos. The LDAP server here is mostly used to store home path information and UID/GID management.

### NAS configuration

The FreeNAS configuration is done on the web interface and hence it should be simple. First of all, open your browser at `https://nas.cluster`.

Initially, set your timezone at `System / General`. In my case, `America/Sao_Paulo`.

Next we'll move to `System / NTP Servers`. Here, we'll remove EVERY NTP server and add one of your choice. In my case, I've chosen `1.br.pool.ntp.org`.

Then we move to `Directory Services / LDAP` and set (match your own):

* Hostname: auth.cluster
* Base DN: dc=cluster
* Bind DN: cn=admin,dc=cluster
* Bind Password: <your-admin-password>
* Enable: checked

After this is set, go to `Storage / Pools` and add one with the 16 GB disk (on my case, called `share`). I've used the suggested layout and worked fine for me. On the same page, I've added two `Datasets` into my pool, called `shared_home` and `shared_storage`. On `shared_home` advanced options, I've set a quote to 5 GB max size.

Next, we edit permissions of both datasets, setting:

* User: root
* Group: <your-ldap-default-group>
* Access Mode: User: RWX, Group: RX, Other: RX
	
Finally, we move to `Sharing / NFS` and, for each dataset, add a share. On advanced options, do:

* Maproot User: root
* Maproot Group: wheel

This step is important because if you don't set it, neither PAM will be able to create home directories, nor users. If asked for, enable the NFS service. After that, your NAS is setup.

### Setting up mounts

**This step should be done on each one of the following machines:**

* Host machine (where slurmctld will run)
* Computing nodes (where slurmd will run)

Start by installing `nfs-common` and making directories

```
su -
apt install nfs-common
mkdir -p /storage
```

After that, edit your `/etc/fstab` file for auto mount, appeding something like this on the bottom (always match your setup)

```
nas.cluster:/mnt/share/shared_home /home nfs rsize=32768,wsize=32768,bg,sync,nolock 0 0
nas.cluster:/mnt/share/shared_storage /storage nfs rsize=32768,wsize=32768,bg,sync,nolock 0 0
```

You can now mount everything

```
mount -a
```

Now we'll set PAM to create users' storage dir (if needed) at login and a logger script

```
mkdir /etc/pam_scripts
chmod -R 700 /etc/pam_scripts
chown -R root:root /etc/pam_scripts
```

Create SSH Logger script (`/etc/pam_scripts/login-logger.sh`):

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

Create storage creation script  (`/etc/pam_scripts/storage.sh`):

```
#!/bin/sh

if [ ! -d "/storage/${PAM_USER}" ]; then
	mkdir -p /storage/${PAM_USER}
	chown -R ${PAM_USER}:storage /storage/${PAM_USER}
	chmod -R 0700 /storage/${PAM_USER}
fi

exit 0
```

Append this to /etc/pam.d/sshd:

```
# Post Login Scripts
session required pam_exec.so /etc/pam_scripts/login-logger.sh
session required pam_exec.so /etc/pam_scripts/storage.sh
```

And you're done!

### Syncing clocks with NTP

**This step should be done on EVERY machine, except NAS**, because we've already set this on the web interface.

Edit the following file: `/etc/systemd/timesyncd.conf` by making it look like this (match your own setup)

```
[Time]
NTP=1.br.pool.ntp.org 0.br.pool.ntp.org
```

Uncomment `FallbackNTP` line, but leave it as is.

Then enable NTP and check its status

```
timedatectl set-ntp true
timedatectl status
```

And you're done again! After this step, you're ready to install slurm and configure it.

### Installing SLURM

As this is a SLURM cluster, we have to install SLURM, right? I know it's been too much before this could happen, but setting everything I've shown before was really important for my setup. Let's do it then!

I've based my installation on the [ubuntu-slurm](https://github.com/mknoxnv/ubuntu-slurm) tutorial but, since I'm in Debian and it's kinda outdated, I made few changes

**This step should be done ONLY on the host(slurmctld/login) machine**

First you have to install some prerequisites

```
su -
apt install gcc make ruby ruby-dev libmariadb-dev-compat libmariadbclient-dev mariadb-server
```

Then install MUNGE for authentication

```
apt install libmunge-dev libmunge2 munge
systemctl enable munge
systemctl start munge
```

Next we'll disable `mariadb` service and enable `mysql` service (if we don't disable `mariadb`, `mysql` will complain about symlinks)

```
systemctl stop mariadb
systemctl disable mariadb
systemctl enable mysql
systemctl start mysql
```

Now we create the database for SLURM by opening a mysql shell running `mysql -u root`

```
create database slurm_acct_db;
create user 'slurm'@'localhost';
set password for 'slurm'@'localhost' = password('slurmdbpass');
grant usage on *.* to 'slurm'@'localhost';
grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
flush privileges;
exit
```

Then we'll download SLURM itself

```
cd /storage
wget https://download.schedmd.com/slurm/slurm-20.02.3.tar.bz2
bunzip2 slurm-20.02.3.tar.bz2
tar xfv slurm-20.02.3.tar
cd slurm-20.02.3
```

There's a [known bug](https://bugs.schedmd.com/show_bug.cgi?id=2872) on SLURM `configure` script using the `--with-mysql_config` option. Here, I'll explain how to fix it or you can download my script from the repository, but notice that it may not (and most likely will not) work for versions different than 20.02.3 of SLURM.

To fix the script, you need to do the following:

* Run `which mysql_config` to get the path to your mysql_config script. I'll refer to this path as <msc-path> from now on
* Open the `configure` script in order to edit it
* Look for the line that has

```
test -n "$HAVEMYSQLCONFIG" || HAVEMYSQLCONFIG="no"
```

or something similar (it's the line where it checks if the mysql_config script exists and it happend to be line 5010 on my original script)

* Just **above** this line, add

```
HAVEMYSQLCONFIG=<msc-path>
```

by replacing `<msc-path>` with your own path, mine is `/usr/bin/mysql_config`

Once it's fixed, run

```
./configure --prefix=/storage/slurm-build --sysconfdir=/etc/slurm --enable-pam --with-pam_dir=/lib/x86_64-linux-gnu/security/ --without-shared-libslurm --with-mysql_config=<msc-path>
```

Then we build SLURM

```
make
make contrib
make install
```

After that, we need to make a package out of slurm and install it

```
gem install fpm
fpm -s dir -t deb -v 1.0 -n slurm-20.02.3 --prefix=/usr -C /storage/slurm-build .
dpkg -i slurm-20.02.3_1.0_amd64.deb
```

Finally, we need to add `slurm` user and make some dirs

```
useradd slurm
mkdir -p /etc/slurm /etc/slurm/prolog.d /etc/slurm/epilog.d /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm
chown slurm /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm
```

Okay. So, by now, SLURM should be fully installed. Before we proceed, we need a `slurm.conf` file, in order to configure our cluster. You can generate your own conf file by using [this amazing tool](https://slurm.schedmd.com/configurator.html) from SchedMD, which is the SLURM official development and support team. Mine looks like this:

```
# slurm.conf file generated by configurator.html.
# Put this file on all nodes of your cluster.
# See the slurm.conf man page for more information.
#
ClusterName=compute-cluster
SlurmctldHost=host(host.cluster)
#SlurmctldHost=
#
#DisableRootJobs=NO
#EnforcePartLimits=NO
#Epilog=
#EpilogSlurmctld=
#FirstJobId=1
#MaxJobId=999999
#GresTypes=
#GroupUpdateForce=0
#GroupUpdateTime=600
#JobFileAppend=0
#JobRequeue=1
#JobSubmitPlugins=1
#KillOnBadExit=0
#LaunchType=launch/slurm
#Licenses=foo*4,bar
#MailProg=/bin/mail
#MaxJobCount=5000
#MaxStepCount=40000
#MaxTasksPerNode=128
MpiDefault=none
#MpiParams=ports=#-#
PluginDir=/usr/lib/slurm
#PlugStackConfig=
#PrivateData=jobs
ProctrackType=proctrack/cgroup
#Prolog=
#PrologFlags=
#PrologSlurmctld=
#PropagatePrioProcess=0
#PropagateResourceLimits=
#PropagateResourceLimitsExcept=
#RebootProgram=
ReturnToService=2
#SallocDefaultCommand=
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmctldPort=6817
SlurmdPidFile=/var/run/slurmd.pid
SlurmdPort=6818
SlurmdSpoolDir=/var/spool/slurm/d
SlurmUser=slurm
#SlurmdUser=root
#SrunEpilog=
#SrunProlog=
StateSaveLocation=/var/spool/slurm/ctld
SwitchType=switch/none
#TaskEpilog=
TaskPlugin=task/cgroup
#TaskPluginParam=
#TaskProlog=
#TopologyPlugin=topology/tree
#TmpFS=/tmp
#TrackWCKey=no
#TreeWidth=
#UnkillableStepProgram=
#UsePAM=0
#
#
# TIMERS
#BatchStartTimeout=10
#CompleteWait=0
#EpilogMsgTime=2000
#GetEnvTimeout=2
#HealthCheckInterval=0
#HealthCheckProgram=
InactiveLimit=0
KillWait=30
#MessageTimeout=10
#ResvOverRun=0
MinJobAge=300
#OverTimeLimit=0
SlurmctldTimeout=120
SlurmdTimeout=300
#UnkillableStepTimeout=60
#VSizeFactor=0
Waittime=0
#
#
# SCHEDULING
#DefMemPerCPU=0
#MaxMemPerCPU=0
#SchedulerTimeSlice=30
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
#
#
# JOB PRIORITY
#PriorityFlags=
#PriorityType=priority/basic
#PriorityDecayHalfLife=
#PriorityCalcPeriod=
#PriorityFavorSmall=
#PriorityMaxAge=
#PriorityUsageResetPeriod=
#PriorityWeightAge=
#PriorityWeightFairshare=
#PriorityWeightJobSize=
#PriorityWeightPartition=
#PriorityWeightQOS=
#
#
# LOGGING AND ACCOUNTING
#AccountingStorageEnforce=0
#AccountingStorageHost=
#AccountingStorageLoc=
#AccountingStoragePass=
#AccountingStoragePort=
AccountingStorageType=accounting_storage/slurmdbd
#AccountingStorageUser=
AccountingStoreJobComment=YES
#DebugFlags=
#JobCompHost=
#JobCompLoc=
#JobCompPass=
#JobCompPort=
JobCompType=jobcomp/none
#JobCompUser=
#JobContainerType=job_container/none
JobAcctGatherFrequency=30
JobAcctGatherType=jobacct_gather/linux
SlurmctldDebug=info
#SlurmctldLogFile=
SlurmdDebug=info
#SlurmdLogFile=
#SlurmSchedLogFile=
#SlurmSchedLogLevel=
#
#
# POWER SAVE SUPPORT FOR IDLE NODES (optional)
#SuspendProgram=
#ResumeProgram=
#SuspendTimeout=
#ResumeTimeout=
#ResumeRate=
#SuspendExcNodes=
#SuspendExcParts=
#SuspendRate=
#SuspendTime=
#
#
# COMPUTE NODES
NodeName=node01 NodeHostName=node01.cluster NodeAddr=node01.cluster CPUs=1 RealMemory=1990 State=UNKNOWN
NodeName=node02 NodeHostName=node02.cluster NodeAddr=node02.cluster CPUs=1 RealMemory=1990 State=UNKNOWN
PartitionName=cpu Nodes=node0[1-2] Default=YES MaxTime=INFINITE State=UP
```

This file shall be placed in `/etc/slurm/slurm.conf`.

Now we'll deploy SLURM. For it to work, I'll provide some scripts that I got from the original tutorial in order to add'em to systemd (cloning this repository is optional but, in order to ease the tutorial, I chose to do it)

```
apt install git
git clone https://github.com/gabriel-milan/slurm-cluster

cp slurm-cluster/slurmdbd.service /etc/systemd/system/
cp slurm-cluster/slurmctld.service /etc/systemd/system/

systemctl daemon-reload
systemctl enable slurmdbd
systemctl start slurmdbd
systemctl enable slurmctld
systemctl start slurmctld
```

Okay, so now you can check if it's working fine by doing

```
systemctl status slurmdbd
systemctl status slurmctld
```

If everything is working fine, you can proceed. Now, create the SLURM cluster, account and some users (I recommend creating those users before on LDAP and Kerberos. That way, you'll be able to SSH into any machine and run commands without worrying with authentication and stuff)

```
sacctmgr add cluster compute-cluster
sacctmgr add account compute-account description="Compute accounts" Organization=OurOrg
sacctmgr create user <your-user> account=compute-account adminlevel=None
```

Finally, if everything went well, you can check system health by typing `sinfo`. Don't worry if your nodes show no state, we haven't configured them yet. So let's do this now!

**This step should be done ONLY on the node(slurmd) machines**

First we're gonna install MUNGE, for authentication

```
su -
apt update
apt install libmunge-dev libmunge2 munge
```

We need to have the same copy of `munge.key` (located at `/etc/munge/munge.key`) for every node. In order to do that, copy it from `host.cluster` to every other machine. One way to do that is over SCP

```
scp host.cluster:/etc/munge/munge.key /etc/munge/.
```

Then fix permissions

```
chown munge:munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
```

And restart MUNGE

```
systemctl enable munge
systemctl restart munge
```

After setting MUNGE, we're going to install the same copy of SLURM we've built on the host machine by doing

```
dpkg -i /storage/slurm-20.02.3_1.0_amd64.deb
```

Next, we need to create the `/etc/slurm` directory and have the same copy of `/etc/slurm/slurm.conf` on every machine. Similarly to the MUNGE key, do

```
scp host.cluster:/etc/slurm/slurm.conf /etc/slurm/.
```

Then you need to create the `/etc/slurm/gres.conf` file, which takes care of listing resources other than CPUs (like GPUs, for example). **In my case, this will be just an empty file** but, if you need an example, a configuration for the DGX-1 server is below

```
NodeName=linux1 Name=gpu File=/dev/nvidia0 CPUs=0-19,40-59
NodeName=linux1 Name=gpu File=/dev/nvidia1 CPUs=0-19,40-59
NodeName=linux1 Name=gpu File=/dev/nvidia2 CPUs=0-19,40-59
NodeName=linux1 Name=gpu File=/dev/nvidia3 CPUs=0-19,40-59
NodeName=linux1 Name=gpu File=/dev/nvidia4 CPUs=20-39,60-79
NodeName=linux1 Name=gpu File=/dev/nvidia5 CPUs=20-39,60-79
NodeName=linux1 Name=gpu File=/dev/nvidia6 CPUs=20-39,60-79
NodeName=linux1 Name=gpu File=/dev/nvidia7 CPUs=20-39,60-79
```

After that, as we'll use `cgroup`, you need to create both of these files:

* `/etc/slurm/cgroup.conf`

```
CgroupAutomount=yes 
CgroupReleaseAgentDir="/etc/slurm/cgroup" 

ConstrainCores=yes 
ConstrainDevices=yes
ConstrainRAMSpace=yes
#TaskAffinity=yes
```

* `/etc/slurm/cgroup_allowed_devices_file.conf`

```
/dev/null
/dev/urandom
/dev/zero
/dev/sda*
/dev/cpu/*/*
/dev/pts/*
/dev/nvidia*
```

This will be useful later for forbiding users from SSH into machines they have no jobs allocated

Then, create missing SLURM user and make some directories

```
useradd slurm
mkdir -p /var/spool/slurm/d
```

Assuming you've cloned this repository, copy the `slurmd.service` file and then enable it

```
cp /storage/slurm-cluster/slurmd.service /etc/systemd/system/
systemctl enable slurmd
systemctl start slurmd
```

At this point, if you've done everything right, you should be able to do `sinfo` and see your node state `idle`.

### Future work

* Multi-factor priority: https://slurm.schedmd.com/priority_multifactor.html
* SLURM account synchronization with UNIX groups and users: https://slurm.schedmd.com/SLUG19/DTU_Slurm_Account_Sync.pdf
* Setup cgroup: https://github.com/mknoxnv/ubuntu-slurm
* Setup passwordless SSH with Kerberos, following the end of this tutorial: http://techpubs.spinlocksolutions.com/dklar/kerberos.html
* Setup multiple job queues: add new partitions to `slurm.conf`
* Some sort of mechanism to keep `slurm.conf` always synced. Maybe a shared mount?
* Web dashboard (couldn't put slurm-web to work)

