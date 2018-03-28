# My Installation of SignServer (open source edition)

I will describe here, step by step, my installation of SignServer 4.0.0 (https://www.signserver.org/) on GNU/Linux Debian 9. When writing this document, I used Debian 9.4

So let's start and assume you have an up-to-date operating system. We will install WildFly as application server and for testing we will create our own PKI using openssl tool. We will also using MariaDB as database server.

Official install guide is https://www.signserver.org/doc/current/manual/installguide.html


## Installation of prerequisites

```
apt-get install unzip openjdk-8-jdk ant
```
Do not install MariaDB from debian repository, because we need to use at least version 10.2.2 (otherwise index key prefix length is will be a problem https://stackoverflow.com/questions/45822688/key-was-too-long-in-mariadb-but-same-script-with-same-encoding-works-on-mysql).

## User JBoss

We will create a jboss user which will execute the application server WildFly.

```
addgroup jboss
adduser  --disabled-password --home /home/jboss --shell /bin/bash --ingroup jboss jboss
``` 

I do not configure password for this account, not needed. I will just add my SSH key to his authorized_keys file :

```
mkdir /home/jboss/.ssh
vi /home/jboss/.ssh/authorized_keys
chown jboss.jboss /home/jboss/.ssh -R
chmod og-r /home/jboss/.ssh/authorized_keys 
```

When editing authorized_keys file, just add your SSH key.


## Install SignServer

Download SignServer 4.0.0 archive (binary edition) : https://www.signserver.org/download.html

```
wget https://sourceforge.net/projects/signserver/files/signserver/4.0/signserver-ce-4.0.0-bin.zip
```

And unzip it in /opt directory

```
unzip signserver-ce-4.0.0-bin.zip -d /opt
ln -s /opt/signserver-ce-4.0.0 /opt/signserver
chown jboss.jboss /opt/signserver* -R
```
