Security Example
================
This demo exploits 2 features of the new Enterprise Security Module

Secure Property Placeholder
=========================== 
Just like the normal property placeholder which resolves ${property.name} occurrences against the
named properties file on the classpath with the added benefit that any of the said properties could have been encrypted by Studio's secure property 
editor. 
```xml
<secure-property-placeholder:config name="Secure_Property_Placeholder" key="${properties.key}" 
	location="config.${env}.properties" doc:name="Secure Property Placeholder" />
```
The private key used to encrypt the property must be the same as that supplied to mule upon server startup. So, in this example, 
given the key 1234567890123456 we refer to the key as ${properties.key} and pass it at server startup as a JVM argument
```bash
$MULE_HOME/bin/mule -Dproperties.key=1234567890123456
```

Secure Token Service
====================
OAuth2 specifies 4 roles in its dance:
	
	1. Resource Server
	2. Authorization Server
	3. Client
	4. Resource Owner
	
The idea in this example is to allow Mule to play the part of two roles: Resource Server and Authorization Provider, in that it can both issue tokens
and verify incoming tokens. As we interact with it, we of course are the Resource Owner, using the Client, curl to make the requests on our behalf. 

In our requests we provide the username and password, grantType=password and the required scopes when requesting a token, like so:

```bash
curl http://localhost:9999/newToken?grant_type=password&client_id=demos-client&username=nialdarbey&password=hello123&scope=READ%20WRITE
```
which will give a response like:

```json
{"scope":"READ WRITE","expires_in":86400,"token_type":"bearer","access_token":"l8bFMEC9PA7NcpmHeTYS43Wl96_Y6LuIOhGci2zMJf0Qso9llgRLkgQjarMzUhvQz8vGVHmazrZ2C-Gjo20khg"}
```

The configuration of the Authorization Provider follows. Note that the provider scopes are specific to the application and we set the accessTokenEndpointPath to "newToken" as invoked above, the provider-authorized-grant-type to PASSWORD and the 
supportedGrantTypes to "RESOURCE_OWNER_PASSWORD_CREDENTIALS":

```xml
<oauth2-provider:config name="oauth2-provider"
	providerName="Pre-sales Demos" resourceOwnerSecurityProvider-ref="demos-security-provider"
	scopes="READ WRITE" connector-ref="http-connector"
	supportedGrantTypes="RESOURCE_OWNER_PASSWORD_CREDENTIALS"
	accessTokenEndpointPath="token" doc:name="OAuth provider module">
	<oauth2-provider:clients>
		<oauth2-provider:client clientId="demos-client"
			type="PUBLIC" clientName="Demos Client" description="Demos Client desc">
			<oauth2-provider:authorized-grant-types>
				<oauth2-provider:authorized-grant-type>PASSWORD</oauth2-provider:authorized-grant-type>
			</oauth2-provider:authorized-grant-types>
			<oauth2-provider:scopes>
				<oauth2-provider:scope>READ</oauth2-provider:scope>
				<oauth2-provider:scope>WRITE</oauth2-provider:scope>
			</oauth2-provider:scopes>
		</oauth2-provider:client>
	</oauth2-provider:clients>
</oauth2-provider:config>
```

Having the token in hand, the client can then request access to the service like so:

```bash
curl -vv http://localhost:9999/api/demos?access_token=l8bFMEC9PA7NcpmHeTYS43Wl96_Y6LuIOhGci2zMJf0Qso9llgRLkgQjarMzUhvQz8vGVHmazrZ2C-Gjo20khg -H 'signature:xGmhL3iEP70UQMUQVlwI0Q=='
```

There are security check-points in this application:
	
	1. The incoming token is validated. If it has expired, is not provided or not recognised, then a 403 FORBIDDEN is sent back to the client
	2. If the token is validated, then the signature of the value '/api/demos' using the key '1@s9bl&gt;1LOJ94z4' is compared with the incoming header 'signature'. If the two match then the signature is verirified

```xml
	<flow name="server" doc:name="server">
		<http:inbound-endpoint exchange-pattern="request-response"
			host="localhost" port="9999" path="api" connector-ref="http-connector"
			doc:name=":9999/api">
			<object-to-string-transformer />
		</http:inbound-endpoint>
		<oauth2-provider:validate  config-ref="oauth2-provider" doc:name="validate" />
        <set-property propertyName="http.status" value="403" doc:name="http.status = 403"/>
		<signature:verify-signature config-ref="Signature"
			using="JCE_SIGNER" input-ref="#[message.inboundProperties['http.request.path']]"
			expectedSignature="#[message.inboundProperties.signature]" doc:name="verify">
			<signature:jce-signer algorithm="HmacMD5"
				key="1@s9bl&gt;1LOJ94z4" />
		</signature:verify-signature>
        <set-property propertyName="http.status" value="200" doc:name="http.status = 200"/>
		<set-payload value="Welcome" doc:name="Welcome" />
	</flow>

```

	
Both the OAuth2 validator and the Signature verifier are filters. The message will only pass through upon validation and verification respectively.

Contact
=======
* nial.darbey@mulesoft.com
