# AUTHTYPE = CLIENTCERT

You can force authentication when using a worker, for example with client certificate : to be able to use a worker, client must offer a valid and authorized certificate.

To do so, AUTHTYPE of a worker need to be set to CLIENTCERT.

(if worker id is 4) 

```shell
./bin/signserver setproperty 4 AUTHTYPE=CLIENTCERT
./bin/signserver reload 4
```

Then add authorized client : you need to know the certificate serial number of the client and the DN of the Issuer certificate. Be carrefull, it is not the DN client certificate, it's the DN of the CA certificate who sign the client certificate.

```shell
./bin/signserver addauthorizedclient 4 6654FDZ3F4 "E=email@domain.com,CN=The CA Infra,OU=IT,O=CA Company,L=NewYork,C=US"
```

When connnecting to signserver https secure web interface (the one which need to valide your client certificate), you will be able to test your worker with the demo interface.
