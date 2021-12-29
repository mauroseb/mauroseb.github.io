---
layout: posts
title: "Quay Single Sign On"
date: 2021-07-24
categories: [blog]
tags: [ quay, sso, keycloack, idm ]
author: mauroseb
excerpt_separator: "<!-- more -->"
---
### Intro

In this post I will cover the configuration overview of Red Hat Single Sign On (SSO) with an Red Hat Identity Management (IdM) backend and then integrate the authentication of Red Hat Quay enterprise registry with SSO.


### Environment

 1. **Red Hat IdM 4.6.8.** IdM is setup with default configuration. Some groups and users were created for this test: like the **mauro** or **jdoe** which to the **devops** group which will be the test subject. <br/>

  <img src="/images/quay-sso-idm_users.png" alt="IdM users PNG" style="width:30%;"/>

 2. **Red Hat SSO 7.3.8.** SSO has been installed and configured with a realm called LAB <br/>

  <img src="/images/quay-sso-sso_realm.png" alt="SSO realm PNG" style="width:60%;"/>

   - In addition the following configuration has been set to source federated users from IdM. The account **uid=rhsso,cn=sysaccounts,cn=etc,dc=lab,dc=local** is used as BindDN. <br/>
  <img src="/images/quay-sso-sso_federation.png" alt="SSO federation PNG" style="width:60%;"/>

   - The provider settings are displayed in the next image <br/>
  <img src="/images/quay-sso-sso_federation_provider.png" alt="SSO federation provider PNG" style="width:60%;"/>

   - I have created an LDAP mapper configuration to map groups in iDM to groups in SSO. <br/>
  <img src="/images/quay-sso-sso_ldapmapper.png" alt="SSO LDAP mapper PNG" style="width:60%;"/>


 3. **Quay 3.3.5.** Is already installed using the _Basic_ method[^1] (as this is a LAB) which is just using plain containers. There are other alternatives like using an HA deployment, or on OpenShift 4 using the Quay Operator directly if you have OpenShift 4 running. Then I configured it to use the certificates signed by IdM CA to serve user traffic. Quay setup steps may slightly differ from one install method to other but in the end in all cases it will wind up to setup the Quay config.yaml file.


### Procedure

#### Red Hat Single Sign On Setup

To begin with, I will setup Single Sign On for Quay using ice-sso-01.lab.local server as the authentication source. This is achieved by using the OpenID Connect protocol.
OpenIDC 1.0 is an authentication protocol that is built upon OAuth 2.0 protocol (ment for authorization) [^2].

 1. Log in to SSO with LAB realm admin privileges

 2. Create a new OpenID Connect client for Quay. Make sure we are in the right realm LAB, and then in the Clients menu and click Create. <br/>

<img src="/images/quay-sso-sso_quayclient-1.png" alt="SSO quay client 1 PNG" style="width:75%;"/>

 3. Once in the Add Client menu, add quay-enterprise as the Client ID and openid-connect as Client Protocol. Then click Save. <br/>

<img src="/images/quay-sso-sso_quayclient-2.png" alt="SSO quay client 2 PNG" style="width:75%;"/>

 4. That will open the Settings tab. Fill in the fields as follows: <br/>
   a. It is mandatory to set Access Type as confidential. Confidential access type is for server-side clients that need to perform a browser login and require a client secret when they turn an access code into an access token. <br/>
   b. It is also mandatory to set _Valid Redirect URL_ as the Quay’s SSO callback webhook which in my case will be: **https://ice-quay-01.lab.local/oauth2/redhatsso/callback**. Then click Save. <br/>

<img src="/images/quay-sso-sso_quayclient-3.png" alt="SSO quay client 3 PNG" style="width:75%;"/>


#### Red Hat Quay Setup

Now we need to setup Quay to use Red Hat SSO as external authentication.

 1. Start Quay in config mode and log in. For that I am going to start the Quay container in config mode and access the portal at **https://ice-quay-01.lab.local:8443/**. The user as usual is _quayuser_ and the password the one passed to the quay container.

 2. Click on _Modify existing configuration_ <br/>
  <img src="/images/quay-sso-quay_oidc-1.png" alt="SSO quay OIDC 1 PNG" style="width:50%;"/>

 3. Upload the current config tarball to open the _Config_ menu

 4. In _External Configuration_ section hit _Add OIDC Provider_ button. <br/>
  <img src="/images/quay-sso-quay_oidc-2.png" alt="SSO quay OIDC 2 PNG" style="width:50%;"/>

 5. Enter **redhatsso** as the provider name and hit _OK_. <br/>
  <img src="/images/quay-sso-quay_oidc-3.png" alt="SSO quay OIDC 3 PNG" style="width:50%;"/>

 6. Fill in the configuration for the SSO client Quay we created in the previous section. <br/>
   a. _OIDC Server_ will be the SSO FQDN + '/auth/realms/<REALM NAME>' (where <REALM NAME> is LAB). <br/>
   b. _Client ID_ should be the name we created in SSO: **quay-enterprise** <br/>
   c. The _Client Secret_ should be copied from SSO. Within the **quay-enterprise** client menu, _Credentials_ tab, check the _Secret_ field. <br/>
      <img src="/images/quay-sso-quay_oidc-4.png" alt="SSO quay OIDC 4 PNG" style="width:50%;"/> <br/>
   d. Fill in _Service Name_. <br/>
   e. For _Login Scopes_ type **openid** and then hit _Add_. <br/>
   f. Check the _Callback URLs_ are as expected by SSO. <br/>
   g. In the section _Access Settings_ toggle _Enable Open User Creation_ to allow federated users creation. <br/>
   h. Leave _Internal Authentication_ with the default _Local Database_, which will ask the users to create a local password for pushing/pulling content after the first login. <br/>
   i. Finally hit _Save Configuration Changes_. <br/>
      <img src="/images/quay-sso-quay_oidc-5.png" alt="SSO quay OIDC 5 PNG" style="width:50%;"/>

  **NOTE:** At this point I received a TLS error due to Red Hat SSO using self-signed certs. <br/>
      <img src="/images/quay-sso-quay_oidc-6.png" alt="SSO quay OIDC 6 PNG" style="width:50%;"/>

  See Appendix 1 to see how to setup Red Hat SSO with custom certificates issued and signed by Red Hat IdM so Quay can trust them. After this fix we can observe that if we try to save Quay configuration again it now succeeds. <br/>
      <img src="/images/quay-sso-quay_oidc-7.png" alt="SSO quay OIDC 7 PNG" style="width:50%;"/>

{:start="7"}
 7. Download the new configuration as **quay-config.tar.gz**, stop the quay container in config mode, and upload the tarball to Quay’s config volume [2]. Finally start the container in normal mode.

{% highlight console %}
[root@ice-quay-01 config]# pwd
/var/lib/quay/config
[root@ice-quay-01 config]# tar xvf quay-config.tar.gz
ssl.cert
ssl.key
config.yaml
extra_ca_certs/
extra_ca_certs/ipa-ca.crt
[root@ice-quay-01 config]# podman run -d \
>   --restart=always \
>   --name quay \
>   --sysctl net.core.somaxconn=4096 \
>   --privileged=true \
>   --publish 80:8080 \
>   --publish 443:8443 \
>   -v /var/lib/quay/config:/conf/stack:Z \
>   -v /var/lib/quay/storage:/datastorage:Z \
>   quay.io/redhat/quay:v3.3.1
3c0ce6b6beb3c2ef92cc4e35cba2f2e6e0454fcea3c9e82b8b013b50bfb46926
{% endhighlight %}

 8. Now at the Quay login page observe the LAB SSO backend being displayed.<br/>
  <img src="/images/quay-sso-quay_oidc-8.png" alt="SSO quay OIDC 8 PNG" style="width:30%;"/>

 9. Once we click on it we are immediately redirected to our Red Hat SSO instance to login.<br/>
  <img src="/images/quay-sso-quay_oidc-9.png" alt="SSO quay OIDC 9 PNG" style="width:30%;"/>

 10. After authentication succeeds, the callback to Quay is correctly invoked and upon first login the new user creation needs to be confirmed within Quay.<br/>
  <img src="/images/quay-sso-quay_oidc-10.png" alt="SSO quay OIDC 10 PNG" style="width:50%;"/>

 11. When user is confirmed the user is finally in.<br/>
  <img src="/images/quay-sso-quay_oidc-11.png" alt="SSO quay OIDC 11 PNG" style="width:50%;"/>


### Appendix 1 - Setup custom certificates for Red Hat SSO

 1. Create a JKS keystore for **jboss** user. For IdM is very important to pick the right DN otherwise it will not accept the CSR.

{% highlight console %}
[root@ice-sso-01 ~]# su - jboss -s /bin/bash
[jboss@ice-sso-01 ~]$ keytool -genkey -alias ice-sso-01.lab.local -dname "CN=ice-sso-01.lab.local,O=LAB.LOCAL" -keyalg RSA -keystore ~jboss/ice-sso-01.jks -keysize 2048
Enter keystore password:  
Re-enter new password: 
Enter key password for <ice-sso-01.lab.local>
	(RETURN if same as keystore password):  

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore /opt/rh/rh-sso7/root/usr/share/keycloak/ice-sso-01.jks -destkeystore /opt/rh/rh-sso7/root/usr/share/keycloak/ice-sso-01.jks -deststoretype pkcs12".

[jboss@ice-sso-01 ~]$ keytool -certreq -alias ice-sso-01.lab.local -keystore ice-sso-01.jks > /tmp/ice-sso-01.csr
Enter keystore password:  redhat

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore ice-sso-01.jks -destkeystore ice-sso-01.jks -deststoretype pkcs12".

[jboss@ice-sso-01 ~]$ cat /tmp/ice-sso-01.csr
-----BEGIN NEW CERTIFICATE REQUEST-----
MIICqDCCAZACAQAwMzESMBAGA1UEChMJTEFCLkxPQ0FMMR0wGwYDVQQDExRpY2Ut
c3NvLTAxLmxhYi5sb2NhbDCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
ALZUj/P7msyL6cfO5RJhetleE1aKQkR+kUVUUKU4WJ5kelk53zut4TQNziik58jE
5G/hTJn1fM5OTwagT7Q0XWJ7/MYtMfA42oKEjsK0vI2T5Agln+12KbsuATfBjlAF
BNs6P65BHwzUT09KMrEKo1RTr7v458jaGu/gaPjyZZDION5YBa4L2JLqBnTWJJLe
phIOUCtL9rEkNfcZWKIjJZbJyw0j+eCnFmceslRmTz+X8otAER3SikTi80X5I4U9
9/vbjhZwpZ2NaYIL8YCYuI6bHDuiyOpHu46rYm62E8ihCVKaoS8CIGj4kZ0tls2m
plUcINYEFqwEBgA1vzJrfikCAwEAAaAwMC4GCSqGSIb3DQEJDjEhMB8wHQYDVR0O
BBYEFBypK+ogKXhbOXfqHGYQ3pMQ9axcMA0GCSqGSIb3DQEBCwUAA4IBAQBEB/sC
s2kVf1IfoaKvWAZmQ0v/CR+NtxiKIbQ7vtqGkfBZoCYrPMhGjmHx3CaXwATbEx5T
e5Lc2wQ4ftmbTvlMH/4sdJZwdAa5qE4AsmHxsTqT2wcV3cVIx66fuZy8g58unorr
fKrp/eCGz6rKAKz1SYnMvF1BC4NDZMmso6TgUCs1vmr1c61nrXs9Tlr51Q6hgiAz
EOgLD6VQvYrel1gXxuIJNxua9wzLUa7IAwDT+gdB/T1pt6pRI4FCickNKrJFraLZ
JN6tlK0lhop2+bJAxWRa9rTngO8FtLH4qTuflAg+nvOXSbEaz/uFWqtgXlLqy8JP
Sst7UkQz6vMwx31E
-----END NEW CERTIFICATE REQUEST-----
{% endhighlight %}

 2. In IdM create a host for ice-sso-01 server, then create a service for SSO, and finally issue a certificate for the principal **sso/ice-sso-01.lab.local@LAB.LOCAL** passing the CSR created in the previous step.

 3. Prior to downloading the signed certificate to configure SSO, we need to add the CA from IdM into our JKS keystore[^3].

{% highlight console %}
[jboss@ice-sso-01 ~]$ ls
bin     ice-sso-01.crt  JBossEULA.txt      LICENSE.txt  themes
docs    ice-sso-01.jks  jboss-modules.jar  modules      version.txt
domain  ipa-ca.crt      License.html       standalone   welcome-content
[jboss@ice-sso-01 ~]$ keytool -import -keystore ice-sso-01.jks -file ipa-ca.crt -alias root
Enter keystore password:  
Owner: CN=Certificate Authority, O=LAB.LOCAL
Issuer: CN=Certificate Authority, O=LAB.LOCAL
Serial number: 1
Valid from: Tue Oct 27 08:38:05 EDT 2020 until: Sat Oct 27 08:38:05 EDT 2040
Certificate fingerprints:
	 MD5:  B6:C9:9B:0A:0A:CF:02:F5:73:83:00:9A:D5:D4:9F:4A
	 SHA1: 8F:3A:AF:14:AC:0E:63:FE:6A:3A:DD:1F:20:8F:A8:F6:9C:EC:E5:D4
	 SHA256: D0:58:C7:C2:10:AE:25:EE:E4:87:6E:43:3E:6B:4B:3C:D8:67:A4:F8:C7:35:84:39:3F:A0:A6:A0:61:17:D4:A7
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

Extensions: 

#1: ObjectId: 1.3.6.1.5.5.7.1.1 Criticality=false
AuthorityInfoAccess [
  [
   accessMethod: ocsp
   accessLocation: URIName: http://ipa-ca.lab.local/ca/ocsp
]
]

#2: ObjectId: 2.5.29.35 Criticality=false
AuthorityKeyIdentifier [
KeyIdentifier [
0000: 70 2D 23 F1 B1 B8 F0 C5   7E 4A 74 71 0F A1 26 60  p-#......Jtq..&`
0010: 63 92 4F 1E                                        c.O.
]
]

#3: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:true
  PathLen:2147483647
]

#4: ObjectId: 2.5.29.15 Criticality=true
KeyUsage [
  DigitalSignature
  Non_repudiation
  Key_CertSign
  Crl_Sign
]

#5: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 70 2D 23 F1 B1 B8 F0 C5   7E 4A 74 71 0F A1 26 60  p-#......Jtq..&`
0010: 63 92 4F 1E                                        c.O.
]
]

Trust this certificate? [no]:  yes
Certificate was added to keystore

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore ice-sso-01.jks -destkeystore ice-sso-01.jks -deststoretype pkcs12".

[jboss@ice-sso-01 ~]$ keytool -import -keystore ice-sso-01.jks -file ice-sso-01.crt -alias ice-sso-01.lab.local
Enter keystore password:  
Certificate reply was installed in keystore

Warning:
The JKS keystore uses a proprietary format. It is recommended to migrate to PKCS12 which is an industry standard format using "keytool -importkeystore -srckeystore ice-sso-01.jks -destkeystore ice-sso-01.jks -deststoretype pkcs12".
{% endhighlight %}

 4. Now that the JKS key store is ready add it to the Wildfly configuration. I could specify the full path to it but I preferred to place it in the configuration directory.

{% highlight console %}
[jboss@ice-sso-01 ~]$ mv ice-sso-01.jks standalone/configuration/
[jboss@ice-sso-01 ~]$ ./bin/jboss-cli.sh 
You are disconnected at the moment. Type 'connect' to connect to the server or 'help' for the list of supported commands.
[disconnected /] connect
[standalone@localhost:9990 /] /core-service=management/security-realm=UndertowRealm:add()
{"outcome" => "success"}


[standalone@localhost:9990 /] /core-service=management/security-realm=UndertowRealm/server-identity=ssl:add(keystore-path=ice-sso-01.jks, keystore-relative-to=jboss.server.config.dir, keystore-password=redhat)
{
    "outcome" => "success",
    "response-headers" => {
        "operation-requires-reload" => true,
        "process-state" => "reload-required"
    }
}
[standalone@localhost:9990 /]  /subsystem=undertow/server=default-server/https-listener=https:write-attribute(name=security-realm, value=UndertowRealm)
{
    "outcome" => "success",
    "response-headers" => {
        "operation-requires-reload" => true,
        "process-state" => "reload-required"
    }
}
[standalone@localhost:9990 /] exit
[jboss@ice-sso-01 ~]$ grep -5 UndertowRealm standalone/configuration/
application.keystore          logging.properties            standalone.xml
application-roles.properties  mgmt-groups.properties        standalone_xml_history/
application-users.properties  mgmt-users.properties         
ice-sso-01.jks                standalone-ha.xml             
[jboss@ice-sso-01 ~]$ grep -5 UndertowRealm standalone/configuration/standalone.xml 
                </authentication>
                <authorization>
                    <properties path="application-roles.properties" relative-to="jboss.server.config.dir"/>
                </authorization>
            </security-realm>
            <security-realm name="UndertowRealm">
                <server-identities>
                    <ssl>
                        <keystore path="ice-sso-01.jks" relative-to="jboss.server.config.dir" keystore-password="redhat"/>
                    </ssl>
                </server-identities>
--
        </subsystem>
        <subsystem xmlns="urn:jboss:domain:undertow:7.0" default-server="default-server" default-virtual-host="default-host" default-servlet-container="default" default-security-domain="other">
            <buffer-cache name="default"/>
            <server name="default-server">
                <http-listener name="default" socket-binding="http" redirect-socket="https" enable-http2="true"/>
                <https-listener name="https" socket-binding="https" security-realm="UndertowRealm" enable-http2="true"/>
                <host name="default-host" alias="localhost">
                    <location name="/" handler="welcome-content"/>
                    <http-invoker security-realm="ApplicationRealm"/>
                </host>
            </server>

{% endhighlight %}

 5. Now we can see the connection to the SSO server is secured by a trusted CA.<br/>
  <img src="/images/quay-sso-sso_trusted.png" alt="IdM users PNG" style="width:50%;"/>

### Appendix 2 - Quay config.yaml

{% highlight console %}
ACTION_LOG_ARCHIVE_LOCATION: default
AUTHENTICATION_TYPE: Database
BITTORRENT_FILENAME_PEPPER: ce8f9a0e-377c-4219-be71-5da2bd063a65
BUILDLOGS_REDIS:
  host: 192.168.122.249
  port: 6379
CONTACT_INFO:
- htttp://help.lab.local
- mailto:help@lab.local
- tel:666-666-666
- irc://#quay@freenode
DATABASE_SECRET_KEY: '114574198320227682855976016094317140393082551235599498954476001569940299217225'
DB_URI: mysql+pymysql://quayuser:redhat@192.168.122.249/enterpriseregistrydb
DEFAULT_TAG_EXPIRATION: 2w
DISTRIBUTED_STORAGE_CONFIG:
  default:
  - LocalStorage
  - storage_path: /datastorage/registry
DISTRIBUTED_STORAGE_DEFAULT_LOCATIONS: []
DISTRIBUTED_STORAGE_PREFERENCE:
- default
ENTERPRISE_LOGO_URL: /static/img/RH_Logo_Quay_Black_UX-horizontal.svg
FEATURE_ACI_CONVERSION: false
FEATURE_ACTION_LOG_ROTATION: false
FEATURE_ANONYMOUS_ACCESS: true
FEATURE_APP_REGISTRY: true
FEATURE_APP_SPECIFIC_TOKENS: true
FEATURE_BUILD_SUPPORT: false
FEATURE_CHANGE_TAG_EXPIRATION: true
FEATURE_DIRECT_LOGIN: true
FEATURE_INVITE_ONLY_USER_CREATION: false
FEATURE_MAILING: false
FEATURE_PARTIAL_USER_AUTOCOMPLETE: true
FEATURE_PROXY_STORAGE: true
FEATURE_REPO_MIRROR: true
FEATURE_REQUIRE_ENCRYPTED_BASIC_AUTH: false
FEATURE_REQUIRE_TEAM_INVITE: true
FEATURE_RESTRICTED_V1_PUSH: true
FEATURE_SECURITY_NOTIFICATIONS: true
FEATURE_SECURITY_SCANNER: false
FEATURE_TEAM_SYNCING: false
FEATURE_USERNAME_CONFIRMATION: true
FEATURE_USER_CREATION: true
FEATURE_USER_LOG_ACCESS: true
GITHUB_LOGIN_CONFIG: {}
GITHUB_TRIGGER_CONFIG: {}
GITLAB_TRIGGER_KIND: {}
GPG2_PRIVATE_KEY_FILENAME: signing-private.gpg
GPG2_PUBLIC_KEY_FILENAME: signing-public.gpg
LDAP_EMAIL_ATTR: mail
LDAP_UID_ATTR: uid
LOGS_MODEL: database
LOGS_MODEL_CONFIG: {}
LOG_ARCHIVE_LOCATION: default
MAIL_DEFAULT_SENDER: support@quay.io
MAIL_PORT: 587
MAIL_USE_TLS: true
PREFERRED_URL_SCHEME: https
REDHATSSO_LOGIN_CONFIG:
  CLIENT_ID: quay-enterprise
  CLIENT_SECRET: 799a974a-4053-48e6-8ee6-51fc514e14ff
  LOGIN_SCOPES:
  - openid
  OIDC_SERVER: https://ice-sso-01.lab.local:8443/auth/realms/lab/
  SERVICE_NAME: LAB SSO
REGISTRY_TITLE: Red Hat Quay
REGISTRY_TITLE_SHORT: Red Hat Quay
REPO_MIRROR_SERVER_HOSTNAME: null
REPO_MIRROR_TLS_VERIFY: true
SECRET_KEY: '<NOT REDACTED>'
SECURITY_SCANNER_ENDPOINT: http://192.168.122.252/securitas
SECURITY_SCANNER_ISSUER_NAME: security_scanner
SERVER_HOSTNAME: ice-quay-01.lab.local
SETUP_COMPLETE: true
SIGNING_ENGINE: gpg2
SUPER_USERS:
- admin
TAG_EXPIRATION_OPTIONS:
- 0s
- 1d
- 1w
- 2w
- 4w
TEAM_RESYNC_STALE_TIME: 60m
TESTING: false
USERFILES_LOCATION: default
USERFILES_PATH: userfiles/
USER_EVENTS_REDIS:
  host: 192.168.122.249
  port: 6379
USE_CDN: false
{% endhighlight %}

### References

 [^1]: <https://access.redhat.com/documentation/en-us/red_hat_quay/3.3/html/deploy_red_hat_quay_-_basic>
 [^2]: <https://openid.net/connect/>
 [^3]: <https://www.keycloak.org/docs/latest/server_installation/index.html>
