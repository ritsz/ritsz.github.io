---
title: Authentication and Authorization
created: '2021-03-09T14:16:13.169Z'
modified: '2021-05-16T08:59:25.992Z'
---

# Authentication and Authorization

### Reference

* [Introduction to OAuth](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2)

### Authorization

#### OAuth v2.0
* Earlier each service (eg budget-planner) needed to have username/password for each resource you want to access (eg bank transactions, mutual funds). This sharing of passwords is not secure.
* OAuth 2 is an authorization framework that enables applications to obtain limited access to user accounts on an HTTP service.
* OAuth v2.0 provides a standard to securly allow one service to access data from another service.
* OAuth allows authorization between services.
* A User gives one service limited access to a resource served by another service. This is called delecated access.
* Each service trusts the user, but doesn't trust one another.
* The App (budget-planner) will be given a key (token) that gives them access to very specific resources on another service (eg. transaction history).
* The **User** authorizes one **application** to access specific resources from an **API** without giving the User's password to the application. (eg when facebook wants to be authorized to access your contact list.)

#### Terminologies
* **Resource Owner** The user that grants access to the client to access it's resources. (eg r.ranjan789)
* **Client** The application that requests for the Data. (eg Spotify)
* **Authorization Server** Application that knows the resource owner where the resource owner already has an account. (eg. auth server of google.com)
* **Resource Server** API/Service that the client wants to use on behalf of the resouce owner (eg Contact list API or Spotify Playlist etc.)
* **Redirect/Callback URI** The URL that the Authorization server redirects the client to after successful authorization. 
* **Response type** The type of response a client expects to receive. (eg Code where the client expects an authorization code.)
* **Scope** The granular permissions that the client wants/receives from the authorization server.
* **Consent** The resources owners consents a list of scope for a specific client.
* **Client ID** Much before the OAuth exchange, client registers with the Auth Server and gets a client ID and a client secret that it can use to identify itself to the Auth Server (eg Spotify registers with google auth server.)

#### Client registration
* Before using OAuth with your application, you must register your application with the service. This is done through a registration form in the *developer* or *API* portion of the service’s website, where you will provide the following information (and probably details about your application):
    * Application Name
    * Application Website
    * Redirect URI or Callback URL.
* Once your application is registered, the service will issue “client credentials” in the form of a client identifier and a client secret. 
* The *Client ID* is a publicly exposed string that is used by the service API to identify the application, and is also used to build authorization URLs that are presented to users. 
* The *Client Secret* is used to authenticate the identity of the application to the service API when the application requests to access a user’s account, and must be kept private between the application and the API.

#### Authorization Workflow
* The application requests authorization to access service resources from the user.
* If the user authorized the request, the application receives an authorization grant.
* The application requests an access token from the authorization server (API) by presenting authentication of its own identity, and the authorization grant.
* If the application identity is authenticated and the authorization grant is valid, the authorization server (API) issues an **access token** to the application. Authorization is complete.
* The application requests the resource from the resource server (API) and presents the access token for authentication.
* If the access token is valid, the resource server (API) serves the resource to the application.
* The authorization code grant type is the most commonly used because it is optimized for server-side applications, where source code is not publicly exposed, and Client Secret confidentiality can be maintained. 
* This is a redirection-based flow, which means that the application must be capable of interacting with the user-agent (i.e. the user’s web browser) and receiving API authorization codes that are routed through the user-agent.
* The client asks for authorization using an API like this:
```
https://cloud.digitalocean.com/v1/oauth/authorize?response_type=code&client_id=CLIENT_ID&redirect_uri=CALLBACK_URL&scope=read
```
* `redirect_uri=CALLBACK_URL`: where the service redirects the user-agent after an authorization code is granted.
* `scope=read`: specifies the level of access that the application is requesting
* `response_type=code`: specifies that your application is requesting an authorization code grant.
* The user will be prompted by the service to authorize or deny the application access to their account.
* If the user clicks “Authorize Application”, the service redirects the user-agent to the application redirect URI, which was specified during the client registration, along with an authorization code.
```
https://dropletbook.com/callback?code=AUTHORIZATION_CODE
```
* Client sends the client ID + Client Secret + authorization code to Auth Server's **Token endpoint**, and receives an Auth Token. The client issues a POST request to the Auth Server's token grant API endpoint with these parameters and receives a token in response.
```
https://cloud.digitalocean.com/v1/oauth/token?client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=CALLBACK_URL
```
* Sending the POST request and receiving the access token happens in the back channel (ie, non browser code, as opposed to getting the Authorization Code which happens on the front channel, ie browser). The back channel is more secure than the front channel (browser) and Auth token is a sensetive data, so the communications tha require auth token happen on the back channel. We do not trust the browser with things like the token, the client key etc.
* If the authorization is valid, the API will send a response containing the access token (and optionally, a refresh token) to the application. If a refresh token was issued, it may be used to request new access tokens if the original token has expired.
```json
{
  "access_token":"ACCESS_TOKEN",
  "token_type":"bearer",
  "expires_in":2592000,
  "refresh_token":"REFRESH_TOKEN",
  "scope":"read",
  "uid":100101,
  "info":{
    "name":"Mark E. Mark",
    "email":"mark@thefunkybunch.com"
  }
}
```
* The Auth Token needs to have the user allowed permissions, and also needs to be trustable by the issuer (resource server).
* OAuth2.0 token format is not specified in the standard. A wide variety of token can be used. eg. SAML token, JWT 


### Authentication
#### OpenID Connect
* OpenID allows a client to establish a login session (*authentication*).
* OAuth 2.0 with OpenID works as *Authentication as a service*.
* Also has information about the client that has logged in (*Identity*). Also called an **identity provider**.
* One login can be used across multiple applications. (*SSO*) 
* Sits on top of the OAuth 2.0 protocol.
* It enables client (Spotify) to verify the identity of the Resource Owner (`riteranj@foobar.com`) based on the authentication performed by the Authorization Server (google.com auth server).
* The resource owner has already authenticated itself with the **Authorization Server**.
* One of the resources a resource owner owns is their own identity.
* When the client tries to authorize itself, it sets the scope parameter as `OPENID`. This scope means the authorization server has to authenticate the resource owner as well as authorize the client.
* Apart from the access token, client will also receive basic identity information about the **Resouce Owner**. This is called an *ID Token*.
* The ID Token contains claims about the resource owner:
  * Subject (username)
  * Issuing authority (identity provider)
  * Issue date 
  * Expiration date
* ID Token are encoded as a JSON web token(**JWT**). 
* The client is also called the Relying Party (**RP**).
* The resource server has the userinfo endpoint.
* The **UserInfo endpoint** is an OAuth 2.0 protected resource, which means that the credential required to access the endpoint is the access token.
* After the client has received the Auth Token and ID Token, if the client application needs more information about the user, it connects to the userinfo endpoint.
* To obtain the claims for a user, a client makes a request to the UserInfo endpoint by using an access token as the credential. The access token must be one that was obtained through OpenID Connect authentication. The claims for the user who is represented by the access token are returned as a JSON object that contains a collection of name-value pairs for the claims.
```
https://server.example.com:443/oidc/endpoint/<provider_name>/userinfo
```


### SAML: Security Assertion Markup Language
* SAML is a standard that can be used for both Authentication and Authorization.
* Has 3 components: User, Service Provider and Identity Provider.
* Identity Provider has the job of authenticating the user to all the service providers.
* A trust relation has to be established between SP and IDP. The Service Provider should trust the assertions provided by the IDP.
* The client accesses the service provider resource which asks the Identity Provider to authenticate the client.
  * The IDP can authenticate the client using any method: User/Password, ActiveDirectory etc.
  * The IDP creates an "assertion" for the client, which is an XML that has the identity of the client, eg X509 Certificate etc.
  * The IDP signs the assertion so that the service provider can verify the issuer of the assertion. Now the assertion has the client's identity information and the identity providers signature.
* This assertion can be used by multiple resource providers to identify the client, thus implementing Single Sign On (SSO)
* IDP-INIT: The user connects to the IDP, authenticates itself and gets a SAML assertion. The user then passes the SAML assertion to the SP, which maps the SAML assertion to a user in it's database and authenticates the user. (for example Google IDP sends an assertion for r.ranjan789@gmail.com and spotify maps that user to r.ranjan789@gmail.com in it's user database).
* SP-INIT: The user connects to the SP, which redirects the user to the IDP where the authentication happens.

#### VCenter SAML tokens
* Get STS URL
```python
lsClient = LookupServiceClient()
sts = lsClient.GetServiceEndpoints(STS_TYPE_ID, ...)
stsUrl, stscert = (sts[0].url, sts[0].sslTrust[0])
```
* Get **SAML HoK Token** - SAML Holder of Key token is an assertion of identity based on the crt and key file provided. The user provides it's certificate and key to the Identity provider
```python
auth = sso.SsoAuthenticator(sts_url=stsUrl)
samlToken = auth.get_hok_saml_assertion(VUM_CRT, VUM_KEY, token_duration=token_duration)
```
* Login to the vcsa using SAML HoK token.
```python
vcStub = SoapStubAdapter(url=vcUrl, version='vim.version.version12', samlToken=samlToken)
si = vim.ServiceInstance('ServiceInstance', vcStub)
content = si.RetrieveContent()
sessionMgr = content.GetSessionManager()
sessionMgr.LoginByToken()
```
* The SAML Token has been signed by the identity provider (**vmware ssoserversign**):
```sh  
  Signature Algorithm: sha256WithRSAEncryption
      Issuer: CN=CA, DC=vsphere, DC=local, C=US, ST=California, O=wdc-10-191-178-31.nimbus.eng.vmware.com, OU=VMware Engineering
      Validity
          Not Before: Apr 28 11:52:21 2021 GMT
          Not After : Apr 23 12:01:37 2031 GMT
      Subject: CN=ssoserverSign
      Subject Public Key Info:
```

### WCP SVC Certificates and Tokens
* WCP Certificate can be retrieved from vecs-cli:
```sh
/usr/lib/vmware-vmafd/bin/vecs-cli entry getcert --store wcp --alias wcp | openssl x509 -inform pem -noout -text
       Issuer: CN=CA, DC=vsphere, DC=local, C=US, ST=California, O=wdc-10-191-178-31.nimbus.eng.vmware.com, OU=VMware Engineering
       Validity
           Not Before: Apr 28 12:00:47 2021 GMT
           Not After : Apr 28 12:00:47 2023 GMT
       Subject: CN=wcp, C=US, ST=California, L=Palo Alto, O=VMware, OU=mID-dbd8d139-d349-4fee-86eb-97bf180a510e
            Authority Information Access:
              CA Issuers - URI:https://wdc-10-191-178-31.nimbus.eng.vmware.com/afd/vecs/ca
```
* Note that the issuer of the certificates is the VECS on the VCSA.
* JWT Token for the SVC APIServer for sso user Administrator@vsphere.local
```json
	{
	  "sub": "Administrator@vsphere.local",
	  "aud": "vmware-tes:vc:vns:k8s",
	  "domain": "vsphere.local",
	  "iss": "https://wdc-10-191-178-31.nimbus.eng.vmware.com/openidconnect/vsphere.local",
	  "group_names": [
	    "LicenseService.Administrators@vsphere.local",
	    "SystemConfiguration.ReadOnly@vsphere.local",
	    "SystemConfiguration.SupportUsers@vsphere.local",
	    "Administrators@vsphere.local",
	    "Everyone@vsphere.local",
	    "CAAdmins@vsphere.local",
	    "SystemConfiguration.Administrators@vsphere.local",
	    "SystemConfiguration.BashShellAdministrators@vsphere.local",
	    "Users@vsphere.local"
	  ],
	  "exp": 1619931741,
	  "iat": 1619895741,
	  "jti": "b32c4e9b-8f81-41ae-af09-0d185f454944",
	  "username": "Administrator"
	}
```
* `sub`: subject of the token.
* `aud`: audience of the token.
* `iss`: issuer of the token. `https://wdc-10-191-178-31.nimbus.eng.vmware.com/openidconnect/vsphere.local`
* The kube-apiserver is started with these OIDC config:
```sh
	root@4231cdf83d51a4be5116894e34913994 [ ~ ]# grep  oidc /etc/kubernetes/manifests/kube-apiserver.yaml
	    - --oidc-ca-file=/etc/vmware/wcp/tls/vmca.pem
	    - --oidc-client-id=vmware-tes:vc:vns:k8s
	    - --oidc-groups-claim=group_names
	    - '--oidc-groups-prefix=sso:'
	    - --oidc-issuer-url=https://wdc-10-191-178-31.nimbus.eng.vmware.com/openidconnect/vsphere.local
	    - '--oidc-username-prefix=sso:'
```
* Note that oidc client id is the JWT audience id, and oidc issuer url matches the issuer of the token.
* `oidc-issue-url` point to the `.well-known/openid-configuration` : `https://wdc-10-191-178-31.nimbus.eng.vmware.com/openidconnect/vsphere.local/.well-known/openid-configuration` is called the **OIDC discovery endpoint**.
* APIServer needs the public key of the issuer to validate the signature on tokens. The oidc-issuer-url is the URL that APIServer uses to get the public keys of the issuer. `https://wdc-10-191-178-31.nimbus.eng.vmware.com/openidconnect/jwks/vsphere.local`
* **JWKS** - Json web key store is the uri where the public keys of the issuer are stored.

#### How to exchange SAML Token to JWT
```sh

export SAMLT=$(python get_saml_token.py) #BASE-64 ENCODED SAML TOKEN OF THE USER

dcli com vmware vcenter tokenservice tokenexchange exchange          \
  --grant-type "urn:ietf:params:oauth:grant-type:token-exchange"     \
  --subject-token-type "urn:ietf:params:oauth:token-type:saml2"      \
  --requested-token-type "urn:ietf:params:oauth:token-type:id_token" \
  --audience "vmware-tes:vc:vns:k8s"                                 \
  --subject-token $SAMLT
access_token: eyJraWQiOiJDRkZBNUZDREUwNzI2MDNEMDBBRjYxNzU2MUQ1QUUxNUQxMzU0MEE1Ii
wiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiJBZG1pbmlzdHJhdG9yQHZzcGhlcmUubG9jYWwiLCJhdWQiOiJ
2bXdhcmUtdGVzOnZjOnZuczprOHMiLCJkb21haW4iOiJ2c3BoZXJlLmxvY2FsIiwiaXNzIjoiaHR0cHM
6XC9cL3dkYy0xMC0xOTEtMTc4LTMxLm5pbWJ1cy5lbmcudm13YXJlLmNvbVwvb3BlbmlkY29ubmVjdFw
vdnNwaGVyZS5sb2NhbCIsImdyb3VwX25hbWVzIjpbIkxpY2Vuc2VTZXJ2aWNlLkFkbWluaXN0cmF0b3J
zQHZzcGhlcmUubG9jYWwiLCJTeXN0ZW1Db25maWd1cmF0aW9uLlJlYWRPbmx5QHZzcGhlcmUubG9jYWw
iLCJTeXN0ZW1Db25maWd1cmF0aW9uLlN1cHBvcnRVc2Vyc0B2c3BoZXJlLmxvY2FsIiwiQWRtaW5pc3R
yYXRvcnNAdnNwaGVyZS5sb2NhbCIsIkV2ZXJ5b25lQHZzcGhlcmUubG9jYWwiLCJDQUFkbWluc0B2c3B
oZXJlLmxvY2FsIiwiU3lzdGVtQ29uZmlndXJhdGlvbi5BZG1pbmlzdHJhdG9yc0B2c3BoZXJlLmxvY2F
sIiwiU3lzdGVtQ29uZmlndXJhdGlvbi5CYXNoU2hlbGxBZG1pbmlzdHJhdG9yc0B2c3BoZXJlLmxvY2F
sIiwiVXNlcnNAdnNwaGVyZS5sb2NhbCJdLCJleHAiOjE2MjAwMDIyMDIsImlhdCI6MTYxOTk2NjIwMiw
ianRpIjoiMzFiNzVlMTMtYThlNS00OGEyLThiNjEtNDBkZjk3M2UyZWM3IiwidXNlcm5hbWUiOiJBZG1
pbmlzdHJhdG9yIn0.OaT35A5E2oHeiD8rhoEZkc6obwdKdwoAlZ3dLvgVHlmU6wNEeiGA8Ehp5DhnqrP
JIgJpz8EeNKFL6ybmDzVaMWjSHFo8xXA7cMJvcEabtQr26cKqv85gSYyoSvIB7q1nVuu86CLND9k5p7s
1KrwdWcOPxYeYnTdKQOnIf75faRctjLiISVI7IRIcmFRKba2U3vVKilwuCLvSl6Prj4ueph6yTWDVTyx
FQBXWPi2bwBRjHFmjJwaw-Q8WVChEHmF6rFuzkRmc7cOGHHFLnG22qXTdYOjGquNAEikcth0c_UMWtQV
luUyedWML5vV2GeXj7QB36tbyvH9shTbbgfYFtA
refresh_token:
issued_token_type: urn:ietf:params:oauth:token-type:id_token
scope: openid
token_type: N_A
expires_in: 35999
```
* Decoding the JWT gave
```json
	{
	  "sub": "Administrator@vsphere.local",
	  "aud": "vmware-tes:vc:vns:k8s",
	  "domain": "vsphere.local",
	  "iss": "https://wdc-10-191-178-31.nimbus.eng.vmware.com/openidconnect/vsphere.local",
	  "group_names": [
	    "LicenseService.Administrators@vsphere.local",
	    "SystemConfiguration.ReadOnly@vsphere.local",
	    "SystemConfiguration.SupportUsers@vsphere.local",
	    "Administrators@vsphere.local",
	    "Everyone@vsphere.local",
	    "CAAdmins@vsphere.local",
	    "SystemConfiguration.Administrators@vsphere.local",
	    "SystemConfiguration.BashShellAdministrators@vsphere.local",
	    "Users@vsphere.local"
	  ],
	  "exp": 1620002202,
	  "iat": 1619966202,
	  "jti": "31b75e13-a8e5-48a2-8b61-40df973e2ec7",
	  "username": "Administrator"
	}
```
* This is the same JWT that is present in the `.kube/config` when the SSO user logs in to the SVC.
* JWT is also used as a substitute for sessionIDs. Earlier servers used to maintain session informartion in memory, and share the sessionID to client, which it could set in cookies in subsequent requests. The server knows the state of the session using the sessionID. This has issues with microservices where client might need multiple sessionIDs for each service. Instead a JWT can be used and added as a bearer token. 











