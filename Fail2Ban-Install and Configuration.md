# Introduction
When SSH port is exposed through the Internet, threat actors will try to challenge it security by attempting to brute force it.

This guide describes how a service called Fail2Ban can be installed and configured in Ubuntu to protect SSH.

In principle, Fail2Ban is capable to interact with iptables firewall for automatically block an ip addreess when the amount of unsuccessful login attempts hit a given threshold or rules defined in this guide.

# Installing Fail2Ban service

1. Make sure operating system is up to date: 

``` sudo apt-get update ```

2. Install fail2ban

``` sudo apt-get install fail2ban ```

3. Ensure fail2ban runs at the system startup (this is important in the event the system reboot)

``` systemctl enable fail2ban.service 

root@another-fail2ban:~# systemctl enable fail2ban.service
Synchronizing state of fail2ban.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable fail2ban

```

4. Make sure that Fail2Ban config file exists at /etc/fail2ban/

```
ls /etc/fail2ban/

action.d       fail2ban.d  jail.conf  paths-arch.conf    paths-debian.conf
fail2ban.conf  filter.d    jail.d     paths-common.conf  paths-opensuse.conf
root@ubuntu-s-1vcpu-1gb-nyc3-01:~# 
```

# Configuring Fail2Ban settings

Fail2Ban configuration files are located at /etc/fail2ban. Default file is called ``` jail.conf ```. It is not recommended to edit this file but create a new copy named ``` jail.local```. By doing this we will guarantee that our settings are not going to be overriden wheiles running package upgrades.


`sudo cp /etc/fail2ban/jail.{conf,local}`


In order to configure the Fail2Ban based on our ban policy, we need to edit the just created /etc/fail2ban/jail.local file

``` sudo vi /etc/fail2ban/jail.local```

# Whitelisting your IP Address

Fail2Ban allows you to white list (ignore your IP Address, to do so, you need to edit the /etc/fail2ban/jail.conf and uncomment the line starting with # "ignoreip"

`ignoreip = 127.0.0.1/8 ::1 170.245.242.50 `

Note that white listed IP Addresses are separated by a white space.

# Configuring Ban time

Ban time is the lengh of time a specific client IP Address will be banned when its behaviour violates the specific ban policy that will be defined. You can ban a client for 10 minutes, 1 hour, 1 day. In this specific case let's say we wan to ban a client for 1 day.

Uncomment the bantime setting in the jail.local configuration file so it looks like this

`bantime = 1d`

* Note if you want to permanently ban a client, put a negative number in the bantime setting

# Configuring Find Time and Max Retry flags

findtime is the duration between the number of failures before a ban is set. For example, if Fail2ban is set to ban an IP Addrress after five failures (maxretry), those failures must occur within the findtime duration/threshold. In other words, this is our threshold window.

Let's say our approach is to ban a client for 1 day when there have been 5 failed login attempts within a 10 minute window, you need to change this settings:

```
bantime = 1d

findtime = 10m

maxretry = 5

```

# Restart the fail2ban service


# Checking fail2ban service with fail2ban-client

```root@ubuntu-s-1vcpu-1gb-nyc3-01:~# sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	5
|  `- File list:	/var/log/auth.log
`- Actions
   |- Currently banned:	1
   |- Total banned:	1
   `- Banned IP list:	185.156.74.65
```

# Easiest way: Setting up a filtering Service

1. Create a sshd.conf file at /etc/fail2ban/jail.d directory with your favorite editor

``` vi /etc/fail2ban/jail.d/sshd.conf```

2. Paste the following configuration

``` 
[sshd]
enabled = true
port = ssh
action = iptables-multiport
logpath = /var/log/auth.log
bantime  = 1d
findtime = 10m
maxretry = 5
``` 

3. Restart the fail2ban service:

``` systemctl restart fail2ban ````

4. Make sure the service is up and running

``` systemctl status fail2ban ```

5. Tail /var/log/fail2ban.log  to see how the log looks like. When an IP Address is banned, the log looks like this:

```
root@fail2bandemo:/etc/fail2ban/jail.d# tail /var/log/fail2ban.log 
2021-01-06 14:54:24,678 fail2ban.actions        [14255]: INFO      banTime: 86400
2021-01-06 14:54:24,678 fail2ban.filter         [14255]: INFO      encoding: UTF-8
2021-01-06 14:54:24,678 fail2ban.filter         [14255]: INFO    Added logfile: '/var/log/auth.log' (pos = 2012, hash = c2a531b8ff571067a4dc30dfb761f6be1ac370e3)
2021-01-06 14:54:24,680 fail2ban.jail           [14255]: INFO    Jail 'sshd' started
2021-01-06 14:58:07,125 fail2ban.filter         [14255]: INFO    [sshd] Found 142.93.21.15 - 2021-01-06 14:58:06
2021-01-06 14:58:13,640 fail2ban.filter         [14255]: INFO    [sshd] Found 142.93.21.15 - 2021-01-06 14:58:13
2021-01-06 14:58:18,521 fail2ban.filter         [14255]: INFO    [sshd] Found 142.93.21.15 - 2021-01-06 14:58:18
2021-01-06 14:58:22,720 fail2ban.filter         [14255]: INFO    [sshd] Found 142.93.21.15 - 2021-01-06 14:58:22
2021-01-06 14:58:27,193 fail2ban.filter         [14255]: INFO    [sshd] Found 142.93.21.15 - 2021-01-06 14:58:27
2021-01-06 14:58:27,591 fail2ban.actions        [14255]: NOTICE  [sshd] Ban 142.93.21.15
```
7. Check the fail2ban service with fail2ban-client

```

root@fail2bandemo:/etc/fail2ban/jail.d# fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	5
|  `- File list:	/var/log/auth.log
`- Actions
   |- Currently banned:	1
   |- Total banned:	1
   `- Banned IP list:	142.93.21.15
root@fail2bandemo:/etc/fail2ban/jail.d# 

```
