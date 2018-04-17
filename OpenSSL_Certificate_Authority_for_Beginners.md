# OpenSSL Certificate Authority for Beginners

> This guide aims to provide beginners with knowledges on how to use OpenSSL to generate their own certificate authority (CA), which can be used to secure website connections between clients and server.

## Table of Contents

* [Introduction]()
* [Generate Root Key Pair]()
    - [Prepare the directory]()
    - [Prepare the configuration file]()
    - [Generate the root key]()
    - [Generate the root certificate]()
    - [erify the root certificate]()
* [Generate Intermediate Key Pair]()
    - [Prepare the directory]()
    - [Generate the intermediate key]()
    - [Generate the intermediate certificate]()
    - [Verify the intermediate certificate]()
    - [Generate the certificate chain file]()
* [Sign Server and Client Certificates]()
    - [Generate a key]()
    - [Generate a certificate]()
    - [Verify the certificate]()
    - [Deploy the certificate]()
* [Certificate Revocation Lists (CRL)]()
    - [Prepare the configuration file]()
    - [Create the CRL]()
    - [Revoke a certificate]()
    - [Server-side usage of the CRL]()
    - [Client-side usage of the CRL]()
* [Online Certificate Status Protocol (OCSP)]()
    - [Prepare the configuration file]()
    - [Generate the OCSP pair]()
    - [Revoke a certificate]()
* [Appendix]()
    - [Root CA configuration file]()
    - [Intermediate CA configuration file]()

## Section I: Introduction

OpenSSL is a free and open-source cryptographic library that provides several command-line tools for handling digital certificates. Some of these tools can be used to act as a certificate authority.

A certificate authority (CA) is an entity that signs digital certificates. Many websites need to let their customers know that the connection is secure, so they pay an internationally trusted CA (eg, VeriSign, DigiCert) to sign a certificate for their domain.

In some cases it may make more sense to act as your own CA, rather than paying a CA like DigiCert. Common cases include securing an intranet website, or for issuing certificates to clients to allow them to authenticate to a server (eg, Apache, OpenVPN).

## Section II: Generate Root Key Pair

Acting as a certificate authority CA) means dealing with cryptographic pairs of private keys and public certificates. The very first cryptographic pair we’ll create is the root pair. This consists of the root key (`ca.key.pem`) and root certificate (`ca.cert.pem`). This pair forms the identity of your CA.

Typically, the root CA does not sign server or client certificates directly. The root CA is only ever used to create one or more intermediate CAs, which are trusted by the root CA to sign certificates on their behalf. This is best practice. It allows the root key to be kept offline and unused as much as possible, as any compromise of the root key is disastrous.

> **Note**
> 
> It’s best practice to create the root pair in a secure environment. Ideally, this should be on a fully encrypted, air gapped computer that is permanently isolated from the Internet. Remove the wireless card and fill the ethernet port with glue.

### 2.1 Prepare the directory

Choose a directory (`/root/ca`) to store all keys and certificates.
```shell
mkdir /root/ca
```

Create the directory structure. The `index.txt` and `serial` files act as a flat file database to keep track of signed certificates.
```shell
cd /root/ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
```

### 2.2 Prepare the configuration file
You must create a configuration file for OpenSSL to use. Copy the root CA configuration file from the *[Appendix]()* [to `/root/ca/openssl.cnf`.

The `[ ca ]` section is mandatory. Here we tell OpenSSL to use the options from the `[ CA_default ]` section.
```shell
[ ca ]
# `man ca`
default_ca = CA_default
```

```shell
[ CA_default ]
# Directory and file locations.
dir               = /root/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict
```

We’ll apply `policy_strict` for all root CA signatures, as the root CA is only being used to create intermediate CAs.
```shell
[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

We’ll apply `policy_loose` for all intermediate CA signatures, as the intermediate CA is signing server and client certificates that may come from a variety of third-parties.
```shell
[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

Options from the `[ req ]` section are applied when creating certificates or certificate signing requests.
```shell
[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca
```

The `[ req_distinguished_name ]` section declares the information normally required in a certificate signing request. You can optionally specify some defaults.
```shell
[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Optionally, specify some defaults.
countryName_default             = SG
stateOrProvinceName_default     = Singapore
localityName_default            = Singapore
0.organizationName_default      = Vkey
#organizationalUnitName_default =
#emailAddress_default           =
```

The next few sections are extensions that can be applied when signing certificates. For example, passing the `-extensions v3_ca` command-line argument will apply the options set in `[ v3_ca ]`.

We’ll apply the v3_ca extension when we *[Generate the root certificate]()*.
```shell
[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

We’ll apply the v3_ca_intermediate extension when we *[Generate the intermediate certificate]()*. `pathlen:0` ensures that there can be no further certificate authorities below the intermediate CA.
```shell
[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
```

We’ll apply the `usr_cert` extension when signing client certificates, such as those used for remote user authentication.
```shell
[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection
```

We’ll apply the `server_cert` extension when signing server certificates, such as those used for web servers.
```shell
[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```

The `crl_ext` extension is automatically applied when creating *[certificate revocation lists]()*.
```shell
[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always
```

We’ll apply the `ocsp` extension when signing the *[Online Certificate Status Protocol (OCSP)]()* certificate.
```shell
[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

### 2.3 Generate the root key
Create the root key `(ca.key.pem)` and keep it absolutely secure. Anyone in possession of the root key can issue trusted certificates. Encrypt the root key with AES 256-bit encryption and a strong password.

> **Note**
> 
> Use 4096 bits for all root and intermediate certificate authority keys. You’ll still be able to sign server and client certificates of a shorter length.

```shell
cd /root/ca
# openssl genrsa -aes256 -out private/ca.key.pem 4096
openssl genrsa -out private/ca.key.pem 4096
# Enter pass phrase for ca.key.pem: secretpassword
# Verifying - Enter pass phrase for ca.key.pem: secretpassword

chmod 400 private/ca.key.pem
```

### 2.4 Generate the root certificate

Use the root key (`ca.key.pem`) to create a root certificate (`ca.cert.pem`). Give the root certificate a long expiry date, such as twenty years. Once the root certificate expires, all certificates signed by the CA become invalid.

> **Warning**
> Whenever you use the `req` tool, you must specify a configuration file to use with the `-config` option, otherwise OpenSSL will default to `/etc/pki/tls/openssl.cnf`.

```shell
cd /root/ca
openssl req -config openssl.cnf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -sha256 -extensions v3_ca \
      -out certs/ca.cert.pem

# Enter pass phrase for ca.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.
-----
Country Name (2 letter code) [XX]:SG
State or Province Name []:Singpaore
Locality Name []:Singpaore
Organization Name []:Vkey
Organizational Unit Name []:vos
Common Name []:vosCA
Email Address []:vos@v-key.com

chmod 444 certs/ca.cert.pem
```

### 2.5 Verify the root certificate
```shell
openssl x509 -noout -text -in certs/ca.cert.pem
```

The output shows:
* the `Signature Algorithm` used
* the dates of certificate `Validity`
* the `Public-Key` bit length
* the `Issuer`, which is the entity that signed the certificate
* the `Subject`, which refers to the certificate itself

The `Issuer` and `Subject` are identical as the certificate is self-signed. Note that all root certificates are self-signed.

The output also shows the **X509v3 extensions**. We applied the `v3_ca` extension, so the options from `[ v3_ca ]` should be reflected in the output.

## Section III: Generate the intermediate pair

An intermediate certificate authority (CA) is an entity that can sign certificates on behalf of the root CA. The root CA signs the intermediate certificate, forming a chain of trust.

The purpose of using an intermediate CA is primarily for security. The root key can be kept offline and used as infrequently as possible. If the intermediate key is compromised, the root CA can revoke the intermediate certificate and create a new intermediate cryptographic pair.

### 3.1 Prepare the directory

The root CA files are kept in `/root/ca`. Choose a different directory (`/root/ca/intermediate`) to store the intermediate CA files.
```shell
mkdir /root/ca/intermediate
```

Create the same directory structure used for the root CA files. It’s convenient to also create a `csr` directory to hold certificate signing requests.
```shell
cd /root/ca/intermediate
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
```

