Security Example
================
This demo exploits 2 features of the new Enterprise Security Module

Secure Property Placeholder
=========================== 
just like the normal property placeholder which resolves ${property.name} occurances against the
named properties file on the classpath except that any of the said properties could have been encrypted by Studio's secure property 
editor. The private key used to encrypt the property must be the same as that supplied to mule upon server startup. So, in this example, 
given the key 1234567890123456 we say key="${properties.key}" and thus we must supply -Dproperties.key=1234567890123456

Secure Token Service
====================
we allow Mule to play the part of both the role Service Provider and Authorization Provider, in that it can both issue tokens
and verify incoming tokens. We saet the accessTokenEndpointPath to "access-token", the provider-authorized-grant-type to PASSWORD and the 
supportedGrantTypes to "RESOURCE_OWNER_PASSWORD_CREDENTIALS" thus obliging the resource owner and client to provide the username 
and password, grantType=password and the required scopes when requesting a token, like so:

curl "http://localhost:9999/access-token?grant_type=passwordarbey&password=hello123&scope=READ%20WRITE"

This will give a response like:

{"scope":"READ WRITE","expires_in":86400,"token_type":"bearer","access_token":"l8bFMEC9PA7NcpmHeTYS43Wl96_Y6LuIOhGci2zMJf0Qso9llgRLkgQjarMzUhvQz8vGVHmazrZ2C-Gjo20khg"}

Having the token in hand, the client can then request access to the service like so:

curl -vv http://localhost:9999/api/demos?access_token=l8bFMEC9PA7NcpmHeTYS43Wl96_Y6LuIOhGci2zMJf0Qso9llgRLkgQjarMzUhvQz8vGVHmazrZ2C-Gjo20khg

The token is checked by the validate action of the oauth2-provider.
Providing the correct token yields a 200 OK, while an invalid token yields 403 FORBIDDEN

