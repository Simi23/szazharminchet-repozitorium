---
title: Asterisk default settings
description: Here are the install and the settings for asterisk.
published: true
date: 2025-03-17T09:19:06.290Z
tags: linux
editor: markdown
dateCreated: 2025-03-17T09:15:46.348Z
---

# Asterisk

## Install and compile
Asterisk can be installed through installing the files for compile with `wget` and then compile them.
First install the neccessary packages for install and compile
```
apt install build-essential wget
```

Install the the version you want from Asterisk's official webiste, and then decompress the .tar.gz file:
```
cd /usr/src
sudo wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-22-current.tar.gz
sudo tar xvf asterisk-22-current.tar.gz
```

Compile and install Asterisk. After that create sample configuartion files (the command will create all 108 config files under `/etc/asterisk/` directory):
```
cd asterisk-22*
./configure --with-pjsip-bundled
make && make install
make samples && make config
```

## Configuration files
Important files:
- `/etc/asterisk/asterisk.conf` - Main configuration file
- `/etc/asterisk/pjsip.conf` - Phone configuration file
- `/etc/asterisk/extensions.conf` - Dial up plan configuration file

### `/etc/asterisk/asterisk.conf`
In this file the main configurations are stored. Here are the paths that are used by Asterisk.

### `/etc/asterisk/pjsip.conf`
In this file you can set up 'users' like endpoints phones etc.
Here is an example configuration:
```
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0

[6001]
type=endpoint
context=internal
disallow=all
allow=ulaw
auth=6001
aors=6001

[6001]
type=auth
auth_type=userpass
password=Passw0rd
username=6001

[6001]
type=aor
max_contacts=2

[6002]
type=endpoint
context=internal
disallow=all
allow=ulaw
auth=6002
aors=6002

[6002]
type=auth
auth_type=userpass
password=Passw0rd
username=6002

[6002]
type=aor
max_contacts=2
```

### `/etc/asterisk/extensions.conf`
In this file you can set dialplans, with numbers, etc. Generally you will edit this file.
Here is an example configuration:
```
[internal]
exten = 100,1,Answer()
same = n,wait(2)
same = n,Playback(hello-world)
same = n,Hangup()

exten = _60XX,1,Dial(PJSIP/${EXTEN},60)
```
You have to set up a context between '[]' marks, and after that you have to create 'plans', extensions for callable numbers. ([Read more here.](https://docs.asterisk.org/Configuration/Dialplan/Contexts-Extensions-and-Priorities/#dialplan-priorities))

The numbers in order in an extension:
```
exten = 100,1,Answer()
        ^   ^       ^
        |   |       |
     number |  application
            |
         priority
```
- number: The number of the extension that you will call
- priority: You can set up more priorty, and it will be played in order
- application: The apllication that will run in each priorty


After editing this file you can reload the dialplan by:
```
asterisk -rx "dialplan reload"
```

Patterns to use in extension files:
| character | meaning |
| :-: | :-: |
| x/X | a digit from 0 to 9 |
| z/Z | a digit from 1 to 9 |
| n/N | a digit from 2 to 9 |
| [123 - 4] | Numeric range |
| . | one or more chatacter  |
| ! | zero or more character |

## Call files

## Asterisk CLI
You can enter the Asterisk CLI by using `asterisk -rvvvv` command. You can use the cmdlets, and get the logs when something is happening through asterisk. It is useful because it highlit words. (Mostly it is the same as under `/var/log/asterisk/`, but the highlithting helps read the errors.)
