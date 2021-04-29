# VM Installation Guide

## VM Preparation

Prepare a VM with the following minimum specs:

- 4 GB RAM
- 2 GB swap space
- 2 CPU units
- 50 GB disk space

The VM must have a static IP address and a resolvable hostname. A fully qualified domain name (FQDN) is required for production deployments.

The Gluu Open Banking Identity Platform can be installed on one of the following supported operating systems:

- Ubuntu 20
- CentOS 8

## Prerequisites

Prior to installation, the following are needed:

- `curl`
- `Python 3.x`
- [MySql](https://www.mysql.com/)

!!! Important
    MySQL can be installed either on the same machine or remotely. It will be configured for the Gluu Open Banking Identity Platform upon setup.

## Installation 

To install the Gluu Open Banking Identity Platform, just follow these steps:

1. Download the installation script:

    `curl https://raw.githubusercontent.com/JanssenProject/jans-setup/openbank/install.py > install.py`

1. Execute the installation script:

    `python3 install.py`

1. The installation script will install required programs and packages and then will prompt the user for setup instructions. Answer the following questions:

### Certificate Generation Setup

| Prompt | Description |
| ------ | ----------- |
| Enter IP Address | The IP address for the VM. **Use an IP address assigned to one of this server's network interfaces (usage of addresses assigned to loopback interfaces is not supported)** |
| Enter Hostname | The hostname for the VM. Recommended to be a FQDN |
| Enter your city or locality | Used to generate X.509 certificates. |
| Enter your state or province two letter code | Used to generate X.509 certificates. |
| Enter two letter Country Code | Used to generate X.509 certificates. |
| Enter Organization Name | Used to generate X.509 certificates. |
| Enter email address for support at your organization | Used to generate X.509 certificates.|

### Architecture Setup

| Prompt | Description |
| ---- | --------- |
| Enter maximum RAM for applications in MB | Maximum  |
| RDBM Type | Backend type. Currently only MySQL is supported |
| Use remote RDBM | Select if connecting to an external MySQL server |
| MySQL Host | Remote RDBM: Hostname of RDBM Server| 
| MySQL Port | Remote RDBM : Port for RDBM Server|
| Jans Database | |
| Jans Database Username | |
| Jans Database Password | |
| Use external key | |


### Uninstalling Janssen Server

Execute the installation script with the `-uninstall` argument

`python3 install.py -uninstall`

## MTLS Configuration

For MTLS, OBIE-issued certificates and keys should be used. The following discussion assumes that the file `ca.crt` has a CA certificate and `ca.key` has a CA private key. 

The following command is an example of how to create the server’s private key (`server.key`), Certificate Signing Request (CSR) (`server.csr`) and certificate (`server.crt)`:

```
$ openssl genrsa -out server.key 2048
$ openssl req -new -key server.key -out server.csr
$ openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -set_serial 100 -days 365 -outform PEM -out server.crt
```

Now, store the server key (`server.key`) and certificate (`server.crt`) file in some location (preferably inside `/etc/certs`) and set its path in the `.conf` file (`/etc/apache2/sites-enabled/https_jans.conf`) with  `SSLCertificateFile` and  S`SLCertificateKeyFile` directives as:

```
	SSLCertificateFile /etc/certs/bankgluu/server.crt
	SSLCertificateKeyFile /etc/certs/bankorg/server.key
```

The path of CA certificate file should be set to SSLCACertificateFile directive as:

```
	SSLCACertificateFile /etc/apache2/certs/matls.pem    
```

The following commands will create client’s private key (`client.key`), CSR (`client.csr`) and certificate (`client.crt`):

```
$ openssl genrsa -out client.key 2048
$ openssl req -new -key client.key -out client.csr
$ openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -set_serial 101 -days 365 -outform PEM -out client.crt
```

The following command will create a client certification chain (private key, public certificate and ca certificate) into the file `client.pem`:

```
$ cat client.key client.crt ca.crt >client.pem
```

Use this pem file to create JWKs for the clients (if required). To create a JWK, you can use a free utility published at [https://mkjwk.org](https://mkjwk.org). Or you can download the command-line tool from [GitHub](https://github.com/mitreid-connect/json-web-key-generator). There are numerous other online PEM-to-JWKS tools available like [JWKConvertFunctions](https://8gwifi.org/jwkconvertfunctions.jsp). We may need to add/update some data in these generated JWKs.

!!!Note 
    It is important to give different values of the Common Name field (“Common Name (e.g. server FQDN or YOUR name) []”) for the CA, Server and  clients. Other fields may have common values but the same values for Common Name of all certificates results in certificate verification failed at runtime.

### Importing the CA and signing certificates into Auth-Server keystore: 

The command line utility keytool is installed with JDK, it can be used to import the certificate and keys into the JVM keystore and jans-auth server’s keystore.

```
./keytool -importcert -file /path/to/file/filename.cer -keystore /path/to/file/filename.jks -alias yourkeystore

./keytool -importkeystore -srckeystore /path/to/file/filename.jks -srcstoretype JKS -destkeystore /opt/jre/lib/security/cacerts -deststoretype JKS
```

## Accessing the Platform
