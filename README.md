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
openssl pkcs12 -export -in /opt/pki/certificats/https.wildfly.crt -inkey /opt/pki/certificats/https.wildfly.key -out /opt/pki/certificats/wildfly.p12 -name wildfly -chain -CAfile /opt/pki/certificats/ca.crt
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

Activate the CrytoToken worker : 

```shell
bin/signserver activatecryptotoken 1
```

### XML Signer

Create the file /opt/signserver/myconfig/worker-xmlsigner.properties

```
## General properties
WORKERGENID1.TYPE=PROCESSABLE
WORKERGENID1.IMPLEMENTATION_CLASS=org.signserver.module.xmlsigner.XMLSigner
WORKERGENID1.NAME=XMLSigner
WORKERGENID1.AUTHTYPE=NOAUTH

# Crypto token
WORKERGENID1.CRYPTOTOKEN=MyCryptoTokenP12

# Using key from sample keystore
WORKERGENID1.DEFAULTKEY=signer00003
# Key using ECDSA
WORKERGENID1.DEFAULTKEY=generic

WORKERGENID1.KEYSTOREPASSWORD=keystorepwd
```

and import it :

```shell
cd /opt/signserver
./bin/signserver setproperties myconfig/worker-xmlsigner.properties
```

Check status and retrieve worker id (ie 2)

```shell
./bin/signserver getstatus brief all
```
You must see Worker status and Token status = Active. Hoping so.

To test XMLSigning :

```shell
./bin/signclient  signdocument -workername XMLSigner -infile /opt/signserver/res/signingtest/input/test.xml 
<?xml version="1.0" encoding="UTF-8"?><root>
	<my-tag>My Data</my-tag>
<Signature xmlns="http://www.w3.org/2000/09/xmldsig#"><SignedInfo><CanonicalizationMethod Algorithm="http://www.w3.org/TR/2001/REC-xml-c14n-20010315#WithComments"/><SignatureMethod Algorithm="http://www.w3.org/2000/09/xmldsig#rsa-sha1"/><Reference URI=""><Transforms><Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/></Transforms><DigestMethod Algorithm="http://www.w3.org/2000/09/xmldsig#sha1"/><DigestValue>2oTQryPaqodno7SeYQjQSMIBiP0=</DigestValue></Reference></SignedInfo><SignatureValue>FqEphQrrYu[...]XCo=</SignatureValue><KeyInfo><X509Data><X509Certificate>MII[...]6C</X509Certificate></X509Data></KeyInfo></Signature></root>2018-03-28 15:14:55,064 INFO  [SignDocumentCommand] Wrote null.
2018-03-28 15:14:55,065 INFO  [SignDocumentCommand] Processing test.xml took 356 ms.
```

### PDF Signer

Create the file [/opt/signserver/myconfig/worker-pdfsigner.properties](myconfig/worker-pdfsigner.properties)

```
## General properties
WORKERGENID1.TYPE=PROCESSABLE
WORKERGENID1.IMPLEMENTATION_CLASS=org.signserver.module.pdfsigner.PDFSigner
WORKERGENID1.NAME=PDFSigner
WORKERGENID1.AUTHTYPE=NOAUTH

# Crypto token
WORKERGENID1.CRYPTOTOKEN=MyCryptoTokenP12
#WORKERGENID1.CRYPTOTOKEN=CryptoTokenP11

# Using key from sample keystore
WORKERGENID1.DEFAULTKEY=generic
# Key using ECDSA
#WORKERGENID1.DEFAULTKEY=signer00002


## PDFSigner properties

#--------------------------SIGNATURE PROPERTIES--------------------------------------#

# specify reason for signing. it will be displayed in signature properties when viewed
# default is "Signed by SignServer"
#WORKERGENID1.REASON=Signed by SignServer
WORKERGENID1.REASON=Arts et Metiers signed document

# specify location. it will be displayed in signature properties when viewed
# default is "SignServer"
#WORKERGENID1.LOCATION=SignServer
WORKERGENID1.LOCATION=Paris

# digest algorithm used for the message digest and signature (this is optional and defaults to SHA1)
# the algorithm determines the minimum PDF version of the resulting document and is documented in the manual.
# for DSA keys, only SHA1 is supported
#WORKERGENID1.DIGESTALGORITHM=SHA256
WORKERGENID1.DIGESTALGORITHM=SHA1

#--------------------------SIGNATURE VISIBILITY--------------------------------------#

# if we want the signature to be drawn on document page set ADD_VISIBLE_SIGNATURE to True , else set to False
# default is "False"
#WORKERGENID1.ADD_VISIBLE_SIGNATURE = False
WORKERGENID1.ADD_VISIBLE_SIGNATURE = True

# specify the page on which the visible signature will be drawn
# this property is ignored if ADD_VISIBLE_SIGNATURE is set to False
# default is "First"
# possible values are :
        # "First" : signature drawn on first page of the document,
        # "Last"  : signature drawn on last page of the document,
        # page_number : signature is drawn on a page specified by numeric argument. If specified page number exceeds page count of the document ,signature is drawn on last page
        # if page_number specified is not numeric (or negative number) the signature will be drawn on first page
WORKERGENID1.VISIBLE_SIGNATURE_PAGE = 2

# specify the rectangle signature is going to be drawn in
# this property is ignored if ADD_VISIBLE_SIGNATURE is set to False
# defailt is "400,700,500,800"
# format is : (llx,lly,urx,ury). Here llx =left lower x coordinate, lly=left lower y coordinate,urx =upper right x coordinate, ury = upper right y coordinate
#WORKERGENID1.VISIBLE_SIGNATURE_RECTANGLE = 400,700,500,800

# if we want the visible signature to contain custom image , specify image as base64 encoded byte array
# alternatively custom image can be specified by giving a path to image on file system
# note : if specifying a path to an image "\" should be escaped ( thus C:\photo.jpg => "C:\\photo.jpg" )
# note : if specifying image as base64 encoded byte array "=" should be escaped (this "BBCXMI==" => "BBCXMI\=\=")
# if both of these properties are set then VISIBLE_SIGNATURE_CUSTOM_IMAGE_BASE64 will take priority
# if we do not want this feature then do not set these properties
# default is not set (no custom image)
# these properties are ignored if ADD_VISIBLE_SIGNATURE is set to False
#WORKERGENID1.VISIBLE_SIGNATURE_CUSTOM_IMAGE_BASE64=
#WORKERGENID1.VISIBLE_SIGNATURE_CUSTOM_IMAGE_PATH=
WORKERGENID1.VISIBLE_SIGNATURE_CUSTOM_IMAGE_BASE64=iVBORw0KGgoAAAANSUhEUgAAAKwAAAA+CAYAAAC2n/CQAAAAAXNSR0IArs4c6QAAAAZiS0dEAP8A/wD/oL2nkwAAAAlwSFlzAAALEwAACxMBAJqcGAAAAAd0SU1FB9gLFxIWBfv3XJMAAAm/SURBVHhe7Zx5jF5VGYef33Sb0r0UWhAEC6UlQKMVSAWpUlogAgbCUlpZWpZWQFyCwYBR6oYIWI0IjqUJspSlIagJEmoQEE0RKNQCgi0YEC2WzU5XbKHz+sd7bufON3f5ZqYz/b72PMnNXc7v3Hvnznu297zng0gkEolEIpFIJBKJRCKRSKTuEDMWNAKNwHpga4amF9AnHDcCG4DewP9SmkbgAzx/L9qy1RZeSCSyPegNDAX+UybsAn1xY45EukwDsLZM1EWGlAkikWppAN4vE0UitUID0L9MFInUCj1Rw24oE0Qi1dITNezAMkEkUi0NZYJIpJboiS5BJLLd6IkadlOZIBKplp7ow+5ZJohEqqUnatjoJYhsN3rCYGOXILLd6IlB125lgnpAUj9Jj4ft5DJ9pHsQMxb0pxtrwTOPbiqTVE2DWZ97L1v6YeV1SdcCF2Rk2QK8AbwGLAEWmFmnAnEkpb/THDObX6SvdSRdB8wMp2PNLDemRNJ3gdnh9AXgdDNbl6fvTnqXCWqKBmvJSRkMjMxJ2xc4GjgHuELSxWb2WI52VyL9zTK7hpIE/AT4Sri0FJi2o4wV6s1g3aOxsURzHh7bCx7aeABwBjAhHN8v6TAzezMnfx5bgFnheEmRcGdAUgMwH0iCmf8EnLwjjRXqz2DLjBXgITN7L31B0o+Am4EvAsOBG4EZGXlzMbOtwK/KdDsDknoDdwJnh0uLgdPMrLvHO6XUm8F2CjNrkXQFMA0YBhxVpJc0CtgfX0mxrKh/V4akvsA4vKCsMrNXcnR7AqPxQfDfzKxdX70ISQOAQ/BCvaKj+RMk9QMWAZ8Plx4AppvZlvxcjqTd8Hd4H3+HTo0XiugJt1ZNYGab8AEDwH6SBkmaIqk5bAdKGi/pCeBN4EngMeAE8EFXSnt+1jMkPRDSF8n5BrAKWB7utVLSc5ImpPKMlvQgsBp/5l+B9ZJ+IG+Wc5HUS9JVkl7Fu0FPAS8CGyU9IWlcUf5K5Eb/IK3GeidwVpGxSmqQ9HVJK/F3eBr/zhslLZF0aEaeCalveXpleoZ+cdA+0hNurVoiPWhrwNeqDQnbROAvwDGAgOYKPSltP7IZGNIHALcC1wEj8CVIyRq4TwC/l7S/pIPxZ56ELyNK+tWNwNXhHplI2g94HLgW75u3ACvxZ/XF/45nJU3Pu0caSUPwpn9KuNQEnB+6Qnl59gH+ANwAjAmXX8ELaR/gU8AzqijgZvYc8E/8W82hgFDojg/aRfVWww4oE+QhqQ/eXIE3zZXN/C3AZuBcYISZDcOb8T/TcY7BByvzgT3MbG/cmC/DDWt33JgXhvNTgIFm9hFgL9wQAWZKSgxhG5KEN9ufxgvWl0P+seFZ44BncB/4rZL2rrxHBSOAR3FvCsANZnaJmVlBHoC7gc8C64Ar8Hc4yMz2wQ34SbzwNYUClibxd06R9DHyuTjsNwD31JvBdqU1uAbYIxw/k5HeHzjKzO5KBm1mtrYT3gSAQcB8M5tjZu+CD9rM7Bbg/qCZBhwGHGtmDyb9PTNbDXwBr3EbyPYvnwccCRg+GLrJzLatYjazFcBkvKYbAPww4x5pFuNeFIBvm9mVRWIASdPwggnebZhnqUGZmb0KHAe8jhvt9RW3uAs3QgEXkYG8P53UzveY2fp6M9g8P2yawyVNDNskSbMkPQx8M6RvArL+Ibeb2csZ1zvDFuBbOWmLUseZzwyF5KlwOroyHZgb9veZ2eMZ6ZjZBloN9cxQK+eR1HDPU27cCXPD/jdmtjhLEAz4++H0NLn3IUlbj9fQALPSaSlOw1sj8NZqm5dgCO5I/jewT9hnNb9pt9KArbOuLw1sOXvl4DLJ9ubhgrQ1wCWWPVLP/Oid5HUzezsnbVXqeGmOBlp1H01flDQI92CAN+NFPBf2/fH/679ydC8ChwLjgYWSZlhx37UfcFA4rfYd+uAFI/3tm/AZtL3wfvxv22bd1h1YZmZLAXqHH7lYFzZwY4Vyn+fGlkcqa/lup4HqatmED2idmn0SmGdmzTnavH9mZ3ijIC3teC96ZqKrLPHpPu3XJF2EN6tJDarUlh4cjiH/eafgNf8RwFnAVknnFhjtAbR6mC6TdA7Zzxc+AEwYQ8pgzWyZpKfx7s1sUgYr6UDg2HC6bRq83vywA2n7D89ihFVMHFTJmjJBByj1WQaq1aVJD1AOzlW1Z2hB2lp8JP4I8ElgOm6055tlToen32FsRnoeWe/QhBvsiZL2NbOkUCUFcSOtXYe6M9iuDLrKKBsR1wrprsZ03C1WDXldFADMrFnSVLyJ/zgee/GhpAszjDZ9r5nAH6mOdzKu3QvMw435AuA7co/OzCTdUtPB9Waw8WePID1IG2lmr+cJO4qZrZE0BTfa8bjRbJUHDKUL9N9Tx6O68g5m9r6kO3DX3IWSvod3UZLAnDZRcfXmJdjlMXeTvRtOTyrSdobQnToOH4iB+5Ob0l6GMMJPxjqfo+skPtl9gRNpDWVcbmZPp4XRYOuTG8N+qqQkQCUXlUzxVhIKxWTgpXBpNvDzClnyDpMkzaSEoncIrr2kW3ENMDUct4s57tAfEqkZfgwsC8e3SZorDzBvgzw24mbau4tKMbN3cKNNuiCXSvpZSnITrb7iJnnsQztXqKRDQr4idyO01rJH4na5CZ8JbEO9GeywMsGugHkk1nQ8UKYRr5XWSnpJHijyvKT1eNDNpXTyFyTN7C3caFeES5dLmhfSWvAZuaW4++xqoFnSy+Edlktah3ctLqfYSwEeFZYelN1nGVFyHR50NX/m8FTne1C+MDBp2PAySSaXDzhrd+C/Zxz1Swg/qCx6Fc3W7FKY2QpJR+Bz+FficQ8H09bVtRb4HXBH+ztUh5mtljQZj28Yg/t+PzSzK83sH5Im4isSrsJjEsaFLWEd8BA+FZuLmW2RtBD4ariUuQRJVhrf0Ja2BlvOXcN2p1OIwV/69eL1RKpC0kjcUEYBb+GRX69ZN8Sk5iGP6R2Hz1y9Tes7VOVvlrQY9we/YGbjszQdrmF7kMqfno8UEJrvt8p03Yn5dHShvzeP0FocH05/kaertz5sZOflurB/D7g9T1TLNewoPNYzshMiqRFfMzYU9+VODkk/NV8dkkktG2xk52YwcFvFtUcpCW+MBhvZUWzG/cOGr2dbAtxtBWGNEA02soMIPtZTy3SVxEFXpK6IBhupK2rZYFeXCSK7HrVssJFIOzpssC0N1mgw2KCvgQq2vuYL3yKR7UaHvQTDH3t2M+6SKOMDYBWnnlCmi0SqpsMG2wlG03bl5ArKF65tVssuvxQmEolEIpFIJBKJRCKRSCQSiewc/B86YBbPvEtJfQAAAABJRU5ErkJggg\=\=
WORKERGENID1.VISIBLE_SIGNATURE_CUSTOM_IMAGE_PATH=C:\\Dokumanlar\\FOTO\\Photos\\15032009\\100_3801.JPG

# if we want our custom image to be resized to specified rectangle (set by VISIBLE_SIGNATURE_RECTANGLE) then set to True.
# if set to True image might look different that original (as an effect of resizing)
# if set to False the rectangle drawn will be resized to specified image's sizes.
# if set to False llx and lly coordinates specified by VISIBLE_SIGNATURE_RECTANGLE property will be used for drawing rectangle (urx and ury will be calculated from specified image's size)
# this property is ignored if ADD_VISIBLE_SIGNATURE is set to False or if custom image to use is not specified
# default is True
#WORKERGENID1.VISIBLE_SIGNATURE_CUSTOM_IMAGE_SCALE_TO_RECTANGLE = True

# to create a certifying signature that certifies the document set the CERTIFICATION_LEVEL
# possible values are: NOT_CERTIFIED, FORM_FILLING, FORM_FILLING_AND_ANNOTATIONS or NO_CHANGES_ALLOWED
# default is NOT_CERTIFIED
# WORKERGENID1.CERTIFICATION_LEVEL=NOT_CERTIFIED
#--------------------------SIGNATURE TIMESTAMPING--------------------------------------#

# if we want to timestamp document signature, specify timestamp authority url, if required bu tsa uncomment tsa username and password lines and specify proper values
# if we do not want to timestamp document signature , do not set property

# Worker ID or name of internal timestamp signer in the same SignServer
# Default: none
#WORKERGENID1.TSA_WORKER=TimeStampSigner

# URL of external timestamp authority
# note : if path contains characters "\" or "=" , these characters should be escaped (thus "\" = "\\", "=" =>"\=")
# default is not set (no timestamping)
# WORKERGENID1.TSA_URL =
#WORKERGENID1.TSA_URL=http://tsa.example.com:8080/signserver/tsa?workerName\=TSA

# if tsa requires authentication for timestamping , specify username and password
# if tsa does not require authentication, do not set these properties
# these properties are ignored if TSA_URL is not set (no timestamping)
# default is not set (tsa does not require authentication)
#WORKERGENID1.TSA_USERNAME=
#WORKERGENID1.TSA_PASSWORD=

#--------------------------EXTRA PROPERTIES [NOT TESTED YET]--------------------------------------#

#if we want to embedd the crl for signer certificate inside the signature package set to True, otherwise set to False
#default is False
#WORKERGENID1.EMBED_CRL = False

#if we want to embedd the ocsp responce for signer certificate inside the signature package set to True, otherwise set to False
#note : issuer certificate (of signing certificate) should be in certificate chain.
#default is False
#WORKERGENID1.EMBED_OCSP_RESPONSE = False
```

Import the configuration :

```shell
cd /opt/signserver
./bin/signserver setproperties myconfig/worker-pdfsigner.properties
```

To test XMLSigning :

```shell
./bin/signclient  signdocument -workername PDFSigner -infile /opt/signserver/res/signingtest/input/test.pdf  --outfile /tmp/test-signed.pdf
```

Open the test-signed.pdf file with Acrobat Reader.
