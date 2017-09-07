# OpenShift Identity Manager

This quick start for OpenUnison is designed to provide an identity management hub for OpenShift that will:

1. Provide an OpenID Connect Bridge for SAML2, multiple LDAP directories, add compliance acknowledgment, etc
2. Self service portal for requesting access to and getting approval for individual projects
3. Self service requests for gaining cluster level roles
4. Support removing users' access
5. Reporting

The quick start can run inside of OpenShift, leveraging OpenShift for scalability and secret management.  It can also be run externally to OpenShift.  

![OpenShift Identity Manager Architecture](imgs/openunison_qs_openshift.png)

The OpenUnison deployment stores all OpenShift access information as a group in OpenShift, as opposed to a group in an external directory.  The only groups stored outside of OpenShift are approval groups which are stored in the relational database.

# Roles Supported

## Cluster

1.  Administration - Full cluster management access

## Projects

1.  Editors - Can edit and deploy into a project, can not manipulate users
2.  Viewers - Can view contents of a project, but can not make changes

## Non-OpenShift

1.  System Approver - Able to approve access to roles specific to OpenUnison
2.  Auditor - Able to view audit reports, but not request projects or approve access

# Deployment

The deployment model assumes:
1. OpenShift 3.x or higher (Origin or Downstream)
2. An image repository
3. Access to a certified RDBMS (may run on OpenShift)

These instructions cover using the Source-to-Image created by Tremolo Security for OpenUnison, but can be deployed into any J2EE container like tomcat, wildfly, etc.  The Source-to-Image builder will build a container image from your unison.xml and myvd.props file that has all of your libraries running a hardened version of Apache Tomcat 8.5 on the latest CentOS.  The keystore required for deployment will be stored as a secret in OpenShift.

## Getting Started

### Generating Keystore

OpenUnison encrypts or signs everything that leaves it such as JWTs, workflow requests, session cookies, etc. To do this, we need to create a Java keystore that can be used to store these keys as well as the certificates used for TLS by Tomcat. When working with OpenShift something to take note of is Go does NOT work with self signed certificates that are not marked as CA:TRUE no matter how many ways you trust it. In order to use a self signed certificate you have to create a self signed certificate authority and THEN create a certificate signed by that CA. This can be done using Java's keytool but OpenSSL's approach is easier. To make this easier, the makecerts.sh script in this repository (`src/main/bash/makessl.sh`) (adapted from a similar script from CoreOS) will do this for you. Just make sure to change the subject in the script first:

```bash
$ sh makessl.sh
$ cd ssl
$ openssl pkcs12 -export -chain -inkey key.pem -in cert.pem -CAfile ca.pem -out openunison.p12
$ cd ..
$ keytool -importkeystore -srckeystore ./ssl/openunison.p12 -srcstoretype PKCS12 -alias 1 -destKeystore ./unisonKeyStore.jks -deststoretype JCEKS -destalias unison-tls
```

```bash
$ keytool -genseckey -alias session-unison -keyalg AES -keysize 256 -storetype JCEKS -keystore ./unisonKeyStore.jks
$ keytool -genseckey -alias lastmile-oidc -keyalg AES -keysize 256 -storetype JCEKS -keystore ./unisonKeyStore.jks

```

Then the SAML2 RP certificate
```bash
$ keytool -genkeypair -storetype JCEKS -alias unison-saml2-rp-sig -keyalg RSA -keysize 2048 -sigalg SHA256withRSA -keystore  ./unisonKeyStore.jks -validity 3650
```

Import the SAML2 signing certificate from your identity provider
```bash
$ keytool -import -trustcacerts -alias idp-saml2-sig -rfc -storetype JCEKS -keystore ./unisonKeyStore.jks -file /path/to/certificate.pem
```

### Create OpenShift Service Account

The easiest way to create this account is to login to the OpenShift master to run oadm and oc (these instructions from OpenUnison's product manual):

```bash
$ oc new-project unison-service
$ oc project unison-service
$ cat <<EOF | oc create -n unison-service -f -
kind: ServiceAccount
apiVersion: v1
metadata:
  name: unison
EOF
$ oadm policy add-cluster-role-to-user cluster-admin system:serviceaccount:unison-service:unison
$ oc describe serviceaccount unison
$ oc describe secret unison-token-XXXX
```

In the above example, XXXX is the id of one of the tokens generated in the `Tokens` section of the output from the `oc describe serviceaccount unison`.  The final command will output a large, base64 encoded token.  This token is what OpenUnison will use to communicate with OpenShift.  Hold on to this value for the next step.

### Create Environments File

OpenUnison stores environment specific information, such as host names, passwords, etc, in a properties file that will then be loaded by OpenUnison.  This file will be stored in OpenShift as a secret then accessed by OpenUnison on startup to fill in the `#[]` parameters in `unison.xml` and `myvd.conf`.  For instance the parameter `#[OU_HOST]` in `unison.xml` would have an entry in this file.  Below is an example file, this file should be saved as `ou.env`:

```properties
OU_HOST=openunison.demo.aws
OU_HIBERNATE_DIALECT=org.hibernate.dialect.MySQL5InnoDBDialect
OU_JDBC_DRIVER=com.mysql.jdbc.Driver
OU_JDBC_URL=jdbc:mysql://mariadb.openunison.svc:3306/openunison
OU_JDBC_USER=unison
OU_JDBC_PASSWORD=start123
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=XXXX
SMTP_PASSWORD=XXXX
SMTP_FROM=donotreply@tremolosecurity.com
SMTP_TLS=true
OU_JDBC_VALIDATION=SELECT 1
OPENSHIFT_CONSOLE_URL=https://openshift.demo.aws:8443/console/
OPENSHIFT_URL=https://kubernetes.default.svc:443
OPENSHIFT_TOKEN=eyJhbG.....
OU_OIDC_OPENSHIFT_SECRET=secret
OU_OIDC_OPENSHIFT_REIDRECT=https://openshift.demo.aws:8443/oauth2callback/openunison
OPENSHIFT_OU_HOST=openunison.openunison.svc
unisonKeystorePassword=xxxx
IDP_POST=https://idp.ent2k12.domain.com/adfs/ls/
IDP_REDIR=https://idp.ent2k12.domain.com/adfs/ls/
```

### Export SAML2 Metadata

Once your environment file is built, metadata can be generated for your identity provider.  First download the OpenUnion utilities jar file from `https://www.tremolosecurity.com/nexus/service/local/repositories/betas/content/com/tremolosecurity/unison/openunison-util/1.0.12.beta/openunison-util-1.0.12.beta-jar-with-dependencies.jar` and run the export:

```bash
$ $ java -jar ./openunison-util-1.0.12.beta.jar -action export-sp-metadata -chainName enterprise_idp -unisonXMLFile /path/to/openunison-qs-openshift/src/main/webapp/WEB-INF/unison.xml -keystorePath ./unisonKeyStore.jks -envFile ./ou.env -mechanismName SAML2 -urlBase https://openunison.demo.aws
```

Make sure to replace the `-urlBase` with the URL user for accessing OpenUnison.  It should use the same host as in OU_HOST.  This command will generate XML to the console that can be copied&pasted into a file that can be submited to your identity provider.


### Configure Identity Provider

Once the OpenUnison metadata is imported, make sure the following attributes are in the assertion:

| Attribute Name | Active Directory Attribute | Description |
| -------------- | -------------------------- | ----------- |
| uid            | samAccountName             | User's login id |
| givenName      | givenName                  | User's first name |
| sn             | sn                         | User's last name |
| mail           | mail                       | User's email address |

If using Active Directory Federation Services, you can use the following claims transformation rule:
```
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier", "uid", "givenName", "sn", "mail"), query = ";sAMAccountName,sAMAccountName,givenName,sn,mail;{0}", param = c.Value);
```

### Create the OpenUnison Project

```bash
$ oc new-project openunison
$ oc project openunison
```

Next create a secrets file for the `unisonKeyStore.jks` and `ou.env` file base base64 encoding each file and adding it to the `src/main/yaml/openunison-secrets.yaml` file.  Once the secrets are added, the secrets file can be added to your project:

```bash
$ oc create -f ./openunison-secrets.yaml
```
