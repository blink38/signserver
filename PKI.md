## PKI

To be able to do full test, I need certificat signed by a CA. For testing, I prefered to create my own PKI infrastructure to sign all certificates I need.

I will used openssl to create this PKI. (http://artisan.karma-lab.net/creer-sa-propre-mini-pki)


# Create PKI directories and configuration
```shell
mkdir /opt/pki
mkdir -p /opt/pki/db/ca.db.certs
mkdir /opt/pki/config
mkdir /opt/pki/certificats
echo '01' > /opt/pki/db/ca.db.serial
touch /opt/pki/db/ca.db.index
```

create PKI configuration file /opt/pki/config/ca.config

```
[ ca ]
default_ca      = CA_own

[ CA_own ]
dir             = /opt/pki/db
certs           = /opt/pki/db
new_certs_dir   = /opt/pki/db/ca.db.certs
database        = /opt/pki/db/ca.db.index
serial          = /opt/pki/db/ca.db.serial
RANDFILE        = /opt/pki/db/ca.db.rand
certificate     = /opt/pki/certificats/ca.crt
private_key     = /opt/pki/certificats/ca.key
default_days    = 3000
default_crl_days = 30
default_md      = md5
preserve        = no
policy  = policy_anything

[ policy_anything ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

# Generate CA key and certificates

generate CA private key

```shell
openssl genrsa -des3 -out /opt/pki/certificats/ca.key 1024
```

I choose pass phrase = pkipassword

generate CA certificate 

```shell
openssl req -utf8 -new -x509 -days 3000 -key /opt/pki/certificats/ca.key -out /opt/pki/certificats/ca.crt
```
You will be prompt for CA private key password (remember, I choose pkipassword).

generate DER certificate 

```shell
openssl x509 -in /opt/pki/certificats/ca.crt -outform DER -out /opt/pki/certificats/ca.der
```
 
 
 # Create our first certificate
 
 We will create our first certificate for a web service (wildfly)
 
 First create key :
 
 ```shell
 openssl genrsa -out /opt/pki/certificats/https.wildfly.key 1024
 ```
 
 Create certificate request
 
 ```shell
 openssl req -days 365 -new -key /opt/pki/certificats/https.wildfly.key -out /opt/pki/certificats/https.wildfly.csr
```

(challenge password is not required)
