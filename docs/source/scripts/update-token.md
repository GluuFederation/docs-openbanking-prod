# Update Token Interception Script

This script can be used to add custom claims to the id_token. As per the open banking standard, the token should contain claim openbanking_intent_id and the same value should also reflect in the sub claim. A similar expectation is required to be fulfilled by the FAPI-RW standard where the sub claim should have the user id.

## Configuration Prerequisites
- A Janssen Authorization Server installation
- [Update Token script](https://github.com/JanssenProject/jans-setup/blob/openbank/static/extension/update_token/UpdateToken.py) - included in the default Janssen OpenBanking distribution
- Setting configuration Parameters

## Adding the custom script

1. To add or update custom scripts, you can use either jans-cli or curl. jans-cli in interactive mode, option 13 enables you manage custom scripts. For more info, see the [docs](https://github.com/JanssenProject/home/wiki/Custom-Scripts-using-jans-cli).
1. jans-cli in command line argument mode is more conducive to scripting and automation. To display the available operations for custom scripts, use config-cli.py --info CustomScripts. See the [docs](../jans-cli.md) for more info.
1. To use `curl` see these [docs](../curl.md)

!!! Note
    You can normally find `jans-cli.py` in the `/opt/jans/jans-cli/` folder. 
 
## Steps to add / edit / delete configuration parameters:
1. Place a [JSON file] containing configuration parameters and the [custom script](https://github.com/JanssenProject/jans-setup/blob/openbank/static/extension/update_token/updatetoken.json) in a folder. 

1. From this folder, run the following command: 

```
python3 jans-cli-linux-amd64.pyz --operation-id post-config-scripts --data /updatetoken.json \
-cert-file yourcertfile.pem -key-file yourkey.key
```

## Methods

1. UpdateTokenType class and default methods

```
class UpdateToken(UpdateTokenType):

    def __init__(self, currentTimeMillis):
        self.currentTimeMillis = currentTimeMillis

    def init(self, customScript, configurationAttributes):
        return True

    def destroy(self, configurationAttributes):
        return True

    def getApiVersion(self):
        return 11
```

2. modifyIdToken () : the crucial business logic. This script overrides the header and claims. 

```
    # Returns boolean, true - indicates that script applied changes
    # Note :
    # jsonWebResponse - is JwtHeader, you can use any method to manipulate JWT
    # context is reference of io.jans.oxauth.service.external.context.ExternalUpdateTokenContext (in https://github.com/GluuFederation/oxauth project, )
    def modifyIdToken(self, jsonWebResponse, context):
              
         #read from session 
	sessionIdService = CdiUtil.bean(SessionIdService)
	sessionId = sessionIdService.getSessionByDn(context.getGrant().getSessionDn()) # fetch from persistence
        openbanking_intent_id = sessionId.getSessionAttributes().get("openbanking_intent_id")
	acr = sessionId.getSessionAttributes().get("acr_ob")

            # header claims
	jsonWebResponse.getHeader().setClaim("custom_header_name", "custom_header_value")
			
	#custom claims
	jsonWebResponse.getClaims().setClaim("openbanking_intent_id", openbanking_intent_id)
			
	#regular claims        
	jsonWebResponse.getClaims().setClaim("sub", openbanking_intent_id)

	return True
	
```
