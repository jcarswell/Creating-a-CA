# Opening Remarks

What is this document intended to be and what is it not intended to be? I am putting this together for two reasons one, so that I can get a better idea of what is involved in managing a trust and two to provide a comprehensive guide one setting up a trust for you organization. This is not intended to replace the excellent, though some time lacking, documentation on the OpenSSL toolset, or to provide an in depth analysis on the RFC 5280 standard. The information herein is provide as a staring point to understanding the in's and out's of how certificates function as it relates to setting up an OpenSSL based trust. I can almost guarantee that there will be incorrect or incomplete information, I am not an expert on managing a trust. 
I have found several useful articles out on the intra-webs that I have found provided a good starting point for me on getting going but nothing that was comprehensive enough that I didn't need to go and find five other articles and forms that helped me overcome many of the challenges that I have faced in setting up a basic internal PKI. That being said, one of my main intents with this path is to open up the possibility of easily adding various other elements into the mix that help make managing a PKI for an enterprise far easier.
As I continue the journey of Understanding the in's and out's of this complex beast, or I find that Google, or others but mainly Google, have decided that certificates with or without certain extensions defined that they will make the site invalid, I will continue with the intention of keeping this up to date. If you find that there is an issue or have some insights that I should add to the document please leave a comment or drop me a line.

Some useful links
+ [IETF RFC5280](https://tools.ietf.org/html/rfc5280) - Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile
+ [Wikipedia](https://en.wikipedia.org) - Articles written by other peoples on the internet  
  Articles to check out (likely referenced several times throughout):
  - Public Key Infrastructure
  - X.509
  - Certificate Authority
+ [OpenSSL Manual Pages](https://www.openssl.org/docs/manmaster/man1/) - Alternatively these are available on most Linux system with the OpenSSL libraries installed
+ [OpenSSL Relevant Standards](https://www.openssl.org/docs/standards.html) - There just a few
+ [Bulletproof TLS Newsletter](https://www.feistyduck.com/bulletproof-tls-newsletter/) - Just subscribe to this unless you don't want the best source of SSL/TLS news available
+ [Bulletproof SSL and TLS](https://www.feistyduck.com/books/bulletproof-ssl-and-tls/) - If you want the best documentation ever on SSL and TLS this is it, though it's slightly longer than this document


# Notes on security:

Your CA is only as secure as you make it, if you keep you certificate files on a encrypted USB stick and store it in a vault that only designated people have access to, your probably going down the right path. On the flip side if you are running this on a internet exposed server that hasn't been patched in years and has a password of 'Passw0rd' then your definitely doing it wrong and need to re-evaluate your security practices before implementing a trust system.


# Definitions
Yes I know everyone's favorite thing, don't worry there will be a spelling test at the end :)

Public Key Infrastructure (PKI) - This is the processes behind a trust. This is makes up all the induvial components with in a trust. A PKI defines the roles, policies and procedures  on managing the connections between the parts and the parts themselves.

Certificate Authority (CA) - This is the issuer of trusted certificates. This takes the signing request that is generated based on the private key and creates the public key that a client can then use to validate that the private key is trusted.

Certificate - This is a combination of a key and information about that key such as its name and validity period. This can easily be confused with a key but the main difference being that a key is used to encipher or decipher data while the certificate contains information that makes the key trustable. One key can be used for many certificates but a certificate can only be used with one key.

Privacy-Enhanced Mail (PEM) - Ya, I bet that cuaght you off gaurd too... we'll just for get what PEM stands for. PEM is the file format used to store cryptographic information such as keys and certificates. When you see an extension such as .crt .ket or .pem  it is very likely a base64 encoded file containing cryptographic data. So when you see some one calling there files myname.crt.pem and a second person calling theirs myname.crt know that these are the same, just one follows proper file naming conventions and the other doesn't. I would fall in later category, I like the distinguishable extensions, then there's OpenSSH which simply forgoes the extensions other that .pub to indicate the public key.

Key - This is the actual piece that encrypts and decrypts data. There is normally two key generated a private key, used to decipher data, and a public key, used to encipher data.

Certificate Revocation List (CRL) - A really ugly file that contains a list of all certificate serial numbers that have been revoked by the trust

Online Certificate Status Protocol (OCSP) - This enables application to determine the validity of a certificate by querying a server

OCSP Stapleing - This isn't really a term that you need to know to manage a CA but it is something that I will cover briefly here. OCSP Stapling is the process of keeping a local (webserver) copy of the CA's OCSP validation for a certificate reducing the number of additional calls a browser needs to make to validate that the certificate is valid. And allowing limited continuity in the event that the PKI is offline.

Trust - This is a general term that refers to the PKI, but more specifically the trustworthiness of the authority.

# The OpenSSL Configuration file
1. The OpenSSL config file and some background.
   - Lets start with this, first grab the openssl.cnf example from [here \[CA/intermediate/openssl.cnf\]](..blob/master/CA/intermediate/openssl.cnf). Open the file up in you favorite text editor and lets take a quick peak into the configuration structure.
   - File sections, the OpenSSL configuration files uses the ini configuration standard, see [Wikipedia INI file](https://en.wikipedia.org/wiki/INI_file) for more detailed information the informal standard. Most of the section names are flexible as they are referenced at various points in the configuration files as are not directly referenced by OpenSSL. Sections that are directly referenced by OpenSSL are all of the standard commands, ca, crl, req, ocsp, etc. That's not to say that you couldn't use those name wouldn't be true however using them would either cause OpenSSL to outright fail to do anything or make unexpected changes to the action you are performing. This in mind this is countered by only having an effect when you actually call in the configuration file against the specific module, so to make things simple only use the sections as they are intended.  
     Continuing on this point, manual pages (man) are your friend and the OpenSSL documentation within is very extensive starting at OpenSSL as the main library documentation, `man openssl`, and then each of the OpenSSL modules, ca, ec, req, ts, etc. to look up the individual documents simply address the module directly from man, `man ca`, `man req`, `man x509`, etc.
   - Module sections, as previously stated, these are the sections that OpenSSL checks when you call the module with the config flag. These are sections are normally pretty simplistic and tell OpenSSL that certain things are required or simply where to look during to get the defaults for the module. Examples of this are the `[ ca ]` section which tells the OpenSSL ca module which CA specification to look at, more on this later. Or the `[ req ]` section which tells the OpenSSL req(uest) module specification that are needed to perform the actions and ensure that specific fields are present before creating the request and passing on the request for signing. These sections however aren't necessarily required as most of the option can be set via flags or simply using the OpenSSL defaults as is. Using the defaults does have some potential drawbacks when you are building a CA for a organization or even as a public CA due to the ever evolving standards and security improvments that need to be tweeked to keep up with the changes that certain organization are enforcing.
   - CA Specifications: If you are going to use OpenSSL to manage the CA, these specs are _really_ important. You will need at minimum two of these, unless you are getting a publicly signed CA certificate, one for your root certificate and one for you intermediate certificate, or more simply put one strict setting and one loose setting.
   - Certificate Specifications: These are the opposite of the CA specifications, and you will likely have many more of these, including one for each CA specifications as these are certificates as well. We'll go deeper into these later on.
   - The rest of the sections are to support the other sections where an option requires it own set of configurations.
2. The OpenSSL config file (root edition) - If you are planning on getting a signed CA certificate from the likes of Comodo, DigiCert, etc skip to the end.
   1. To start, grab the OpenSSL configuration file from [\[CA/root/openssl.cnf\]](..blob/master/CA/root/openssl.cnf). Once you have this saved open it up and we will go over it piece by piece, because I know you were excited for some light bed time reading.
   1. Section `[ ca ]`: OpenSSL Module definition
      * You still here? Good. This one is pretty easy, it starts with some basic info on this section as it is one on of the OpenSSL module section, I will skip over the comments for now as we will look at these in a later section. This has a few configuration options but we really only care about one, default_ca, there is also preserve and msie_hack. I'm not going to go into too much detail on the specific configuration as that is all very well documented in the manual pages.
      * default_ca: Short and sweat, this tells the OpenSSL CA module which CA specification that you want to use.
   1. Section `[ root ]`: CA specification
      * I think this could be a little more obvious... this is the first CA specification, and only one in this file. This is kinda an important section this basically tells the CA module what to do to sign a certificate, what requirements the request must meet in order to be signed, what needs to be part of the signed certificate and where to store the signed files
     * Storage: the first 11 items are all related to the files that are needed in order to store files, correctly sign certificate and ensure that if you have to revoke a certificate it happens properly. This section, like most is highly customizable depending on how you plan on setting up and configuring you certificate authority. The settings laid out in the configuration files are based on the directory prep that was done previously.
      * `dir`: this isn't actually a configuration item it's a variable. based on the provided template this is saying that when you are using this CA spec you will be in the directory above, the provided template this is the 'CA' folder, and that the files for this CA spec will be in the 'root' directory. If this is you first CA that you are setting up, just copy the template down and leave this as is, when you are building the production CA make sure that this is setup correctly depending on how you have laid out your directory structure. The good news is you can move your 'root' folder wherever you want can call it whatever you want at any given point in time as long as you can access the files and this path is correct.
      * `certs`: [Remove Me] This is a variable to the certificate storage location
      * `crl_dir`: [Remove Me] This is a variable to the crl storage
      * `new_certs_dir`: This is the local storage for all signed certificates, while the files are the same as the output certificate these are not intended for the same usage. The certificates used here are so that when you revoke a certificate by serial number OpenSSL (the CA) can reference the original certificate without having it on hand. This will also be used by the CA when you generate your OCSP and CRL records to determine what is valid and what isn't valid
      * `database`: As the name implies, this is the file that is responsible for all of the certificates that have ever been handled by the CA. That's to say that it is used in conjunction with `new_certs_dir` to manage the CA.
      * `serial`: This is the serial number which you prepopulated in the previous section this file is a counter to indicated the next unique serial number in when signing a request.
      * `RANDFILE`: This one requires a bit of background on how cryptography work. For the time being just know that this is an important piece to building a strong trust in conjunction with the private key.
      * `private_key`: We will get into this later, but needless to say this points at your private key
      * `certificate`: This here is the (self) signed certificate slash your public key or at least this tells OpenSSL where to look to find it.
      * `crlnumber`: This is similar the serial file with the exception that it is the "serial number" for your CRL.
      * `crl`: This is the location that the CRL is stored
      * `crl_extensions`: this tells OpenSSL which section to look at for the CRL generation configuration
      * `default_crl_days`: This is how long your CRL is valid for, in days
      * `default_crl_hours`: This is the same as `default_crl_days` however this specifies CRL validity in days
      * `default_md`: This is what will be used for your message digest, any of the md's supported by your current version of OpenSSL. At a minimum you should use sha256 or sha512.
        - For a full list of available digests avalible use `openssl list-message-digest-algorithms`
      * `name_opt` and `cert_opt`: This tells OpenSSL how to display the certificate when you are generating the certificate. For both of these options use `ca_default` to use the new OpenSSL defaults for displaying the certificates as the old view is depreciated however is the default if not specified.
      * `default_days`: The default validity period used when signing certificates. For server certificates, the typical recommendation is around 375 days to add some give in leeway in the event that the certificate can be changes at the end of 1 year.
      * `copy_extensions`: This should be used to set only as `none`. This tells OpenSSL whether it should copy extensions from the request or not.
        + Quote from manual pages:
          
          > The copy_extensions option should be used with caution. If care is not taken then it can be a security risk. For example if a certificate request contains a basicConstraints extension with CA:TRUE and the copy_extensions value is set to copyall and the user does not spot this when the certificate is displayed then this will hand the requester a valid CA certificate.  
          > This situation can be avoided by setting copy_extensions to copy and including basicConstraints with CA:FALSE in the configuration file. Then if the request contains a basicConstraints extension it will be ignored.
      * `policy`: This tells OpenSSL which section to look at for the CA signing policy
   3. Section `[ policy_strict ]`: CA Policy
      - Okay lets start with what this is section is. This is tied to the `policy` field in the CA specfication section. This tells OpenSSL what to do with each of the Distinguished name (DN) fields. There is three valid arguments for each of the DNs, match, supplied and optional.
      - DN: The distinguished names are countryName, stateOrProvinceName, orginizationName, orginizationalUnitName, commonName and emailAddress each of these should be specified along with the one of the aforementioned values depending on the requirements of the specific CA policy
   4. Section `[ req ]`: OpenSSL Module definition
      - `default_bits`: The default key size this will correspond with your chosen digest algorithm. As a minimum this should be 2048, which is the current minimum supported key length by browsers when using the sha digest, however for better future compatibility choose 4096 especially for the CA. using a longer key length doesn't inhibit your ability to sign request that use a shorter key length, though based on current standards unless under extraordinary circumstances, anything under 2048 should not be permitted for signing.
      - `default_md`: Same deal as default_md from the CA definition, this defines the message digest to use for creating the key and further generating the signing request.
      - `string_mask`: This option masks out the use of certain string types. The current best practice is to use utf8only given the multilingual nature of the web today.
      - `utf8`: This tells the request to use utf8 string format over ASCII when set to `yes`. If this isn't present it is the same as using `no`, which uses ASCII strings instead.
      - `distinguished_name`: This tells OpenSSL where to look to determine what DN's are required in the signing request.
      - `req_extensions`: This tells OpenSSL where to look for extensions to include in the signing request, this feild only specifies X.509 version 1 & 2 fields
      - `x509_extensions`: This is essentailly the sames as `req_extension` but is for X.509 veraion 3 fields. With out this, you wont be able to generate a valid ssl certificate.
      - With all of this in mind, this isn't necessarily needed in the configuration that is included with the CA configuration however having this in the template that is used when generating a request is importation to generate a valid certificate under the current accepted standard.
   5. Section `[ req_dn ]`: Configuration Section `distinguished_name`
      - This sections has two components, What distinguished names (DN) are prompted for in request, and the user friendly names for them, and if you have any defaults for those DN's as well as minimum and maximum lengths. The list of common DN's is below of which there is seven of them, when specifying defaults, minimumns and maximums for each DN it will simply be the `distinguished name_default`, `distinguished name_min` and `distinguished name_max` for example `localityName_default`, `commonName_min` and `countryName_max`
         * countryName (c)
         * stateOtProviceName (ST)
         * localityName (L)
         * 0.orginizationName (O)
         * commonName (CN)
   6. Sections Extension Definitions: Configuration Section `req_extensions` and `x509_extensions`
      - This Section has the sole function of defining the extensions and their values for the signing request. Under todays current standard you must have the subjectAltName defined in order to create a valid SSL certificate. This section also say what the certificates intended use is.   
        When Using a separate configuration section, you must append the OID with a position indcator, where as using a comma separated list you don't need to. To do this you would simply do `IP.1` or `DNS.2`. The position indicator should increment per OID, and not based on item number in the list. Either method is perfectly acceptable; however, one provides for better readability and entry for those inexperienced and the other is more efficient.
      - `subjectKeyIdentifier`: "The subject key identifier extension provides a means of identifying certificates that contain a particular public key." From RFC5280 Page 28. The jist of this is for OpenSSL it should be `hash`
      - `authorityKeyIdentifier`: This feild is used to validate the hash of the public key that corresponds to the private key used to sign the certificate. When you originally generate the root certificate this field may be ommitted as it is a self-signed certificate with no matching public key however this field becomes critical when you are changing over the root CA.
      - `keyUsage`: leaving this field out is typically okay as most CAs will add the appropriate definitions in when signing the certificate and will completely ignore this field. It is good to have a basic understanding of this though. There is two pieces to this puzzle the first is this field and the second is the `extendedKeyusage` field together the tell the device that is using the certificate what it is intended for and provides a measure of control over the certificates and their use providing additional security. Ensuring that both fields are defined ensure that your public and private key pair that is intended to protect your web server isn't used to identify users.  
         This field is normally prefaced with the word critical followed by one or more valid usage names. If you are filling in this field and generating a CSR that will be signed by a public certificate authority, using critical in this field is not a good idea as it will likely cause the signing request to be rejected by the certificate authority. The valid names that are supported are digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment, keyAgreement, keyCertSign, cRLSign, encipherOnly and decipherOnly. For more details on key usage take a look at the [IEEE RFC5280 Section 4.2.1.3](https://tools.ietf.org/html/rfc5280#page-29) this gives a very indepth view of what each of these options are for.
   7. Section `[ crl_ext ]`: Configuration Section `crl_extnesions`
      - This section tells OpenSSL to generate a x509 v2 crl certificate. Currently there is only one extentsion defined `authorityKeyIdentifier` and that ensure it is matched agains the root certificate key and is harder to spoof. X.509 v2 CRLs have several supported extensions that you can review by taking a peek at [RFC 5280 Section 5.2](https://tools.ietf.org/html/rfc5280#section-5.2)
1. OpenSSL config Intermediate edition
   1. This section should be short, sweet and to the point as there is no need to cover things twice. So to start grab a copy of the [intermediate openssl.cnf [CA/intermediate/openssl.cnf]](../blob/master/CA/intermediate/openssl.cnf) file and open it up in your favorite text editor.
   2. Section `[ intermediate ]`
      - This section is equivelent to `[ root ]` under the root openssl.cnf file and contains no additional configuration options over the `root` configuration.
   3. Section `[ policy_loose ]`: CA Policy Configuration
      - This section will be the same, or nearly the same, for all CA `policy` sections, here there is no additional or missing configuration. Any DNs that are not specified will be the same as having listed as optional
   4. Sections `[ req ]`, `[ req_dn ]`, and `[ req_ext]`
      - These section have no changes as opposed to the root configuration.
   5. Sections `[ req_extv3_* ]`: X.509 v3 certificate configurations
      - First, there is a few more of these then under the root configuration, user, server and ocsp. Each of these have there own usages as the name suggests user for user identification certificates, server for server identification, and ocsp which is a special one that is used to sign the OCSP certificate.
      - `authorityInfoAccess`: This section added the OCSP server information to the signed certificate. This section is included here and not in under the root configuration, as your intermediate will typically be online vs offline where CRLs are easier to distribute.
   6. Sections `[ crl_ext ]`
      - This is exactly the same and should be no need to change this.
   7. Section `[ crl_intermediate ]` and `[ crl_issu_intermediate ]`: `crlDistribuitonPoints` configuration section
      - The only differance in these sections is that they identify the intermediate issuer and were to find the certificate revokation list for the intermediate issuer
3. OpenSSL Configuration Request edition
   1. To start to grab the [csr.cnf config [CA/common/1.Template/csr.cnf]](../Blob/master/CA/common/1.Template/csr.cnf) and open it up in you favorite text editor.
   2. Section `[ req ]`: OpenSSL req module configuration
      - This is difference here versus in the the `[ req ]` section is that this is setup to create the certificate signing request versus telling OpenSSL what information to collect to create the certificate signing request. With that said, I still wont cover the items that have been covered in previous sections
      - `subjectAltName`: This can be one of two options, a reference to a separate configuration section or a comma seperated list using the OID:Value pairing. Valid OID for this are email, URI, DNS, RID, IP, and dirName. see `man x509v3_config` for more details. Knowing those values the primary values that will be used are DNS, for the DNS resolvable name and IP for the host IP address(es). 
      - `attributes`: While this isn't in the configuration file, you could add this to add required attributes to the signing request. You would point this to a new section, and include things such as `challengePassword` or `unstructuredName`.
      - `encrypt_key`: This tells OpenSSL when creating a new key to not password protect the private key. This option is equivelent to `-nodes` when using openssl
      - `prompt`: This tells OpenSSL to not ask for any information during the request, and only use the configuration file.
   3. Section `[ req_dn ]`
      - Unlike in the root and intermediate configurations, instead of telling OpenSSL what DNs to prompt for and the user friendly names and their defaults. This section is used to define what the DN values are. Though they can still be used for the latter
   4. Section `[ req_ext ]`
      - This section has one new field and two fields that are missing
      - `authorityKeyIdentifier`: As this is a field that the CA will add, don't add this to the request
      - `keyUsage`: As the certificate authority will define this field, we leave it out
      - `extendedKeyUsage`: Though this field may be defined by the CA, it isn't a bad idea to defined the field in the event that the CA doesn't define the field.
   5. Section `[ alt_names ]`
      - This section can be used as anlternative to defining all of the subject alternative names on one `subjectAltNane` line.


# Building up the CA
Okay time for the nitty gritty, I will try not to make this boring while including as much information to help you build a proper CA with the right amount of dangerousness to wreak havoc. I do strongly encourage that you start by building a test CA that you can issues some certificates from to test the configuration and ensure that you have a good grasp on what the fridgesicles is going on as some change are near impossible to make on your production CA without re-issuing all of your intermediate CA's or worse, doh.

1. Build the directory structure
   - There should be 3 directories, one for your root authority, one for your intermediate, and one as a common directory for certificate signing
   - Your directories can be either simple names or logical names, this is your preference, a simple name would be like 'root' or a logical name would be symbolic of the certificates name for example, my certificates name would be 'Silly CA EC1' therefor my logical name would be 'EC1'
   - Three directories is just a starting point, if you need 17 intermediate certificates then do that instead, ultimately you and your team need to be able to manage the PKI in a manner that suites the goals of the trust
2. Initialize the certificate folders
   - You will need to create several folders and files within each of the certificate folders. These folders are certs, crl, newcerts, private and the files are cert.db, serial and openssl.cnf, more on this in a moment.
     * Firstly, as you will see in a bit, none of the folder/file names are fixed they can be configured in the OpenSSL config file
     * openssl.cnf: this is the default name for an OpenSSL config file and makes it easier for someone down the road to determine what the F' is going on, provided that they have some background in managing CA's the improper way, using OpenSSL utility. This can be changed to simplify the workflow and can be stored in completely different directory, the caveat being that the user managing the CA will still have to be able to access both this file and the certificate folder(s). You could also use a single files to manage all of the certificates
     * serial: this is an important one and it's use may not be immediately obvious, this is the certificate serial number, stored in hex, this will be the field that you need to know in order to revoke certificates that you don't have the original certificate file from. so when you initialize this value ensure that you are using a good serial number that should remain unique.
   - Initialize the serial number, yes I know I'm going to repeat myself. Most of the time when you are reading articles on setting up a CA they recommend that you initialize this file with `1000`, now if you are just building a test CA this is perfectly OK; however, if this will be your production CA, DO NOT use this, use some this that is unique to your organization, like a street address, founding year or even the CA/Company Name, converted to hex, though it might get a little long, and use that as your prefix followed by enough length that the serial will stay unique, so `101000000000` where `1010` is my prefix. to create the file simply do `echo [your serial number] > serial` assuming that you are in the certificate folder
   - Initalize your crl number, this is similar to the serial number but in that it is a hex value however in this case you can simply start with `00` if you like
   - Initalize the certificate database, cert.db, by simply creating an empty file named cert.db
   - Initailze the random number seed this one is easy simply run `openssl rand -out private/.rand 4096`, you'll want to do this once for each of your CA's
     - Special note: if you are running this in a virtual environment and it is taking a long time to create the file or just fails outright, this is a know issue due to the fact that there is not physical devices attached to the server. So either generate this file on your workstation or from a busy server, but preferably from a physical system.
   - Build your OpenSSL config. Just kidding, I can't do this here there is way to much detail to do it right, so go to jail, do not pass go and do not collect 200 dollars.
1. Getting the Root certificate built  
   Provided that you have read the previous section, you should have a pretty good grasp on everything that is going on in the OpenSSL configutation file. Here we will take that configuration file and put it into practical use, starting with customizing the file to meet the needs of your root certificate authority.   
   Note: if you are looking to get a publicly signed intermediate certificate skip over this sections and start at the intermediate section
   1. Modification to the configuration file
      - First and formost if you want to call your root certificate something other than root now is a good time to replace all of the referances to root with what ever you want, as long as there isn't any spaces in the name. For the remainder of this document I will refer to it as root though...
      - `[ root ] -> dir`: Make sure that this is set so that OpenSSL can access the files. This can either be a soft referance, such as `~`, `.` or `..` followed by the path or hard links such as `/root/MyAwesomeCA/root/`
      - `[ root ] -> default_days` and `default_crl_days`: You will have to set this based on your PKI policies, just remeber that this one dosen't impact the server and client certificates only the intermediate certificate authorities this is typically five to ten years in length. Also as you will likely be keeping your CA offline, in accordace with best practises, you will not want to use a short CRL validity period unless you require it.
      - `[ policy_strict ]`: You will wan't to set this up in accordance with your PKI policy; however, the provided configuration is good for most orginizations. The parts that you will not want to set to match are `commonName` and `organizationalUnitName` as `commonName` must be different but you shouldn't sign anything with out one so it must be `supplied` and if you use the `organizationalUnitName` as either as such or as a descriptor field than having this set to `match` will cause the signing to fail. Setting this eiter to loose or two tight to start may cause you grief down the road.
      - `[ req_dn ]`: Customize the defaults to your liking if you wish, just note that when we get to requests this section isn't actually used.
      - `[ crl_root ] -> fullname`: You will want to put a little bit of thought behind this before you get to far ahead, that being said it doesn't really effect anything until you get down to signing the intermediate certificate.
      - `[ req_extv3_intermediate ] -> default_days`: You will want to set this in accordance with your PKI policies for the lenght of your intermediate certificates. This value can be overwritten at runtime with the `-days` flag when you are signing the request
      - `[ crl_issu_root ]`: same as above, Though you will want to ensure that this is set correctly otherwise your CRL checks will fail and it will look like everything is invalid
   2. Generate the root certificate
      1. Grab a copy of the [csr.cnf [CA/common/1. Template/csr.cnf]](..blob/master/CA/root/openssl.cnf) and open it up in your favorite text editor
      2. Modify the following fields
         - `[ req ] -> default_bits`: Change this to 4096 or better, as this is the heart of your trust make it as strong as possible
         - `[ req ] -> encrypt_key`: Change this to yes, unless your pki policy states otherwise. In the event that this some how leaks you wan't to limit the possibility of Joe Blow being able to use this to generate his own certificate. You can also change this password at anytime in the future.
         - `[ req_dn ]`: Complete this section in whole. The `commonName` can be pretty much anything as long as its utf8. You may also want to include the `orginizationalUnitName` as a certficate descriptor.
         - `[ req_ext ] -> subjectKeyIdenifier`: As this is for the Root certificate which will be self signed, you will want to add this field - `subjectKeyIdentifier = hash`
         - `[ req_ext ] -> authorityKeyIdentifier`: Again you will want to add this field - `authorityKeyIdentifier = keyid:always,issuer`
         - `[ req_ext ] -> basicConstraints`: Again you'll want to add this field - `basicConstraints = critical, CA:true`
         - `[ req_ext ] -> keyUsage`: This is the last one to add - `keyUsage = critical, cRLSign, keyCertSign`
         - `[ req_ext ] -> extendedKeyUsage`: Comment out or remove this line, as it isn't needed
         - `[ alt_names ] -> DNS.1`: Comment out or delete this field as it isn't needed for this certificate
      3. Save that file under your root CA folder as root-ca-req.cnf
      4. Run the following command while in your root ca folder  
         `openssl req -new -config root-ca-req.cnf -newkey rsa -rand private/.rand -days <number of days> -x509 -set_serial <your first serial number> -keyout private/root.key -out certs/root.crt`.  
         To break this down, you are asking the OpenSSL req module to create a `-new` request using the `-config` located at the file you just saved, you want to generate a `-newkey` using `rsa` as your algorithm and the length as specified in yout config. 
         You want to seed this certificate using the `-rand` file located under `private/.rand` as generated in section two. 
         As we are generating a certificate not a signing request we want the certificate to be valid for the number of `-days`, this would typically be 10 to 20 years for your root certificate. 
         You are then telling OpenSSL that you want a certificate by specifing `-x509`, and that certificate should have a `-serial` number of whatever you specify, this can either be a decimal value or a hex value by specifing `0x` before you number, just make sure to not include spaces. 
         And finally you are going to output you key, `-keyout`, to the path as defined in your `openssl.cnf` and `-out`put you certifcate to the path as defined in your `openssl.cnf`
      5. So now you have your signed certificate, with this you have two new files your private key along with you public certificate, which containts your public key. The first file is the private key which is under `private/root.key`, this is the piece that needs to be protected and never lost. The private key is the what will be used to sign certificates, including intermediate certificate authority. The other file, which you should grab a copy of, is `cert/root.crt`, this is the public certificate, this is what will be used to validate all certificates that are signed by the trust. For AD evironment, throw this under group policy as a trusted root certificate authority to distrubute to your enpoint devices, or under Linux, add this to ca-certificates.
      6. Go back and populate the `[ crl_issu_root ]` section with the details of your newly minted root Certificate Authority as well as the ` [ crl_root ] -> fullname` if you haven't done so yet.
1. Intermediate Certificate  
   If you are strictly looking at getting signed intermediate authority certificate this is the section that is most applicable to you.
   1. Modification to the configuration file  
      Go Ahead and grab the [ Intermediate openssl.cnf [CA/intermediate/openssl.cnf]](..blob/master/CA/intermediate/openssl.cnf).
      - First and formost if you want to call your intermediate certificate something other than intermediate now is a good time to replace all of the referances to root with what ever you want, as long as there isn't any spaces in the name. For the remainder of this document I will refer to it as intermediate though...
      - `[ intermediate ] -> dir`: Make sure that this is set so that OpenSSL can access the files. This can either be a soft referance, such as `~`, `.` or `..` followed by the path or hard links such as `/root/MyAwesomeCA/intermediate/`
      - `[ intermediate ] -> default_days` and `default_crl_days`: You will have to set this based on your PKI policies, just remeber that this one dosen't impact the server and client certificates only the intermediate certificate authorities this is typically five to ten years in length. Also as you will likely be keeping your CA offline, in accordace with best practises, you will not want to use a short CRL validity period unless you require it.
      - `[ policy_loose ]`: You will wan't to set this up in accordance with your PKI policy; however, the provided configuration is good for most orginizations.
      - `[ req_dn ]`: Customize the defaults to your liking if you wish, just note that when we get to requests this section isn't actually used.
      - `[ crl_intermediate ] -> fullname`: You will want to put a little bit of thought behind this before you get to far ahead, that being said it doesn't really effect anything until you are ready to issue your first CRL. If you have a root CRL as well using the same URI with the differance being the crl that you are pointing to, that is perfectly acceptable.
      - `[ crl_issu_intermediate ]`: Same as above, Though you will want to ensure that this is set correctly otherwise your CRL checks will fail and it will look like everything is invalid
   2. Generate the intermediate key and signing request
      1. Grab a copy of the [csr.cnf [CA/common/1. Template/csr.cnf]](..blob/master/CA/root/openssl.cnf) and open it up in your favorite text editor  
         If you already have already done this up for your root certificate, go ahead and copy that instead
      2. Modify the following fields
         - `[ req ] -> default_bits`: Change this to 4096 or better, as this is the heart of your trust make it as strong as possible
         - `[ req ] -> encrypt_key`: Change this to yes, unless your pki policy states otherwise. In the event that this some how leaks you wan't to limit the possibility of Joe Blow being able to use this to generate his own certificate. You can also change this password at anytime in the future.
         - `[ req_dn ]`: Complete this section in whole. The `commonName` can be pretty much anything as long as its utf8. You may also want to include the `orginizationalUnitName` as a certficate descriptor.  
           **Note:** When completing this you need to check the signing authorties policies for signing certificates as it may require certain fields to match what is specified by the signing authority
         - `[ req_ext ] -> subjectKeyIdenifier`: As this is for the Root certificate which will be self signed, you will want to add this field - `subjectKeyIdentifier = hash`
         - `[ req_ext ] -> authorityKeyIdentifier`: If you have this field you will want to remove it
         - `[ req_ext ] -> basicConstraints`: Again you'll want to add this field - `basicConstraints = critical, CA:true`
           * Based on your PKI polices, you may want to include `pathlen:_x_` to restrict the number of intermediates this intermediate may sign. Using `pathlen:0` would be so that the intermediate cannot sign intermediate certificate, or more correctly stated, as its still technically possible to sign a intemediate with this specified, it cannot validate any certificates signed by a intermediate that it signed.
         - `[ req_ext ] -> keyUsage`: This is the last one to add - `keyUsage = critical, cRLSign, keyCertSign`
         - `[ req_ext ] -> extendedKeyUsage`: Comment out or remove this line, as it isn't needed
         - `[ alt_names ] -> DNS.1`: Comment out or delete this field as it isn't needed for this certificate
      3. Save that file under your intermediate CA folder as intermediate-ca-req.cnf
      4. run the following command while in your intermediate ca folder `openssl req -new -config intermediate-ca-req.cnf -newkey rsa -rand private/.rand -keyout private/intermediate.key -out intermediate.csr`.  
      To break this down, you are asking the OpenSSL **req** module to create a `-new` request using the `-config` located at the file you just saved, you want to generate a `-newkey` as specified either in the config or by a preceeding argument. You want to seed this certificate using the `-rand` file located under `private/.rand` as generated in section two. And finally you are going to output you key, `-keyout`, to the path as defined in your `openssl.cnf` and `-out`put the csr to the path specified.
      2. Navigate out of you intermediate directory back to the root of your PKI.  
         Run the following  
         ```openssl ca -config root/openssl.cnf -out intermediate/certs/intermediate.crt -extensions req_extv3_intermediate -infiles intermediate/intermediate-ca-req.cnf```  
         To break this down you are using the openssl **ca** module with the `-config` file of your root certificate authority to `-out`put the certificate to the location as specified in your intermedate certificate authority with the X.509 `-extensions` for the intermediate authority specification using the the certificate signing request, `-infiles`, that was out but by the request form the previous step.  
         A special note on this point, `-infiles` should be the last argument for the command in the event you want to run several csr through at the same time, alternativly you could use `-in` instead. If you screw up on this process you can reset the cert.db file and newcerts directory, though this is not recommended, instead you should regenerate your root certifiacate, so that incorrect intermediate if it ever gets out some how isn't valid. If you are in your test CA, and you screw up, its a good chance to try out revoking certificates.
      5. So now you have your signed certificate, with this you have two new files the private key along with the public certificate, which containts the public key. The first is the private key which is under `private/intermediate.key`, this is the piece that needs to be protected and never lost. The private key is the what you will use to sign certificates for users, server, etc. The other file, which you can grab a copy of, is `certs/intermediate.crt`, this is the public certificate, you will primarily use this as part of the certificate chain that you install on servers to ensure that you have a full certificate chain to fully validate the authenticity of the certficate. For client devices you don't need to do anything with this certificate as the public certificate for your root certificate authority will validate all server certificates.
      6. Go back and populate the `[ crl_issu_intermediate ]` section with the details of your newly minted intermediate Certificate Authority. You will also want to do the same for ` [ crl_intermediate ] -> fullname` if you haven't done so yet. Lastly you will want to ensure that all of the `[ req_extv3_* ] -> authorityInfoAcccess` fields to point to the OCSP server.
      7. Create the certificate chain that can be stored in the template folder under common. To do this create a new file, or concatinate the two file together to make a new file, and put first the intermediate certificate in followed by the root certificate.  
         The Chain order should be lowest to highest, meaning the first certificate shoud be the server certificate followed by the intermediate that signed it the the authority that signed it, etc, until you reach your root certificate. If you have any additional certificate, say for extended validation, they would be at the end of the file.
   2. Bonus: Active Directory Certificate Servces
      1. Installing the Active Directory Certificate services role
         1. Install the Certificate Authority and if needed the Certificate Authority Web Enrollment 
         2. You will then choose either a standalone or enterprise PKI. The differance being whether you will be integrating with active directory and using the auto enrollment policy, Enterprise PKI, or if the system will be non-integrated with active directory.
         3. Depending on how you want to set up this portion of the certificate authority you can either run this as the Root CA or the Subordinate CA, the latter of which we will be covering.
         4. You can either create a new private key or use one that you you generate similar to the intermediate certificate. We will cover creating a new key and based on testing, I don't recommend using an existing one, unless you are importing a previously signed key.
         5. Fill out the Cryptography page as per your PKI
         6. Fill in the Common name same as you did on the previous two certificates. In the Distinguished name suffix feild be sure to add the O (Orginization), C (Country), and ST (state or province name) distinguished names, along with any other DNs that you wish to include.
         7. You will want to "Save a certificate request to file and manually send it later to a partent CA:" and save it at a location and with a file name that you can grab later.
         8. Change the paths as you need or like
         9. Grab the certificate signing request and sign in with your CA.  
            As a note in order to sign the certificate, I had to changes the `[ policy_strict ]` section of the root certificate authority openssl config to permit non-matching fields, even though the fields do match...  
            <br>
            ```openssl ca -config root/openssl.cnf -extensions req_extv3_intermediate -days 3650 -infiles <path to the certificate signing request from the AD setup> -out root/certs/AD_intermediate.csr```
         10. Generate the CRL for your root certificate authority  
             ```openssl ca -config root/openssl.csr -gencrl -out root/crl/root.crl```  
             I will cover this in more detail in a later section, for the time being you will want grab the generated CRL and install it in your CRL webserver. This needs to be inplace before you can start active directory certificate services. If needed you could set this up on your AD CS server.
         11. Under Server Manager Roles, you should see the role for Active Directory cer tificate services, with one of the sub items being the Certificate authority, should be the last item. When you right click on this you will have the option to Install CA Certificate. Choose this and point it at the newly created certificate.
         12. Provided everything was done right and the server can reach the CRL distribution point you should see the service running that the name under Active Directory Certificate Services change to the name of the certificate.

# Creating and Managing Certificates
1. Generating the certificate signing request
   1. This can be done in a few ways and have a few different caveates depending on the method. The main ways are using the built-in request manager of a system, including using OpenSSLs request module without a configuration and generating 

