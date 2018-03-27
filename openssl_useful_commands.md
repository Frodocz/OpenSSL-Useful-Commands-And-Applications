# Openssl commands

## 1. General OpenSSL Commands
> These commands allow you to generate CSRs, Certificates, Private Keys and do other miscellaneous tasks.

* Generate a new private key and Certificate Signing Request
```openssl
openssl req -out CSR.csr -new -newkey rsa:2048 -nodes -keyout privateKey.key
```
* Generate a self-signed certificate
```openssl
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout privateKey.key -out certificate.crt
```
* Generate a certificate signing request (CSR) for an existing private key
```openssl
openssl req -out CSR.csr -key privateKey.key -new
```
* Generate a certificate signing request based on an existing certificate
```openssl
openssl x509 -x509toreq -in certificate.crt -out CSR.csr -signkey privateKey.key
```
* Remove a passphrase from a private key
```openssl
openssl rsa -in privateKey.pem -out newPrivateKey.pem
```

## 2. Checking Using OpenSSL
> If you need to check the information within a Certificate, CSR or Private Key, use these commands. 

* Check a Certificate Signing Request (CSR)
```openssl
openssl req -text -noout -verify -in CSR.csr
```
* Check a private key
```openssl
openssl rsa -in privateKey.key -check
```
* Check a certificate
```openssl
openssl x509 -in certificate.crt -text -noout
```
* Check a PKCS#12 file (.pfx or .p12)
```openssl
openssl pkcs12 -info -in keyStore.p12
```

## 3. Converting Using OpenSSL
> These commands allow you to convert certificates and keys to different formats to make them compatible with specific types of servers or software. 

* Convert a DER file (.crt .cer .der) to PEM
```openssl
openssl x509 -inform der -in certificate.cer -out certificate.pem
```
* Convert a PEM file to DER
```openssl
openssl x509 -outform der -in certificate.pem -out certificate.der
```
* Convert a PKCS#12 file (.pfx .p12) containing a private key and certificates to PEM
```openssl
openssl pkcs12 -in keyStore.pfx -out keyStore.pem -nodes
```
[comment]: # (You can add -nocerts to only output the private key or add -nokeys to only output the certificates.)
* Convert a PEM certificate file and a private key to PKCS#12 (.pfx .p12)
```openssl
openssl pkcs12 -export -out certificate.pfx -inkey privateKey.key -in certificate.crt -certfile CACert.crt
```
