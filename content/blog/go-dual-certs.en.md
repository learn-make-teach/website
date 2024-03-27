+++
title = 'Go Dual Certs'
summary = "Support both RSA and ECDSA certificates on Go servers for a smooth migration"
date = 2024-03-26T19:36:08+01:00
draft = false
tags = ['Go', 'TLS', 'Tips']
[params]
  image = 'gopher_shirt.png'
+++
There are good reasons to move from RSA certificates to ECDSA, but the main one is speed. As the keys are smaller, they are faster to generate (usefull for short-lived server certs in big infrastructure), and the handshake is faster too. The main drawback for its deployment inside the DC is usually compatibility: old clients may not be able to connect if they don't support eliptic curves. But we will see that there is a way around that: dual certificate support!
## You can keep your CA
One belief that sometimes prevents the adoption of EC certificates is that the whole certificate chain has to be EC. That's obviously not true, an RSA CA can sign an ECDSA server. Let's put that to the test using openssl. We create an RSA CA (root + intermediate) following what has been explained in [this page](/en/blog/tls-demystified/#test-platform). Then we create the server ECDSA certificate (using prime256 curve, the most common curve today), the CSR and we sign it using the intermediate (RSA) CA:
```
openssl ecparam -name prime256v1 -genkey -noout -out server-p256.key
openssl req -new -key server-p256.key -out server-p256.csr -subj '/CN=tls-server.local/O=LMT Corp' -addext 'subjectAltName=DNS:tls-server.local'
openssl x509 -req -in server-p256.csr -CA intermediateCA.crt -CAkey intermediateCA.key -CAcreateserial -out server-p256.crt -copy_extensions=copyall
```
The commands are very similar to the RSA ones, only the private key generation differs (we use `openssl ecparam` instead of `openssl genrsa`). The signature goes according to plan, we have our signed certificate along with our ECDSA key.
## Go implementation
Using this certificate along with the other one in Go is trivial: when configuring the TLSConfig struct of the listener, we just have to specify both pairs in the _Certificates_ field, with the ECDSA first:
```
rsaCert, err := tls.LoadX509KeyPair("server.crt", "server.key")
if err != nil {
    panic(fmt.Errorf("can't load rsa key pair: %v", err))
}
ecCert, err := tls.LoadX509KeyPair("server-p256.crt", "server-p256.key")
if err != nil {
    panic(fmt.Errorf("can't load ecdsa key pair: %v", err))
}
config := tls.Config{
    Certificates: []tls.Certificate{ecCert, rsaCert},
}
```
That's all you need! The server will use the certificate according to the client's support for signature algorithms, and preferably the EC one is both are supported.
## Test
We use `openssl s_client` for our tests:
```
openssl s_client -connect tls-server.local:8443  -CAfile rootCA.crt -showcerts
```
The output contains `Peer signature type: ECDSA` and we can verify that the certificate sent by the server is indeed our ECDSA server.
we can also force using RSA :
```
openssl s_client -connect tls-server.local:8443 -sigalgs "RSA-PSS+SHA256" -CAfile rootCA.crt -showcerts
```
This time we get `Peer signature type: RSA-PSS` and the certificate is the RSA one.
We now have a server ready for eliptic curve with a fallback on RSA 😎. You can find the code [here](https://github.com/learn-make-teach/go-samples/blob/main/cmd/dualtls/main.go)!
