SimpleSAMLphp
=============
[![Build Status](https://travis-ci.org/simplesamlphp/simplesamlphp.svg?branch=master)]
(https://travis-ci.org/simplesamlphp/simplesamlphp) [![Coverage Status](https://img.shields.io/coveralls/simplesamlphp/simplesamlphp.svg)] 
(https://coveralls.io/r/simplesamlphp/simplesamlphp)

This is the official repository of the SimpleSAMLphp software.

* [SimpleSAMLphp homepage](https://simplesamlphp.org)
* [SimpleSAMLphp Downloads](https://simplesamlphp.org/download)

Please, [contribute](CONTRIBUTE.md)!

----------------------------------------------------------

##How to configure simpleSAMLphp 1.3 as SP and Shibboleth 2.1 as IdP ?
====================================================================

I suppose here you already have a server with a working Shibboleth 2.1 IdP at this address:

`https://your-idp-host/idp/shibboleth`

We will explain now how to configure simpleSAMLphp 1.3 as a Service Provider (SP) relying on the Shibboleth IdP for the user's authentications.

###simpleSAMLphp installation/configuration

First of all you have to install simplesaml:

```
cd /var
git clone https://github.com/srprasanna/simplesamlphp.git simplesamlphp
cd simplesaml
cp -r config-templates/*.php config/
cp -r metadata-templates/*.php metadata/
```

Then configure your apache server to map this path
```
/var/simplesamlphp/www
```
to this url (using https is not required):
```
http://your-sp-host/simplesaml
```

To accomplish this task, you can simply add this directive in your apache configuration:

```Alias /simplesaml /var/simplesamlphp/www```

Now you have to configure it as a SP:

Edit /var/simplesamlphp/metadata/saml20-sp-hosted.php and add this metadata to the array:
```
  'your-sp-id' => array(
        'host' => 'your-sp-host',
        'certificate' => 'server.crt',
        'privatekey'  => 'server.pem',
  ),
```

your-sp-id is the string used to identify your SP to other IdP, you can change it if you want.

server.crt and server.pem are public and private keys of your SP certificate located in /var/simplesamlphp/cert/. 

This certificate will be published in the SP metadata and then will be used by Shibboleth to encrypt the transmitted data (assertions).

Edit /var/simplesamlphp/metadata/saml20-idp-remote.php and add this metadata to the end of the file:

```
$metadata['https://your-idp-host/idp/shibboleth'] = array (
  'name' => 'The sexy name of your IdP',
  'description' => 'The description of your idp',
  'SingleSignOnService' => 'https://your-idp-host/idp/profile/SAML2/Redirect/SSO',
  'certFingerprint' => 'xxx',
);
```
certFingerprint can be calculated from your Shibboleth IdP certificate this way:

```cat idp.crt | openssl x509 -fingerprint  | grep SHA1 | sed "s/^[^=]*=//g" | sed "s/://g"```

(In a default shibboleth installation, idp.crt is located in shibboleth-idp/credentials/)

###Shibboleth 2.1 configuration

Last step is to configure Shibboleth to handled simpleSAMLphp specificities:

Edit shibboleth-idp/conf/relying-party.xml and just after the DefaultRelyingParty entry, add this XML block:
```
    <RelyingParty id="your-sp-id"
                  provider="https://your-idp-host/idp/shibboleth"
                  defaultSigningCredentialRef="IdPCredential" >
       <ProfileConfiguration xsi:type="saml:SAML2SSOProfile"
                             encryptNameIds="never"
                             encryptAssertions="never"
                             />
    </RelyingParty>
```

This part of code will override the default profile only for your SP. It will disable the encryption of the NameIDs which is not yet supported in simpleSAMLphp. More informations about the NameIDs problem can be found in this thread. In addition, there is also a discussion about removing the NameIDs encryption in the default shibboleth idp configuration.
Notice : in the next 2.2 shibb release, NameIDs encryption will be disabled by default in the shibboleth configuration.

Configure a new metadata provider for this SP in shibboleth-idp/conf/relying-party.xml:
```
<MetadataProvider id="MDSIMPLSAMLPHP" 
                  xsi:type="ResourceBackedMetadataProvider" 
                  xmlns="urn:mace:shibboleth:2.0:metadata">
  <MetadataResource xsi:type="resource:FilesystemResource"
                    file="/path/to/your/sp/metadata/shibboleth-idp/metadata/yoursp-metadata.xml" />
</MetadataProvider>
```

I used the ResourceBackedMetadataProvider type which just reads data from a static file because Shibboleth 2.1 doesn't support yet HTTP proxies for the ''FileBackedHTTPMetadataProvider'' type. So you'll have also to configure a crontab to retrieve periodically fresh metadata from your SP. For example your can use this:
```
0 * * * * wget http://your-sp-host/simplesaml/saml2/sp/metadata.php -O /path/to/your/sp/metadata/shibboleth-idp/metadata/yoursp-metadata.xml
```

Restart your shibboleth server

Now you should be ready to test it ! Try to open

```
http://your-sp-host/simplesaml/example-simple/saml2-example.php
```

