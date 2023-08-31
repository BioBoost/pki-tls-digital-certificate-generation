Based on Millie Smith's answer @ https://superuser.com/questions/126121/how-to-create-my-own-certificate-chain
Also some extra thanks too Andrew Connell for his gist: https://gist.github.com/andrewconnell/340ba2eecbc35540b83026340abaf03d

## Root and Intermediate CA commands

```bash
mkdir -p root_ca/db && echo "00" > root_ca/db/serial && touch root_ca/db/index

openssl genrsa -des3 -out root_ca/key/root.key 2048
openssl req -new -key root_ca/key/root.key -out root_ca/requests/root.csr -config root_ca/configs/root_req.config
openssl ca -in root_ca/requests/root.csr -out root_ca/certificates/root.pem -config root_ca/configs/root.config -selfsign -extfile ca.ext -days 1095

mkdir -p intermediate_ca/db && echo "00" > intermediate_ca/db/serial && touch intermediate_ca/db/index

openssl genrsa -des3 -out intermediate_ca/key/intermediate.key 2048
openssl req -new -key intermediate_ca/key/intermediate.key -out root_ca/requests/intermediate.csr -config intermediate_ca/configs/intermediate_req.config
openssl ca -in root_ca/requests/intermediate.csr -out root_ca/certificates/intermediate.pem -config root_ca/configs/root.config -extfile ca.ext -days 730
```

Some more info about the commands can be found at [command details](./commands.md).

## Leaf Certificate

Best to use the bash script:

```bash
./new_leaf
```

Or use these if you like the manual work (make sure to change the filenames then)

```bash
openssl genrsa -out leafs/leaf.key 2048
openssl req -new -key leafs/leaf.key -out intermediate_ca/requests/leaf.csr -config leaf_req.config
openssl ca -in intermediate_ca/requests/leaf.csr -out leafs/leaf.pem -config /intermediate_ca/configs/intermediate.config -extfile leaf_base.ext -days 365

cat leafs/leaf.pem root_ca/certificates/intermediate.pem root_ca/certificates/root.pem > leafs/fullchain_leaf.pem

openssl verify -x509_strict -CAfile root_ca/certificates/root.pem -untrusted root_ca/certificates/intermediate.pem leafs/leaf.pem
```
