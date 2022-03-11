# Pushed Authorization Requests(PAR) and JWT Secured Authorization Response Mode(JARM)
This section covers details of two important features required by openbanking ecosystem. The Gluu Open Banking Identity Platform 
supports [PAR](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-par-07) and [JARM](https://openid.net/specs/openid-financial-api-jarm-ID1.html) specification.  
These two features are bundled in the installation so when you install the Gluu Openbanking Identity Platform the Authorization 
Server(AS) will support these features by default. The older/existing installation may require to update WAR/ image. Moreover, these features are also [FAPI certified for Brazil Open Banking (Based on FAPI 1 Advanced Final).](https://openid.net/certification/#FAPI_OPs)

## Pushed Authorization Requests-PAR:
PAR are handled by an additional endpoint of Authorization Server (AS). Clients POST their authorization parameters to this endpoint, 
in return the clients gets a reference (named as request URI value) that will be used in further authorization requests by the client. 
PAR enables the OAuth clients to push the payload of an authorization request directly to the authorization server in exchange for a 
request URI value. This request URI value is used as reference to the authorization request payload data in a subsequent call to the 
authorization endpoint.
    We can set different PAR lifetime for the different clients. PAR lifetime will be 600 seconds if it is unspecified.

We have a new configuration property for PAR as:

    *     parEndpoint - String, corresponds to pushed_authorization_request_endpoint 
          as defined by specification.

Moreover, there are two new client configuration as:

    *     parLifetime: An integer parameter representing the lifetime (in seconds) of 
          the pushed authorization request. 
    *     requirePar - Boolean parameter indicating whether the only means of 
          initiating an authorization request the client is allowed to use is a 
          pushed authorization request. If omitted, the default value is "false".

## JWT Secured Authorization Response Mode-JARM
   
This is a new JWT-based response mode to encode authorization responses known as JARM, (see [Financial-grade API: JWT Secured Authorization Response Mode for OAuth 2.0](https://openid.net/specs/openid-financial-api-jarm-ID1.html)). 
Here clients are enabled to request the transmission of the authorization response parameters along with additional data in JWT format. 
This mechanism enhances the security of the standard authorization response since it adds support for signing and encryption,sender authentication, 
audience restriction. It also provides protection from replay, credential leakage, and mix-up attacks. It can be combined with any response type.

For this feature AS supports new response modes (`query.jwt`,`fragment.jwt`,`form_post.jwt`,`jwt`) and additional signing, encryption algorithms. 
