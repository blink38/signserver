# My Installation of SignServer (open source edition)

I will describe here, step by step, my installation of SignServer 4.0.0 (https://www.signserver.org/) on GNU/Linux Debian 9. When writing this document, I used Debian 9.4

So let's start and assume you have an up-to-date operating system. We will install WildFly as application server and for testing we will create our own PKI using openssl tool. We will also using MariaDB as database server.

Official install guide is https://www.signserver.org/doc/current/manual/installguide.html


## Installation of prerequisites

```shell
apt-get install unzip openjdk-8-jdk ant
```
Do not install MariaDB from debian repository, because we need to use at least version 10.2.2 (otherwise index key prefix length is will be a problem https://stackoverflow.com/questions/45822688/key-was-too-long-in-mariadb-but-same-script-with-same-encoding-works-on-mysql).

## User JBoss

We will create a jboss user which will execute the application server WildFly.

```shell
addgroup jboss
adduser  --disabled-password --home /home/jboss --shell /bin/bash --ingroup jboss jboss
``` 

I do not configure password for this account, not needed. I will just add my SSH key to his authorized_keys file :

```shell
mkdir /home/jboss/.ssh
vi /home/jboss/.ssh/authorized_keys
chown jboss.jboss /home/jboss/.ssh -R
chmod og-r /home/jboss/.ssh/authorized_keys 
```

When editing authorized_keys file, just add your SSH key.


## Install SignServer

Download SignServer 4.0.0 archive (binary edition) : https://www.signserver.org/download.html

```shell
wget https://sourceforge.net/projects/signserver/files/signserver/4.0/signserver-ce-4.0.0-bin.zip
```

And unzip it in /opt directory

```shell
unzip signserver-ce-4.0.0-bin.zip -d /opt
ln -s /opt/signserver-ce-4.0.0 /opt/signserver
chown jboss.jboss /opt/signserver* -R
```

## Install WildFly server

From WildFly download page http://wildfly.org/downloads/, get the 9.0.2.Final version :

```shell
wget http://download.jboss.org/wildfly/9.0.2.Final/wildfly-9.0.2.Final.tar.gz
```

Unpack archive in /opt directory

```shell
tar xfz wildfly-9.0.2.Final.tar.gz -C /opt
ln -s /opt/wildfly-9.0.2.Final /opt/wildfly
chown jboss.jboss /opt/wildfly* -R
```

Edit jboss .bashrc to add environment variables :

```shell
vi /home/jboss/.bashrc
```

and add at the end of the file :

```
export APPSRV_HOME=/opt/wildfly
export SIGNSERVER_NODEID=node1
```


## Install MariaDB server

We need to install MariaDB server version >= 10.2.2. It was not possible to install such mariadb version from debian repository. 

From https://mariadb.com/downloads/mariadb-tx, download current debian version (release = current / version = 10.2.14 GA / OS = debian / OS Version = Debian 9 (Stretch))

```shell
wget https://downloads.mariadb.com/MariaDB/mariadb-10.2.14/repo/debian/mariadb-10.2.14-debian-stretch-amd64-debs.tar
```

Unpack and install :

```shell
tar xf mariadb-10.2.14-debian-stretch-amd64-debs.tar
cd mariadb-10.2.14-debian-stretch-amd64-debs
./setup_repository
apt-get update && apt-get install mariadb-server
```

Create database :

```shell
mysql -u root -p mysql
```

```sql
create database signserver;
grant all on signserver.* to "signserver"@"localhost" identified by "signserver";
flush privileges;
quit
```

We need to install MariaDB java connector to wildfly. From https://mariadb.com/downloads/mariadb-tx/connector, download the connector (type = database client / language = java / version = 2.2.3 GA)

```shell
wget https://downloads.mariadb.com/Connectors/java/connector-java-2.2.3/mariadb-java-client-2.2.3.jar
mkdir -p /opt/wildfly/modules/system/layers/base/org/mariadb/main/
mv mariadb-java-client-2.2.3.jar /opt/wildfly/modules/system/layers/base/org/mariadb/main/
```

Create the file module.xml 

```shell
vi /opt/wildfly/modules/system/layers/base/org/mariadb/main/module.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.0" name="org.mariadb">
  <resources>
      <resource-root path="mariadb-java-client-2.2.3.jar"/>
  </resources>
  <dependencies>
      <module name="javax.api"/>
      <module name="javax.transaction.api"/>
  </dependencies>
</module>
```

```shell
chown jboss.jboss -R /opt//wildfly*
``` 

## Start WildFly

Connect as user jboss. From root, use sudo, or connect via SSH. I recommand you to use another terminal, because we will launch wildfly ni foreground (and be able to see all log information).

```shell
sudo -u jboss -s
```

And start standalone wildfly 
```shell
cd /opt/wildfly
./bin/standalone.sh
```

Open another terminal with jboss user connected in, and launch jboss CLI :

```shell
cd /opt/wildfly
./bin/jboss-cli.sh
```

then connect (just typing connect after [disconnected /] prompt)

```
You are disconnected at the moment. Type 'connect' to connect to the server or 'help' for the list of supported commands.
[disconnected /]  connect
[standalone@localhost:9990 /] 
```

Register the mariadb connector (in jboss CLI) :

```
[standalone@localhost:9990 /] /subsystem=datasources/jdbc-driver=org.mariadb.jdbc.Driver:add(driver-name=org.mariadb.jdbc.Driver,driver-module-name=org.mariadb,driver-xa-datasource-class-name=org.mariadb.jdbc.MySQLDataSource)
{"outcome" => "success"}
[standalone@localhost:9990 /] :reload
{
    "outcome" => "success",
    "result" => undefined
}
```

And then configure database  (in jboss CLI) :

In this command, you can see the mariadb username, password and database name we used when create database. Change here if you choose another values.

```
[standalone@localhost:9990 /] data-source add --name=signserverds --driver-name="org.mariadb.jdbc.Driver" --connection-url="jdbc:mysql://127.0.0.1:3306/signserver" --jndi-name="java:/SignServerDS" --use-ccm=true --driver-class="org.mariadb.jdbc.Driver" --user-name="signserver" --password="signserver" --validate-on-match=true --background-validation=false --prepared-statements-cache-size=50 --share-prepared-statements=true --min-pool-size=5 --max-pool-size=150 --pool-prefill=true --transaction-isolation=TRANSACTION_READ_COMMITTED --check-valid-connection-sql="select 1;"
[standalone@localhost:9990 /] shutdown
```

Last shutdown command will stop wildfly application server. Just restart it in first jboss terminal with standalone.sh script.



