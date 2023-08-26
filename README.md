Based on Millie Smith's answer @ https://superuser.com/questions/126121/how-to-create-my-own-certificate-chain

## Command Summary

```bash
openssl genrsa -des3 -out root.key 2048
openssl req -new -key root.key -out root.csr -config root_req.config
openssl ca -in root.csr -out root.pem -config root.config -selfsign -extfile ca.ext -days 1095

openssl genrsa -des3 -out intermediate.key 2048
openssl req -new -key intermediate.key -out intermediate.csr -config intermediate_req.config
openssl ca -in intermediate.csr -out intermediate.pem -config root.config -extfile ca.ext -days 730

openssl genrsa -out leaf.key 2048
openssl req -new -key leaf.key -out leaf.csr -config leaf_req.config
openssl ca -in leaf.csr -out leaf.pem -config intermediate.config -days 365

openssl verify -x509_strict -CAfile root.pem -untrusted intermediate.pem leaf.pem
```

These commands rely on some setup which I will describe below. They are a bit of an overkill if you just want a few certs in a chain, which can be done with just the x509 command. These commands will also track your certs in a text database and auto-increment a serial number. I would recommend reading the warnings and bugs section of the openssl ca man page before or after reading this answer.

## Directory structure

We will need the following directory structure before starting.

```
ca.ext              # the extensions required for a CA certificate for signing certs
intermediate.config # configuration for the intermediate CA
root.config         # configuration for the root CA

leaf_req.config         # configuration for the leaf cert's csr
intermediate_req.config # configuration for the intermediate CA's csr
root_req.config         # configuration for the root CA's csr

intermediate_ca/    # state files specific to the intermediate CA
    index           # a text database of issued certificates
    serial          # an auto-incrementing serial number for issued certificates
root_ca/            # state files specific to the root CA
    index           # a text database of issued certificates
    serial          # an auto-incrementing serial number for issued certificates
```

If this is a more permanent CA, the following changes are probably a good idea:

- Moving each CA's configuration file, private key (generated later), and certificate file (generated later) to the CA's directory. This will require changes to the configuration file.
- Creating a subdirectory in the CA's directory for issued certificates. This requires changes to the configuration file
- Encrypting the private key
- Setting a default number of days for issued certificates in the CA configuration files

## Detailed commands

```bash
# create the private key for the root CA
openssl genrsa 
    -des3         # password protect the key
    -out root.key # output file
    2048          # bitcount

# create the csr for the root CA
openssl req 
    -new 
    -key root.key           # private key associated with the csr
    -out root.csr           # output file
    -config root_req.config # contains config for generating the csr such as the distinguished name

# create the root CA cert
openssl ca 
    -in root.csr        # csr file
    -out root.pem       # output certificate file
    -config root.config # CA configuration file
    -selfsign           # create a self-signed certificate
    -extfile ca.ext     # extensions that must be present for CAs that sign certificates
    -days 1095          # 3 years

# create the private key for the intermediate CA
openssl genrsa 
    -des3                 # password protect the key
    -out intermediate.key # output file
    2048                  # bitcount

# create the csr for the intermediate CA
openssl req 
    -new 
    -key intermediate.key           # private key associated with the csr
    -out intermediate.csr           # output file
    -config intermediate_req.config # contains config for generating the csr such as the distinguished name

# create the intermediate CA cert
openssl ca 
    -in intermediate.csr  # csr file
    -out intermediate.pem # output certificate file
    -config root.config   # CA configuration file (note: root is still issuing)
    -extfile ca.ext       # extensions that must be present for CAs that sign certificates
    -days 730             # 2 years

# create the private key for the leaf certificate
openssl genrsa 
    -out leaf.key # output file
    2048          # bitcount

# create the csr for the leaf certificate
openssl req 
    -new 
    -key leaf.key           # private key associated with the csr
    -out leaf.csr           # output file
    -config leaf_req.config # contains config for generating the csr such as the distinguished name

# create the leaf certificate (note: no ca.ext. this certificate is not a CA)
openssl ca 
    -in leaf.csr                # csr file
    -out leaf.pem               # output certificate file
    -config intermediate.config # CA configuration file (note: intermediate is issuing)
    -days 365                   # 1 year

# verify the certificate chain
openssl verify 
    -x509_strict                # strict adherence to rules
    -CAfile root.pem            # root certificate
    -untrusted intermediate.pem # file with all intermediates
    leaf.pem                    # leaf certificate to verify
```

