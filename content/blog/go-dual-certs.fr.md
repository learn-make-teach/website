+++
title = 'Go Dual Certs'
summary = "Support des certificats à la fois RSA et ECDSA sur un serveur Go pour une migration en douceur"
date = 2024-03-26T19:36:08+01:00
draft = false
tags = ['Go', 'TLS', 'Tips']
[params]
  image = 'gopher_shirt.png'
+++
Il y a plein de bonnes raison d'abandonner les certificats RSA au profit ECDSA, la principale étant la vitesse. Les clés étant plus petites, elles sont plus rapides à générer (ce qui peut etre un atout non négligeable quand on gère une grosse infra avec des certificats qui sont regénérés souvents), et les handshakes sont également plus rapide. L'inconvénient principal quand on parle d'un déploiement en DC est généralement la compatibilité client: les vieux clients risquent de ne plus pouvoir se connecter s'ils ne supportent pas les courbes elliptiques. Mais nous allons voir qu'il y a un moyen de conserver cette compatibilité en servant les 2 types de certificats en même temps.

## Pas besoin de changer de CA
Une croyance qui parfois freine l'adoption des certificats EC est qu'il faudrait changer toute la chaine de certification. Ca n'est pas du tout nécessaire, on peut signer un certificats ECDSA avec une CA en RSA, et nous allons d'ailleurs le faire en utilisant openssl. Nous créons la CA de la même façon que dans [l'article sur le mtls](/fr/blog/tls-demystified/#platforme-de-test). Puis nous générons une clé ECDSA (courbe la plus courante: prime256), une CSR qu'on signe avec notre CA intermédiaire (en RSA donc):
```
openssl ecparam -name prime256v1 -genkey -noout -out server-p256.key
openssl req -new -key server-p256.key -out server-p256.csr -subj '/CN=tls-server.local/O=LMT Corp' -addext 'subjectAltName=DNS:tls-server.local'
openssl x509 -req -in server-p256.csr -CA intermediateCA.crt -CAkey intermediateCA.key -CAcreateserial -out server-p256.crt -copy_extensions=copyall
```
Les commandes sont assez similaires à la version RSA, seule la génération de la clé privée diffère (on utilise `openssl ecparam` à la place de `openssl genrsa` ). La signature se passe bien, nous obtenons un certificats signé par la CA en RSA alors que la clé privée est en ECDSA.
## Implementation en Go
L'utilisation de 2 certificats en Go est assez triviale. Au moment de configurer la structure TLSConfig pour la passer au listener, il suffit de spécifier les 2 paires clés/certificats dans le champs _Certificates_, en prenant soin de mettre le certificat EC en premier:
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
C'est tout! Le serveur choisira le certificat en fonction de ce que le client déclare supporter dans le Client Hello, et si le client supporte à la fois ECDSA et RSA, le serveur choisira ECDSA.
## Test
On utilise `openssl s_client` pour tester:
```
openssl s_client -connect tls-server.local:8443  -CAfile rootCA.crt -showcerts
```
La sortie contient `Peer signature type: ECDSA` et le certificat présenté par le serveur est bien le certificat ECDSA.
Si on force l'utilisation de RSA :
```
openssl s_client -connect tls-server.local:8443 -sigalgs "RSA-PSS+SHA256" -CAfile rootCA.crt -showcerts
```
On obtient cette fois-ci `Peer signature type: RSA-PSS` ainsi que le certificat RSA.

On a donc un serveur qui gère les courbes elliptiques et qui repasse sur RSA au cas où 😎. Et le code est [ici](https://github.com/learn-make-teach/go-samples/blob/main/cmd/dualtls/main.go)!
