To start this "Project" is intended to help guide you with creating a private Certificate Authority and operating the Certifcate Authority once it is up and running. This is not intended as a replacement for the documentation provided as part of OpenSSL, I strongly advise looking at the man pages at https://www.openssl.org/docs/manpages.html or using `man ca` and `man req` once you are comfortable with concepts of certificate signing reviewing `man x509` and `x509v3_config` will bring you up to date with current certificate standards. Another great resource is Ivan Ristić book (OpenSSL Cookbook)[https://www.feistyduck.com/books/openssl-cookbook/].

This is currently a work in progress, you can find the openssl CA config file in the root directory openssl.cnf as well as the basic templates to get you started under the CA directory. Below you will find some of the common commands that I use day to day and the basic process of generating the certificates.

## Certificate Signing Requests (CSR)
### Create a CSR with a new Private Key
```shell
openssl reg -new -config [configfile] -keyout [privatekey] -out [csrFile] (-nodes)
```

### Create a CSR with an existing Private Key
```shell
openssl reg -new -config [configfile] -key [privatekey] -out [csrFile] (-nodes)
```
## Certificate Management
### Create a Certificate from a CSR
```shell
openssl ca -config [caConfig] -extensions [x509 extension]-out [signedCert] -infiles [csrFile]
```

### Create a PFX archive with the signed cert
```shell
openssl pkcs12 -export -out [pfxArchive] -inkey [privateKey] -in [signedCert] -certfile [caChain]
```

### Revoke Signed Certificates
```shell
openssl ca -config [caConfig] -revoke [signedCert]
```

### Update the Certificate Revocation List
```shell
openssl ca -config [caConfig] -gencrl -out [crlFile]
```


# Creating the Certificate

1. Generate the CSR
 1. Copy the template [1.Template] in the common directory to the name of the site
 2. Modify the .cnf file and fill in the details
 3. use _OpenSSL_ to generate the signing request
2. Sign the cert
 1. Use _OpenSSL_ to sign the cert from the intermediate configuration
 2. Use _OpenSSL_ to create the PFX archive
 3. combine using tar for ease of transport `tar -czf [certname].tgz [certfile] [[certfile] ...]`
3. revoke the old
 1. once the new SSL cert has been installed use _OpenSSL_ to revoke the old


# Certificate Authority Directory structure

| Directory/File                    | Purpose                                    |
|:----------------------------------|:-------------------------------------------|
| `/`                                  |                                            |
| `/common`                            | Certificate store                          |
| `/common/1.Template`                 | Certificate template to copy               |
| `/common/1.Template/csr.cnf`         | CSR config template                        |
| `/common/1.Template/ca-chain.csr`    | CA Certificate chain                       |
| `/intermediate`                      | Signing CA                                 |
| `/intermediate/openssl.cnf`          | Signing CA config                          |
| `/intermediate/crl/intermediate.crl` | CRL for the intermediate certificate       |
