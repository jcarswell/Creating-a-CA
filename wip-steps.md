#Notes on security:

Your CA is only as secure as you make it, if you keep you certificate files on a encrypted USB stick and store it in a vault that only designated people have access to, your probably going down the right path. On the flip side if you are running this on a internet exposed server that hasn't been patched in years and has a password of 'Passw0rd' then your definetly doing it wrong and need to re-evaluate your security practices before implementing a trust system.

#Definitions
Yes I know eveyones favorite thing, don't worry there will be a spelling test at the end :)

Primary Key Infrastructure (PKI) - 

Certificate Authority (CA)   

Certificate

PEM -

Key -

Certificate Revokation List (CRL) -

Online Certificate Status Protocol (OCSP) - This enables application to determin the validity of a certificate by querying a server

OCSP Stapaling - This isn't really a term that you need to know to manage a CA but it is something that I will cover briefly here. OCSP Stapling is the process of keeping a local (webserver) copy of the CA's OCSP validation for your certificate reducing the number of additional calls a browser needs to make to validate that the certificate is valid. 

Trust - This is a general term that refers to the certificate authority, but more specificatly the trustworthyness, i think i just made that up, of the authority.

#Building up the CA
Okay time for the nittiy gritty, I will try not to make this boring while including as much information to help you build a proper CA with the right amount of dangerousness to wreak havoc. I do strongly encourage that you start by building a test CA that you can issues some certificates from to test the configuration and ensure that you have a good grasp on what the fridgesicles is going on as some change are near impossible to make on your producation CA without re-issuing all of your internediate CA's or worse, doh.

1. Build the directory structure
  - There should be 3 directories, one for your root authority, one for your intermediate, and one as a common directory for certificate signing
  - Your directories can be either simple names or logincal names, this is your preferance, a simple name would be like 'root' or a logical name would be symbolic of the certificates name for example, my certificates name would be 'Silly CA EC1' therefor my logical name would be 'EC1'
  - 3 directories is just a starting point, if you need 17 intermediate certificates then do that instead, ultiamtely you and your predicessors need to be able to manage the CA in a manner that suites the goals of implementing at certificate authority (CA) or primary key infrastructure (pki)
2. Initalize the certificate folders
  - You will need to create several folders and files withing each of the certificate folders. These folders and files are certs, crl, private and the files are serial and openssl.cnf, more on this in a moment. To simplify this process you can run <insert script here> and specify the folder(s) which you want to initialize.
    * Firstly, as you will see in a bit, none of the folder/file names are fixed they can be configured in the openssl config file
    * openssl.cnf: this is the default name for an openssl config file and makes it easier for someone down the road to determine what the F is going on, provided that they have some background in managing CA's the improper way, using openssl utility. This can be changed to simplify the workflow and can be stored in completely different directory, the caveaut being that the user managing the CA will still have to be able to access both this file and the certificate folder(s). You could also use a single files to manage all of the certificates
    * serial: this is an important one and it's use may not be imediatly obvious, this is the certificate serial number, stored in hex, this will be the field that you need to know in order to revolk certificates that you don't have the original cerificate file from. so when you initalize this value ensure that you are using a good serial number that should remain uniqe.
  - Initalize the serial number, yes I know im going to repeat myself. Most of the time when you are reading articals on setting up a CA they recommend that you initalize this file with 10 00, now if you are just building a test CA this is perfectly OK; however, if this will be your production CA, DO NOT use this, use some this that is uniqe to your orginization, like a street address, founding year or even the CA/Company Name, coverted to hex, though it might get a little long, and use that as your prefix followed by enough length that the serial will stay uniqe, so 10 10 00 00 00 00 where 10 10 is my prefix. to create the file simply do `echo [your serial number] > serial` assuming that you are in the certificate folder
  - Build your openssl config. Just kidding, I can't do this here there is way to much detilal to do it right, so go to jail, do not pass go and do not collect 200 dollor's.
4. The OpenSSL Configuration
 1. The OpenSSL config file and some background.
  - Lets start with this, fisrt grab the openssl.cnf example from [here \[CA/intermediate/openssl.cnf\]](..blob/master/CA/intermediate/openssl.cnf). Open the file up in you favorite text editor and lets take a deep dive into the configuration.
  - File sections, the openssl configuration files uses the ini configuration standard, see [Wikipedia INI file](https://en.wikipedia.org/wiki/INI_file) for more detailed informationon the informal standard. Most of the section names are flexible as they are referanced at various points in the configuration files as are not directly referanced by openssl. Sections that are drectly referanced by OpenSSL are all of the standard commands, ca, crl, req, ocsp, etc. That's not to say that you couldn't use those name wouldn't be true however using them would either cause OpenSSL to outright fail to do anything or make unexpected changes to the action you are performing. This in mind this is countered by only having an effect when you actually call in the configuration file against the specific module, so to make things simple only use the sections as they are intended.
  
  Continuing on this point, manual pages (man) are your friend and the openssl documentation within is very extensive starting at OpenSSL as the main library documentation, `man openssl`, and then each of the OpenSSL modules, ca, ec, req, ts, etc. to look up the individual doncuments simply address the module directly from man, `man ca`, `man req`, `man x509`, etc.
  - Module sections, as previously stated, these are the sections that openssl checks when you call the module with the config flag. These are sections are normally pretty simplistic and tell OpenSSL that certin things are required or simply where to look during to get the defaults for the module. Examples of this are the `[ ca ]` section which tells the OpenSSL ca module which CA specification to look at, more on this later. Or the `[ req ]` section which tells the OpenSSL req(uest) module specification that are needed to perform the actions and ensure that specific feilds are present before creating the request and passing on the request for signing. These sections however aren't necissarly required as most of the option can be set via flags or simply using the OpenSSL defaults as is. Using the defaults does have some potential drawbacks when you are building a CA for a orginization or even as a public CA due to the ever evolving standards and security improvments that need to be tweeked to keep up with the changes that certain orginization are enforcing.
  - CA Specifications: If you are going to use OpenSSL to manage the CA, these specs are _really_ important. You will need at minimum two of these, unless you are getting a publicly signed CA certificate, one for your root certificate and one for you intermediate certificate, or more simply put one strict setting and one loose setting.
  - Certificate Specifcations: These are the opposite of the CA specifications, and you will likely have many more of these, incuding one for each CA specifications as these are certificates as well. We'll go deeper into these later on.
  - The rest of the sections are to support the other sections where an option requires it own set of configurations.
 2. The OpenSSL config file (root edition) - If you are planning on getting a signed CA certifiacte from the likes of Comodo, DigiCert, etc skip to the end.
  - To start, grab the OpenSSL configuration file from [\[CA/root/openssl.cnf\]](..blob/master/CA/root/openssl.cnf). Once you have this saved open it up and we will go over it piece by piece, because I know you were excited for some light bed time reading.
   1. Section `[ ca ]`
    - You still here? Good. This one is pretty easy, it starts with some basic info on this section as it is one on of the OpenSSL module section, I will skip over the comments for now as we will look at these in a later section. This has a few configuration options but we really only care about one, default_ca, there is also preserve and msie_hack. I'm not going to go into too much deatil on the specific configuration as that is all very well documented in the manual pages.
    - default_ca: Short and sweat, this tells the OpenSSL CA module which CA specification that you want to use.
   2. Section `[ root ]`
    - I think this could be a little more obvious... this is the first CA specification, and only one in this file. This is kinda an important section this basically tells the CA module what to do to sign a certificate, what requirments the request must meet in order to be signed, what needs to be part of the signed certificate and where to store the signed files
    - Storage: the first 11 items are all related to the files that are needed in order to store files, correctly sign certificate and ensure that if you have to revoke a certificate it happens properly. This section, like most is highly customizable depending on how you plan on setting up and configuring you certificate authority. The settings laied out in the confiurationt files are based on the directory prep that was done previously.
    - `dir`: this isn't actually a configuration item it's a variable. based on the provided template this is saying that when you are using this CA spec you will be in the directory above, the the provided template this is the 'CA' folder, and that the files for this CA spec will be in the 'root' directory. If this is you first CA that you are setting up, just copy the template down and leave this as is, when you are building the production CA make sure that this is setup correctly depending on how you have laied out your directory structure. The good new is you can move your 'root' folder where ever you want can call it what ever you want at any given point in time as long as you can access the files and this path is correct.
    - `certs`: [Remove Me] This is a variable to the certificate storage location
    - `crl_dir`: [Remove Me] This is a variable to the crl storage
    - `new_certs_dir`: This is the local storage for all signed certificates, while the files are the same as the output certificate these are not intended for the same usage. The certificates used here are so that when you revoke a certificate by serial number OpenSSL (the CA) can referance the original certificate whitout having it on hand. This will also be used by the CA when you generate your OCSP and CRL records to determin what is valid and what isn't valid
    - `database`: As the name implies, this is the file that is responsible for all of the certificates that have ever been handled bu the CA. Thats to say that it is used in conjunction with `new_certs_dir` to manage the CA.
    - `serial`: This is the serial number which you prepopulated in the previous section this file is a counter to indicated the next unique serial number in when signing a request.
    - `RANDFILE`: This one requires a bit of background on how cryptography work. For the time being just know that this is an important piece to building a strong trust in conjunction with the private key.
    - `private_key`: We will get into this later, but needless to say this points at your private key
    - `certificate`: This here is the (self) signed certificate slash your public key or atleast this tells OpenSSL where to look to find it.
    - `crlnumber`: This is similar the serial file with the exception that it is the "serial number" for your CRL.
    - `crl`: This is the location that the CRL is stored
