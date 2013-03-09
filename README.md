Security Example
================
This demo exploits 2 features of the new Enterprise Security Module

Secure Property Placeholder
=========================== 
Just like the normal property placeholder which resolves ${property.name} occurrences against the
named properties file on the classpath with the added benefit that any of the said properties could have been encrypted by Studio's secure property 
editor. The private key used to encrypt the property must be the same as that supplied to mule upon server startup. So, in this example, 
given the key 1234567890123456 we refer to the key as ${properties.key} and pass it at server startup as a JVM property -Dproperties.key=1234567890123456

```xml
	<secure-property-placeholder:config name="Secure_Property_Placeholder" key="${properties.key}" location="config.${env}.properties" doc:name="Secure Property Placeholder" />
```

Secure Token Service
====================
OAuth2 specifies 4 roles in its dance:
1. Service Provider
2. Authorization Provider
3. Client
4. Resource Owner
The idea in this example is to allow Mule to play the part of both the role Service Provider and Authorization Provider, in that it can both issue tokens
and verify incoming tokens. We set the accessTokenEndpointPath to "access-token", the provider-authorized-grant-type to PASSWORD and the 
supportedGrantTypes to "RESOURCE_OWNER_PASSWORD_CREDENTIALS" thus obliging the resource owner and client to provide the username 
and password, grantType=password and the required scopes when requesting a token, like so:

	curl "http://localhost:9999/access-token?grant_type=passwordarbey&password=hello123&scope=READ%20WRITE"

which will give a response like:

```json
	{"scope":"READ WRITE","expires_in":86400,"token_type":"bearer","access_token":"l8bFMEC9PA7NcpmHeTYS43Wl96_Y6LuIOhGci2zMJf0Qso9llgRLkgQjarMzUhvQz8vGVHmazrZ2C-Gjo20khg"}
```

The configuration of the Authorization Provider (note that the provider scopes are specific to the application):
```xml
	<oauth2-provider:config name="oauth2-provider"
		providerName="Pre-sales Demos" resourceOwnerSecurityProvider-ref="demos-security-provider"
		scopes="READ WRITE" connector-ref="http-connector"
		supportedGrantTypes="RESOURCE_OWNER_PASSWORD_CREDENTIALS"
		accessTokenEndpointPath="access-token" doc:name="OAuth provider module">
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

	curl -vv http://localhost:9999/api/demos?access_token=l8bFMEC9PA7NcpmHeTYS43Wl96_Y6LuIOhGci2zMJf0Qso9llgRLkgQjarMzUhvQz8vGVHmazrZ2C-Gjo20khg

The token is checked by the validate action of the oauth2-provider.

```xml
        <http:inbound-endpoint exchange-pattern="request-response" host="localhost" port="9999" path="api" connector-ref="http-connector" doc:name="/api">
            <not-filter> 
                <wildcard-filter pattern="/favicon.ico"/> 
            </not-filter>
            <object-to-string-transformer/>
            <oauth2-provider:validate/>
        </http:inbound-endpoint>
```

Providing the correct token yields a 200 OK, while an invalid token yields 403 FORBIDDEN


