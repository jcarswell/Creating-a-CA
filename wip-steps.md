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

OCSP -

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
4. Build the OpenSSL config file (root edition)
