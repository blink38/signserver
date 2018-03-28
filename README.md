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



## Configure Wildfly SSL

From our own PKI (see PKI.md), create a certificate for the wildfly service. The common name of the certificate must be the same as the FQDN you will use in the service url.

We have to create a PKCS12 certificate from wildfly private key and public certificate :

```shell
openssl pkcs12 -export -in /opt/pki/certificats/https.wildfly.crt -inkey /opt/pki/certificats/https.wildfly.key -out /opt/pki/certificats/wildfly.p12 -name wildfly -CAfile /opt/pki/certificats/ca.crt
```

I choose p12secret as export password.

Then create a keystore which will contains the wildfly SSL certificate :

```shell
mkdir /opt/wildfly/standalone/configuration/keystore
keytool -importkeystore -deststorepass keystorepwd  -destkeypass keystorepwd -destkeystore /opt/wildfly/standalone/configuration/keystore/wildfly.keystore.jks -srckeystore /opt/pki/certificats/wildfly.p12 -srcstoretype PKCS12 -srcstorepass p12secret -alias wildfly
```

I choose keystorepwd as keystore password. Be carefull, deststorepass and destkeypass must be the same. Otherwise wildfly will not be able to retrieve key file. Change srcstorepass with your p12 export password.

You can migrate the keystore as warning message tell you :

```shell
keytool -importkeystore -srckeystore /opt/wildfly/standalone/configuration/keystore/wildfly.keystore.jks -destkeystore /opt/wildfly/standalone/configuration/keystore/wildfly.keystore.jks -deststoretype pkcs12
```

Use keystore password choose previously (keystorepwd for me)

Now, we have to create the truststore file which will contains users certificate. They will be able to connect to HTTPs private area with they own certificate (SSL connection will verify the client certificate).

```shell
keytool -alias john.smith -import -file /opt/pki/certificats/user.john.crt -keystore /opt/wildfly/standalone/configuration/keystore/truststore.jks
```

I choose trustpwd as truststore password.

```
chown jboss.jobss /opt/wildfly* -R
```

We are now able to configure wildfly SSL. For that, we will use the jboss CLI (under user jboss). So return to the terminal in which you previously launch the jboss CLI.

(code starting with [standalone@localhost:9990 /] must be paste into jboss CLI)

```
[standalone@localhost:9990 /]
 /interface=http:add(inet-address="0.0.0.0")
/interface=httpspub:add(inet-address="0.0.0.0")
/interface=httpspriv:add(inet-address="0.0.0.0")
```

Secure the CLI by removing the http-remoting-connector from using the http port and instead use a separate port 4447. 

```
[standalone@localhost:9990 /]
/subsystem=remoting/http-connector=http-remoting-connector:remove
/subsystem=remoting/http-connector=http-remoting-connector:add(connector-ref="remoting",security-realm="ApplicationRealm")
/socket-binding-group=standard-sockets/socket-binding=remoting:add(port="4447")
/subsystem=undertow/server=default-server/http-listener=remoting:add(socket-binding=remoting)
:reload
```

```
[standalone@localhost:9990 /]
/socket-binding-group=standard-sockets/socket-binding=httpspriv:add(port="8443",interface="httpspriv")
/core-service=management/security-realm=SSLRealm:add()
/core-service=management/security-realm=SSLRealm/server-identity=ssl:add(keystore-path="keystore/wildfly.keystore.jks", keystore-relative-to="jboss.server.config.dir", keystore-password="keystorepwd", alias="wildfly")
/core-service=management/security-realm=SSLRealm/authentication=truststore:add(keystore-path="keystore/truststore.jks", keystore-relative-to="jboss.server.config.dir", keystore-password="trustpwd")
/subsystem=undertow/server=default-server/https-listener=httpspriv:add(socket-binding="httpspriv", security-realm="SSLRealm", verify-client=REQUIRED)
```

Pay attention to the third and the fourth command : 

3. keystore-password must be the password you choose to use when creating the wildfly.keystore.jks file. Alias is the alias you choose when importing the wildfly SSL certificate.
4. keystore-password must be the password you choose to use when creating the truststore.jks file.

Set-up the public SSL port which doesn't require the client certificate

```
[standalone@localhost:9990 /]
/socket-binding-group=standard-sockets/socket-binding=httpspub:add(port="8442",interface="httpspub")
/subsystem=undertow/server=default-server/https-listener=httpspub:add(socket-binding="httpspub", security-realm="SSLRealm")
```

Then reload all the configuration change

```
reload
```

Hope you do not have red message in wildfly console.

Port 8443 and 8442 must be open. Command netstat -taonp will help you to check for it.


Fix web service problem in JBoss AS 7/EAP 6/WildFly

Configure WSDL web-host rewriting to use the request host. Needed for webservices to work correctly when requiring client certificate.

```
[standalone@localhost:9990 /]
/subsystem=webservices:write-attribute(name=wsdl-host, value=jbossws.undefined.host)
/subsystem=webservices:write-attribute(name=modify-wsdl-address, value=true)
```
If the server is not so fast, we have to wait a little before we can reload, otherwise it will be bad

```
[standalone@localhost:9990 /]
:reload
```

## SignServer deployment

Copy conf/signserver_deploy.properties.sample to conf/signserver_deploy.properties and open it for editing. 

```shell
cp /opt/signserver/conf/signserver_deploy.properties.sample /opt/signserver/conf/signserver_deploy.properties
vi /opt/signserver/conf/signserver_deploy.properties
```

Uncomment lines :

```
appserver.home=${env.APPSRV_HOME}
database.name=mysql
```

```shell
chown jboss.jboss /opt/signserver* -R
```

As user jboss, deploy configuration :

```
cd /opt/signserver
./bin/ant deploy
```

Check signserver version :

```shell
bin/signserver getstatus brief all
Current version of server is : SignServer EE 4.0.0
```

## SignServer workers 

### Crypto Worker

This work will manage key used to sign. It will store key as p12 (in order to have private and public key).

We will create a store with a generic key pair.

Create the keystore including the generic certificate :

```shell
mkdir /opt/signserver/myconfig
keytool -importkeystore -deststorepass keystorepwd  -destkeypass keystorepwd -destkeystore /opt/signserver/myconfig/worker.crypto.keystore.jks -srckeystore /opt/pki/certificats/crypto.generic.p12 -srcstoretype PKCS12 -srcstorepass generic -alias generic
keytool -importkeystore -srckeystore /opt/signserver/myconfig/worker.crypto.keystore.jks -destkeystore /opt/signserver/myconfig/worker.crypto.keystore.jks -deststoretype pkcs12
```

Create the properties file /opt/signserver/myconfig/worker-crypto.properties :

```
# Type of worker
WORKERGENID1.TYPE=CRYPTO_WORKER

# This worker will not perform any operations on its own and indicates this by
# using the worker type CryptoWorker
WORKERGENID1.IMPLEMENTATION_CLASS=org.signserver.server.signers.CryptoWorker

# Uses a soft keystore:
WORKERGENID1.CRYPTOTOKEN_IMPLEMENTATION_CLASS=org.signserver.server.cryptotokens.KeystoreCryptoToken

# Name for other workers to reference this worker:
WORKERGENID1.NAME=MyCryptoTokenP12

# Type of keystore
# PKCS12 and JKS for file-based keystores
# INTERNAL to use a keystore stored in the database (tied to the crypto worker)
WORKERGENID1.KEYSTORETYPE=PKCS12

# Path to the keystore file (only used for PKCS12 and JKS)
WORKERGENID1.KEYSTOREPATH=/opt/signserver/myconfig/worker.crypto.keystore.jks

# Optional password of the keystore. If specified the token is "auto-activated".
WORKERGENID1.KEYSTOREPASSWORD=keystorepwd

# Optional key to test activation with. If not specified the first key found is
# used.
WORKERGENID1.DEFAULTKEY=generic
```

Defaultkey is the name of the generic certificate imported in the keystore  (-alias generic)

Import the configuration into SignServer :

```shell
cd /opt/signserver
./bin/signserver setproperties /opt/signserver/myconfig/worker
```

Check worker status :

```shell
bin/signserver getstatus brief all
Current version of server is : SignServer CE 4.0.0

Status of CryptoWorker with id 1 (MyCryptoTokenP12) is:
   Worker status : Active
   Token status  : Active
```

