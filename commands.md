# Command Details

These commands rely on some setup which I will describe below. They are a bit of an overkill if you just want a few certs in a chain, which can be done with just the x509 command. These commands will also track your certs in a text database and auto-increment a serial number. I would recommend reading the warnings and bugs section of the openssl ca man page before or after reading this answer.

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
    -extfile leaf.ext           # extensions for leaf, for example alt-names
    -days 365                   # 1 year

# verify the certificate chain
openssl verify 
    -x509_strict                # strict adherence to rules
    -CAfile root.pem            # root certificate
    -untrusted intermediate.pem # file with all intermediates
    leaf.pem                    # leaf certificate to verify

# Concatenate the different certificates into a single full chain file
# ORDER IS HERE IMPORTANT !! Leaf DC must be on top !
cat leaf.pem intermediate.pem root.pem > fullchain_leaf.pem
```

## Steve's internet guide

Basic key generation with only root CA and no config.

http://www.steves-internet-guide.com/mosquitto-tls/

```bash
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 1826 -key ca.key -out ca.crt
openssl genrsa -out server.key 2048
openssl req -new -out server.csr -key server.key
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 360
cat ca.crt server.crt > fullchain.crt
```
