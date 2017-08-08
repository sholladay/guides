# Create a PKI chain

> Root CA => Intermediate CA => Server

## Root CA

Set up the directory structure for the root CA.

```sh
mkdir root-ca;
cd root-ca;
mkdir key cert crl newcert;
chmod 700 key;
touch index.txt;
echo 1000 > serial;
```

Make sure you have a config file for the root CA. For example [root-ca/openssl.conf](pki/root-ca/openssl.conf) or [root-config.txt](https://jamielinux.com/docs/openssl-certificate-authority/_downloads/root-config.txt).

```sh
nano openssl.conf;
```

Create a secret key for the root CA.

```sh
openssl genrsa -aes256 -out key/root-ca.key 4096;
```

Secure the secret key for the root CA.

```sh
chmod 400 key/root-ca.key;
```

Create a self-signed public certificate for the root CA.

```sh
openssl req -new -sha512 -x509 \
    -config openssl.conf \
    -days 3660 \
    -key key/root-ca.key \
    -out cert/root-ca.cert;
```

Secure the public certificate for the root CA.

```sh
chmod 444 cert/root-ca.cert;
```

**Optional:** Check the root CA's public certificate details.

```sh
openssl x509 -noout -text -in cert/root-ca.cert;
```

## Intermediate CA

**NOTE:** For many use cases, such as serving a simple web app, an intermediate CA is unnecessary. You can use the root CA to sign the server certificate directly instead. You will need to modify the relevant paths to point to the root CA.

Intermediate CAs are useful for HTTPS web proxies (such as [hoxy](https://github.com/greim/hoxy)), because they need to be able to sign certificates at runtime, but you don't want to put the root CA's secret key in your project's repository. So you create an intermediate CA that can do the same thing. This limits your security risk to that individual project rather than everywhere the root CA is used.

Set up the directory structure for the intermediate CA.

```sh
mkdir ../server-ca;
cd ../server-ca;
mkdir key csr cert crl newcert;
chmod 700 key;
touch index.txt;
echo 1000 > serial;
echo 1000 > crlnumber;
```

Create a config file for the intermediate CA. For example [server-ca/openssl.conf](pki/server-ca/openssl.conf) or [intermediate-config.txt](https://jamielinux.com/docs/openssl-certificate-authority/_downloads/intermediate-config.txt).

```sh
nano openssl.conf;
```

Create a secret key for the intermediate CA.

```sh
openssl genrsa -aes256 -out key/server-ca.key 4096;
```

Secure the secret key for the intermediate CA.

```sh
chmod 400 key/server-ca.key;
```

Create a temporary CSR file that will help the root CA create a public certificate for the intermediate CA.

```sh
openssl req -config openssl.conf -new -sha512 \
    -key key/server-ca.key \
    -out csr/server-ca.csr;
```

```sh
cd ../root-ca;
```

Create the intermediate CA's public certificate, based on its CSR.

```sh
openssl ca -notext \
    -config openssl.conf \
    -extensions v3_intermediate_ca \
    -days 2565 \
    -in ../server-ca/csr/server-ca.csr \
    -out ../server-ca/cert/server-ca.cert;
```

Secure the public certificate for the intermediate CA.

```sh
chmod 444 ../server-ca/cert/server-ca.cert;
```

Check the intermediate CA's public certificate details.

```sh
cd ../server-ca;
```

```sh
openssl x509 -noout -text -in cert/server-ca.cert;
```

Verify the integrity of the intermediate CA's public certificate.

```sh
openssl verify -CAfile ../root-ca/cert/root-ca.cert \
    cert/server-ca.cert;
```

Create the intermediate CA's public certificate chain file. You usually won't need this, but it is good practice in case this CA ever needs to create another CA. In that case, you need to maintain the certificate chain because the client needs to either have the public certificates of every intermediate CA or it needs to receive a chain file with them from the server.

```sh
cat cert/server-ca.cert \
    ../root-ca/cert/root-ca.cert > cert/server-ca-chain.cert;
chmod 444 cert/server-ca-chain.cert;
```

## Server

Set up the directory structure for the server.

```sh
mkdir ../server;
cd ../server;
mkdir key csr cert;
chmod 700 key;
```

Create the server's secret key. You can add a password with the `-aes256` flag, but you will need to enter the password whenever the server restarts.

```sh
openssl genrsa -out key/localhost.key 4096;
```

Secure the secret key for the server.

```sh
chmod 400 key/localhost.key;
```

Create a temporary CSR file that will help the intermediate CA create a public certificate for the server.

```sh
openssl req -new -sha512 \
    -config ../server-ca/openssl.conf \
    -key key/localhost.key \
    -out csr/localhost.csr;
```

Create the server's public certificate, based on its CSR.

```sh
cd ../server-ca;
```

```sh
openssl ca -notext \
    -config openssl.conf \
    -extensions server_cert \
    -days 1105 \
    -in ../server/csr/localhost.csr \
    -out ../server/cert/localhost.cert;
```

Secure the public certificate for the server.

```sh
cd ../server && chmod 444 cert/localhost.cert;
```

Check the server's public certificate details.

```sh
openssl x509 -noout -text -in cert/localhost.cert;
```

Verify the integrity of the server's public certificate.

```sh
openssl verify -CAfile ../server-ca/cert/server-ca-chain.cert \
    cert/localhost.cert;
```

Create the server's public certificate chain file. This file allows clients to verify the integrity of the server's public certificate without themselves knowing anything about the intermediate CA(s). This step is unnecessary if you did not create an intermediate CA. The root CA's public certificate should **never** end up in this file.

```sh
cat cert/localhost.cert \
    ../server-ca/cert/server-ca.cert > cert/localhost-chain.cert;
chmod 444 cert/localhost-chain.cert;
```
