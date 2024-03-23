+++
title = 'TLS handshake démistifié'
date = 2024-02-29T19:40:22+01:00
summary = 'Comprendre comment fonctionne le handshake mTLS et pourquoi il peut échouer'
draft = false
[params]
  image = 'spy-vs-spy.webp'
+++
## Intro
TLS (pour Transport Layer Security) est le successeur du protocol SSL qui est considéré non sécurisé. Le but est d'apporter de la confidentialité (chiffrement) et d'éviter leur falsification (signature) des données et des identités. Ca utilise du chiffrement asymétrique basé sur des pairs de clés/certificats pour échanger des clés de sessions symétriques. Aucun échange de secret (clés privées) n'est requis au préalable.

Dans sa forme mutuelle (mTLS), on a aussi l'autentification de l'identité du client grace à une seconde paire de clé/certificat.

## Lab
Pour illustrer comment TLS fonctionne et où ça peut échouer, nous allons mettre en place un serveur et un client grâce à OpenSSL. Ce couteau suisse est l'unique outil nécessaire pour générer et vérifier des certificats, debugger une connection, et bien plus. OpenSSL est généralement déjà installé sur linux, il existe également des ports pour les autres OS (mais sous Windows je recommende d'utiliser WSL pour accéder à l'implémentation linux).

### Quelques certificats et un peu de confiance

TLS est basé sur du chiffrement asymétrique, on utilise donc une clé privée (généralement appelé _la clé_) et une clé publique (_le certificat_). La clé privée est nécessaire pour déchiffrer et signer, alors qu'il suffit de la clé publique pour chiffrer et vérifier la signature. Dans un échange standard, le client va donc chiffrer la donnée avec la clé publique, le serveur va la déchiffrer avec la clé privée. Le serveur va signer la donnée avec la clé privée, le client va vérifier que la signature est correcte.

En tant que client, pour pouvoir établir une telle conversation avec un serveur, il va d'abord falloir faire confiance à sa clé publique. Pour que ça soit à peu près gérable, on ne fait pas directement confiance à chaque clé de chaque serveur individuellement. Pour faire un parallèle avec la vie courante, on ne connait pas individuellement chaque personne avec qui on échange mais on va faire confiance à la société qui lui a délivrer un badge ou au gouvernement qui lui a fournit une pièce d'identité. Pour nos serveurs, on va faire confiance à des Autorités de Certifications (AC ou CA en anglais), des entités qui sont habilités à délivrer des certificats.

La clé publique de notre serveur va donc être signée par une CA, et cette signature est incluse dans le certificat que le serveur présente. Le client pourra donc vérifier cette signature en utilisant la clé publique de la CA, et donc en faisant juste confiance à la CA, le client pourra indirectement faire confiance aux serveurs. La clé publique de la CA sera généralement signée par une autre CA et ainsi de suite jusqu'à atteindre la CA racine (celle tout au bout de la chaine), qui sera autosignée (signée par elle même). Il faudra donc à minima faire confiance à cette CA racine pour pouvoir vérifier n'importe quel certificat de la chaine. Les OS ont un magasin contenant toutes les CA racines de confiance, et ce magasin est régulièrement mis à jour et permet d'avoir une solide base de référence. Sur Ubuntu par exemple, vous pouvez trouver ce magasin dans `/etc/ssl/certs`.

Faisons un test avec Google:
```
openssl s_client -connect google.com:443 <<< "Q"
```
Cette commande va établir une connexion TLS avec google.com sur le port 443. Le `<<< "Q"` à la fin permet juste de fermer la connexion dès qu'elle est établie (pour éviter d'avoir un prompt et de devoir `Ctrl+C` pour en sortir). Si on regarde le résultat de cette commande, on trouve entre autre:
```
Certificate chain
 0 s:CN = *.google.com
   i:C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
 1 s:C = US, O = Google Trust Services LLC, CN = GTS CA 1C3
   i:C = US, O = Google Trust Services LLC, CN = GTS Root R1
 2 s:C = US, O = Google Trust Services LLC, CN = GTS Root R1
   i:C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
```
Ici on peut voir la chaine de certificats, la racine étant _GlobalSign Root CA_, puis 2 CA intermédiaires ( _GTS Root R1_ et _GTS CA 1C3_), puis enfin le certificat serveur (_*.google.com_). On peut vérifier que cette CA racine est bien dans notre magasin de confiance via la commande `openssl x509 -text -noout -in /etc/ssl/certs/GlobalSign_Root_CA.pem`. On obtient alors tout le détail de cette CA, ce qui permettra à nos navigateurs et nos outils de pouvoir se connecter à Google et vérifier la chaine de certificat. Parmis les infos de cette CA, on peut voir que l'émetteur (_Issuer_) est le même que le sujet (_Subject_), ce qui définit un certificat autosigné.
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            04:00:00:00:00:01:15:4b:5a:c3:94
        Signature Algorithm: sha1WithRSAEncryption
        Issuer: C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
        Validity
            Not Before: Sep  1 12:00:00 1998 GMT
            Not After : Jan 28 12:00:00 2028 GMT
        Subject: C = BE, O = GlobalSign nv-sa, OU = Root CA, CN = GlobalSign Root CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:da:0e:e6:99:8d:ce:a3:e3:4f:8a:7e:fb:f1:8b:
                    83:25:6b:ea:48:1f:f1:2a:b0:b9:95:11:04:bd:f0:
                    63:d1:e2:67:66:cf:1c:dd:cf:1b:48:2b:ee:8d:89:
                    8e:9a:af:29:80:65:ab:e9:c7:2d:12:cb:ab:1c:4c:
                    70:07:a1:3d:0a:30:cd:15:8d:4f:f8:dd:d4:8c:50:
                    15:1c:ef:50:ee:c4:2e:f7:fc:e9:52:f2:91:7d:e0:
                    6d:d5:35:30:8e:5e:43:73:f2:41:e9:d5:6a:e3:b2:
                    89:3a:56:39:38:6f:06:3c:88:69:5b:2a:4d:c5:a7:
                    54:b8:6c:89:cc:9b:f9:3c:ca:e5:fd:89:f5:12:3c:
                    92:78:96:d6:dc:74:6e:93:44:61:d1:8d:c7:46:b2:
                    75:0e:86:e8:19:8a:d5:6d:6c:d5:78:16:95:a2:e9:
                    c8:0a:38:eb:f2:24:13:4f:73:54:93:13:85:3a:1b:
                    bc:1e:34:b5:8b:05:8c:b9:77:8b:b1:db:1f:20:91:
                    ab:09:53:6e:90:ce:7b:37:74:b9:70:47:91:22:51:
                    63:16:79:ae:b1:ae:41:26:08:c8:19:2b:d1:46:aa:
                    48:d6:64:2a:d7:83:34:ff:2c:2a:c1:6c:19:43:4a:
                    07:85:e7:d3:7c:f6:21:68:ef:ea:f2:52:9f:7f:93:
                    90:cf
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                60:7B:66:1A:45:0D:97:CA:89:50:2F:7D:04:CD:34:A8:FF:FC:FD:4B
    Signature Algorithm: sha1WithRSAEncryption
         d6:73:e7:7c:4f:76:d0:8d:bf:ec:ba:a2:be:34:c5:28:32:b5:
         7c:fc:6c:9c:2c:2b:bd:09:9e:53:bf:6b:5e:aa:11:48:b6:e5:
         08:a3:b3:ca:3d:61:4d:d3:46:09:b3:3e:c3:a0:e3:63:55:1b:
         f2:ba:ef:ad:39:e1:43:b9:38:a3:e6:2f:8a:26:3b:ef:a0:50:
         56:f9:c6:0a:fd:38:cd:c4:0b:70:51:94:97:98:04:df:c3:5f:
         94:d5:15:c9:14:41:9c:c4:5d:75:64:15:0d:ff:55:30:ec:86:
         8f:ff:0d:ef:2c:b9:63:46:f6:aa:fc:df:bc:69:fd:2e:12:48:
         64:9a:e0:95:f0:a6:ef:29:8f:01:b1:15:b5:0c:1d:a5:fe:69:
         2c:69:24:78:1e:b3:a7:1c:71:62:ee:ca:c8:97:ac:17:5d:8a:
         c2:f8:47:86:6e:2a:c4:56:31:95:d0:67:89:85:2b:f9:6c:a6:
         5d:46:9d:0c:aa:82:e4:99:51:dd:70:b7:db:56:3d:61:e4:6a:
         e1:5c:d6:f6:fe:3d:de:41:cc:07:ae:63:52:bf:53:53:f4:2b:
         e9:c7:fd:b6:f7:82:5f:85:d2:41:18:db:81:b3:04:1c:c5:1f:
         a4:80:6f:15:20:c9:de:0c:88:0a:1d:d6:66:55:e2:fc:48:c9:
         29:26:69:e0
```
A noter que certaines plateforme comme le JRE de Java utilise un magasin spécifique et par défaut pas celui du système (_/etc/ssl/certs/java/cacerts_ sur Ubuntu).

### Platforme de test

Pour nos tests, nous allos avoir besion d'un certificat serveur. Pour être proche de la réalité, nous allons créer une chaine de 3 certificats, 2 CA et un certificat serveur. Nous pourrions utiliser une CA connue (comme _Let's encrypt_ qui fournit gratuitement des certificats) mais nous aurions besoin de prouver que le nom de domaine que nous voudrions utiliser nous appartient bien en réussissant un challenge (mise à disposition d'un fichier à la racine du domaine qu'on veut signer par exemple). Pour rester simple, nous allons générer notre propre CA racine, la difference sera donc que cette CA ne sera pas par défaut dans le magasin de confiance de l'OS, il faudra donc soit l'y rajouter, soit explicitement indiquer qu'on lui fait confiance dans nos outils.

Créons d'abord la clé privée de la CA racine. Nous allons générer une clé RSA de longueur 4096 bits, ce qui est assez standard. Cette clé sera stockée au format PEM (ascii) dans un fichier _rootCA.key_.
```
openssl genrsa -out rootCA.key 4096
```
Créons ensuite le certificat associé (certificat autosigné: -x509 -new), non chiffré (-nodes), en utilisant notre clé privée (-key rootCA.key), valide pendant un an (-days 365) et avec un sujet correct:
```
openssl req -out rootCA.crt -x509 -new -nodes -key rootCA.key -days 365 -subj '/CN=rootCA/O=LMT Corp'
```
Créons maintenant la clé privée de la CA intermédiaire de la même façon que nous avons créer la racine:
```
openssl genrsa -out intermediateCA.key 4096
```
Pour la signer, nous devons générer un CSR (demande de signature de certificat):
```
openssl req -new -key intermediateCA.key -out intermediateCA.csr -subj '/CN=intermediateCA/O=LMT Corp' -addext 'basicConstraints = critical,CA:true'
```
Puis nous la signons en utilisant la clé privée de la CA racine:
```
openssl x509 -req -in intermediateCA.csr -CA rootCA.crt -CAkey rootCA.key  -CAcreateserial -out intermediateCA.crt -copy_extensions=copyall
```
Enfin, même chose pour la clé privée du serveur:
```
openssl genrsa -out server.key 4096
```
CSR (avec un SAN, obligatoire maintenant):
```
openssl req -new -key server.key -out server.csr -subj '/CN=tls-server.local/O=LMT Corp' -addext 'subjectAltName=DNS:tls-server.local'
```
et on le signe avec la clé privée de la CA intermédiaire:
```
openssl x509 -req -in server.csr -CA intermediateCA.crt -CAkey intermediateCA.key -CAcreateserial -out server.crt  -copy_extensions=copyall
```
Coté serveur ça devrait être tout bon, créons dans la foulée une CA et un certificat client. C'est la même chose que pour le serveur à part l'extension du certificat, nous devons préciser qu'il s'agira d'un certificat client:
```
openssl genrsa -out clientCA.key 4096
openssl req -out clientCA.crt -x509 -new -nodes -key clientCA.key -days 365 -subj '/CN=clientCA/O=LMT Corp'
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr -subj '/CN=client' -addext 'extendedKeyUsage = clientAuth'
openssl x509 -req -in client.csr -CA clientCA.crt -CAkey clientCA.key -CAcreateserial -out client.crt  -copy_extensions=copyall
```
Enfin fini! Nous pouvons maintenant lancer un serveur et tester une connexion. Nous utilisons `openssl s_server` pour créer un serveur, et `curl` comme client:
```
openssl s_server -key server.key -cert server.crt -accept 8443 -www -cert_chain chain.crt -build_chain -CAfile clientCA.crt -Verify 10 -verify_return_error -tls1_3 -strict -ciphersuites TLS_CHACHA20_POLY1305_S
HA256
```
Cette commande crée un serveur écoutant sur le port 8443, on utilise la paire clé/certificat de notre serveur et on spécifie notre CA clientCA pour l'autentification du client. On rend le certificat client obligatoire (-Verify, -verify_return_error). Ce serveur acceptera uniquement de connexions en TLS 1.3 utilisant le cipher (algorithme) TLS_CHACHA20_POLY1305_S. Le fichier _chain.crt_ contient la concatenation des certificats publiques de la CA racine (rootCA.crt) et intermédiaire (intermediateCA.crt) afin de créer une chaine de confiance complete. Ce serveur enverra un code HTTP en réponse avec un résumé des paramètres de la connexion si tout se passe bien.
Pour le tester, nous devons d'abord ajouter cette ligne dans le fichier _/etc/hosts_:
```
127.0.0.1 tls-server.local
```
Ceci permet d'utiliser le nome de domaine tls-server.local, le SAN que nous avons déclaré lors de la création de notre certificat serveur. 
Il est maintenant temps de faire notre premier appel client, en spécifiant notre paire de clé/certificat client et en indiquant qu'on fait confiance à la CA racine de notre serveur:
```
curl https://tls-server.local:8443 --cacert ./rootCA.crt --key ./client.key --cert ./client.crt
```
Ce qui devrait vous retourner un status HTTP 200 et tout un tas d'infos passionnantes.
Maintenant que nous avons prouvé que ça marche, on peut commencer à échouer. Pour cela nous devons comprendre les grands principes du handshake TLS.
# TLS handshake![Handshake mTLS simplifié](/img/handshake.png)
Je ne vais pas rentrer dans les détails technique du protocole TLS qui est vraiment très riche (et parfois complexe) mais me concentrer sur les échanges qui peuvent poser problème entre le client et le serveur.
Avant le handshake TLS, il y a tout d'abord un handshake TCP, ce qui veut dire que si vous ne résolvez par l'hote (problème DNS), n'avez pas de route (problème de gateway/proxy), ou si vous avez un timeout (problème de firewall), il faudra d'abord vous attarder sur votre configuration réseau avant de regarder coté TLS.

## Client Hello
Le client commence par envoyer la liste de versions TLS et cipher qu'il supporte. Le server vérifie qu'il supporte au moins une version et un cipher dans cette list. Il sélectionne la version la plus récente et le cipher le plus sécurisé. Si le serveur ne trouve pas de version commune ou de cipher commun, il envoie une erreur et ferme la connexion.
Par exemple si on essaye d'utiliser TLS 1.2 coté client alors que notre serveur ne supporte que TLS 1.3:
```
curl https://tls-server.local:8443 --cacert ./rootCA.crt --key ./client.key --cert ./client.crt --tls-max 1.2 -v
```
Nous recevons une erreur:
```
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Unknown (21):
* TLSv1.2 (IN), TLS alert, protocol version (582):
curl: (35) error:0A00042E:SSL routines::tlsv1 alert protocol version
```
De la même manière, avec un cipher non supporté par notre serveur:
```
curl https://tls-server.local:8443 --cacert ./rootCA.crt --key ./client.key --cert ./client.crt --tls13-ciphers TLS_AES_256_GCM_SHA384 -v
```
Nous recevons une erreur plsu générique:
```
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Unknown (21):
* TLSv1.3 (IN), TLS alert, handshake failure (552):
curl: (35) error:0A000410:SSL routines::sslv3 alert handshake failure
```
Si le client et le serveur ont une version et cipher en commun, le serveur envoie à son tour son Hello.
## Server Hello, certificates, acceptable CAs
Le serveur envoie la version TLS et le cipher sélectionnés, suivi de:
- sa chaine de certificat
- dans le cas de mTLS une demande de certificat client accompagnée de la liste des CA autorisées

Le client va vérifier:
- que la chaine de certificats présentée par le serveur est valide: chaque certificat est signé par le certificat suivant dans la chaine, aucun n'est expiré, et qu'il fait confiance à un des certificats (généralement la CA racine)
- que le certificat présenté par le serveur correspond bien au nom de domaine qu'il essaye de joindre (tls-server.local dans notre exemple)
Vu que ces vérifications sont faites par le client, elles peuvent être ignorées ( mode non sécurisé ou "trust all").
Par exemple si retire la CA racine de notre liste de CA de confiance:
```
curl https://tls-server.local:8443 -v
```
Le client échoue avec:
```
* TLSv1.2 (OUT), TLS header, Unknown (21):
* TLSv1.3 (OUT), TLS alert, unknown CA (560):
* SSL certificate problem: self-signed certificate in certificate chain
```
Si on appelle un nom de domaine mais que le serveur présente un certificat pour DN différent:
```
curl https://127.0.0.1:8443 --cacert ./rootCA.crt -v
```
```
*  subjectAltName does not match 127.0.0.1
* SSL: no alternative certificate subject name matches target host name '127.0.0.1'
```
Le client va par défaut vérifier que le nom de domaine est bien dans le SAN (Subject Alternative Names) du certificat du serveur. Comme nous avons juste mis subjectAltName=DNS:tls-server.local lors de la création du certificat, celui-ci n'est pas valide pour 127.0.0.1.
Nous pouvons ignorer ces vérifications en utilisant le paramètre -k, mais il est toujours préférable d'avoir un handshake valide. Ces vérifications nous permettent en effet de vérifier que le serveur qui nous répond est bien celui qui doit nous répondre et pas quelqu'un d'autre (attaque de l'homme du milieu ou man in the middle, quelqu'un intercepte le traffic et se fait passer pour le serveur afin de récupérer vos données, cet individu n'aura pas un certificat valide).

## Certificat client
L'étape d'après pour notre client est l'envoi de son certificat en accord avec ce que le serveur demande. Le serveur devrait envoyer une liste de CA qu'il considère valide, on peut le voir grâce à `openssl s_client`:
```
openssl s_client -connect tls-server.local:8443
```
La réponse va contenir entre autre:
```
Acceptable client certificate CA names
CN = clientCA, O = LMT Corp
```
Pour satisfaire le serveur, on va donc lui envoyer notre certificat signé par cette CA, ainsi qu'une signature qui va prouvée qu'on possède bien la clé privée associée à ce certificat.
Si le client ne possède pas de certificat correspondant à la demande (parce qu'il n'a pas de certificat signé par cette CA ou que celui qu'il a est expiré), alors le serveur fermera la connexion avec une erreur.
Pas de certificat:
```
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS alert, unknown (628):
* OpenSSL SSL_read: error:0A00045C:SSL routines::tlsv13 alert certificate required, errno 0
```
Certificat signé par une CA inconnue:
```
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS alert, unknown CA (560):
* OpenSSL SSL_read: error:0A000418:SSL routines::tlsv1 alert unknown ca, errno 0
```
A noter que certains client (comme Java), échoueront s'ils ne trouvent pas de certificats valide dans leur magasin alors que d'autres clients peuvent envoyer un certificat même en sachant qu'il ne correspond pas (ex: curl), voire pas de certificat du tout alors que le serveur en demande un.

Si vous recevez un code HTTP de la part du serveur, c'est que le handshake a réussi. Félicitations!

# TLDR - Pourquoi ça marche pas
| Erreur | Cause fréquente |
| ----- | ------------ |
| Le client est déconnecté juste après le Client Hello | Le serveur n'a pas trouvé de version TLS ou cipher compatible avec ce que le client supporte. Vérifiez les versions des outils/librairies coté client et serveur | 
| Le client ferme la connexion juste après le Server Hello, en se plaignant d'un CA inconnue ou autosignée | Le client ne fait pas confiance à la CA racine du serveur. Soit elle n'est pas dans son magasin de confiance (celui de l'OS, celui du JRE, en ligne de commande), ou ce n'est pas la bonne. Vérifiez la chaine de certificat du serveur avec openssl et comparez avec le magasin utilisé |
| Le client ferme la connexion juste après le Server Hello, se plaignant que le nom du serveur ne correspond pas | Le nom de domaine que vous appelez ne correspond pas au SAN déclaré dans le certificat du serveur. Ceci peut venir d'un problème de configuration du serveur (reverse proxy en coupure TLS), ou vous avez besoin d'adapter votre configuration réseau via /etc/hosts (peu courant dans un setup de production). Vérifez ce qui est déclaré dans le certificat serveur en utilisant `openssl s_client -showcerts` pour le récupérer, puis `openssl x509` pour le parser. Le nom de domaine doit se trouver dans le SAN du certificat. |
| Dans un setup mTLS, le client abandonne la connexion juste après le Server Hello, indiquant qu'il n'a pas de certificat à présenter | Le client est strict (ex: java) et ne trouve pas de certificat client valide correspondant à ce que le serveur demande, ou le serveur ne demande rien. Vérifiez ce que le serveur demande (partie acceptable CAs, via `openssl s_client`), et comparer avec votre certificat client. Vérifiez aussi le magasin utilisé (keystore pour la clé privée, truststore pour le certificat). Certaines technologies de client nécessitent d'avoir la chaine CA cliente dans un magasin de confiance (truststore) en plus de la CA serveur. |
| Le serveur ferme la connexion en demandant un certificat client | Le serveur est configuré pour faire du mTLS et vous n'avez pas envoyé de certificat client. Vérifiez que vous avez un certificat valide signé par la CA attendue par le serveur et que vous avez la clé privée correspondante dans vos magasins (truststore & keystore) |
| Le serveur ferme la connexion en se plaignant que la CA est inconnue | Le certificat client envoyé n'est pas signé pas une CA que le serveur reconnait. Vérifiez la CA attendue par le serveur (acceptable CAs) et celle qui a signé le certificat client (`openssl x509` pour parser votre certificat client, la CA est appelée Issuer) |


