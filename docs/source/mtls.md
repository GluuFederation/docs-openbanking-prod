# Mutual TLS Configuration
Mutual TLS(MTLS) helps servers and clients in identifying each other. It also establishes a two-way encrypted channel in between the server and clients. This document discusses MTLS configuration in Gluu Open Banking Distribution. 

To support MTLS configuration the token_endpoint_auth_method client property in the server should have at least one of the following two new values:
* tls_client_auth: Indicates that client authentication to the authorization server will occur with mutual TLS utilizing the PKI method of associating a certificate to a client.
* self_signed_tls_client_auth: Indicates that client authentication to the authorization server will occur using mutual TLS with the client utilizing a self-signed certificate.

The following discussion is based on Ubuntu similar settings will be there for other distributions and operating systems. We need to add all the MTLS related settings in https_jans.conf file in the /etc/apache2/sites-available folder. The MTLS settings require the ssl module. To check if the ssl module is already installed, run below command. 

apachectl -M | grep ssl

Its output as “ssl_module (shared)” confirms that the ssl_module is installed.

Usually third party or Certbot SSL certificates are used for web server ssl connections. 

The following lines should be included in Apache configuration which is responsible for ssl connection. The paths of cert files could be different so refer the details of your distribution. Like for CentOS /etc/pki/tls is the path. /etc/ssl/ is for Debian based distributions. 

SSLEngine On
SSLCertificateFile /etc/letsencrypt/live/bank.gluu.org/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/bank.gluu.org/privkey.pem

We can define the URLs endpoints which should be processed using MTLS or without MTLS. The endpoints to be processed with MTLS get the client certificate under the X-ClientCert header. 
To add an endpoint under MTLS, use the following set of lines as a guideline: 

```
<LocationMatch /jans-auth/restv1/token>
        SSLVerifyClient require
        SSLVerifyDepth 10
        SSLOptions +StdEnvVars +StrictRequire +ExportCertData
        RequestHeader set X-ClientCert "%{SSL_CLIENT_CERT}s"
        ProxyPass http://localhost:8081/jans-auth/restv1/token
        ProxyPassReverse http://localhost:8081/jans-auth/restv1/token
        Order allow,deny
        Allow from all
</LocationMatch>
```
We define the Open Banking (OB) trusted and revoked certificates by setting following properties in https_jans.conf file. 

```
     SSLCACertificateFile /etc/apache2/certs/matls.pem
     SSLCARevocationFile revoked.pem
     SSLCARevocationCheck chain no_crl_for_cert_ok  
     SSLCARevocationPath /etc/apache2/certs/revoke/
```

Open banking periodically (every 24 hours) updates the list of revoked certificates. The above lines indicates that the mtls.pem file has the valid OB certificates and the revoked.pem has the revoked certificates. 

SSLCARevocationCheck directive enables the certificate revocation list (CRL) and the with no_crl_for_cert_ok mod_ssl succeeds when no CRL(s) for the checked certificate(s) were found in the locations defined with SSLCARevocationFile and SSLCARevocationPath.

