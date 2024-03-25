+++
title = 'Dans les coulisses de ce site '
date = 2024-03-25T10:47:57+01:00
summary = "Comment j'utilise Hugo pour générer ce site et Github Actions pour le déployer"
tags = ['Hugo', 'Github Actions', 'Website']
draft = false
[params]
  image = 'hugo-logo-wide.svg'
+++
Quand j'ai décidé de créer ce blog, je me suis demandé comment j'allais le faire. J'avais déjà réalisé [un site vitrine avec Wordpress](https://agnes-coaching.fr) donc j'avais envie d'apprendre autre chose. J'ai choisi Hugo parce que la séparation contenu/présentation colle bien avec mes habitudes de développement de logiciel.
## Présentation de Hugo
[Hugo](https://gohugo.io/) est un générateur de site statique. Concrétement ça signifie qu'il y a une phase de build et que le résultat de ce build est votre site. Donc au runtime il n'y a aucune dépendance sur Hugo, le site peut être servi par n'importe quel serveur web (nginx, apache... ). On va donc pouvoir travailler sur les sources, les gérer dans git, builder le site et le publier où on veut via la CI! Workflow qui parait familier :)

Pour le rendu du site, Hugo utilise des templates. Chaque page est un fichier dans content (format markdown ou html), qui va être mixé avec un template pour la mise en forme et qui va produire une page html. On peut donc écrire facile ses articles en md sans se soucier du rendu, et ils auront tous la même apparence puisqu'un même template sera utilisé pour tous les articles. Et par la suite, si on veut changer l'apparence du site, on modifie le template et tout reste cohérent. Il existe des themes prédéfinis qu'on peut échanger assez facilement via une ligne dans un fichier de configuration.

Enfin, Hugo gère le multilingue, et comme je voulais ce blog en anglais et en français, c'était un critère important. On va pouvoir écrire les articles dans différentes langues, et également utiliser des dictionnaires pour traduire les templates.

En résumé:
- <i class="bi bi-hand-thumbs-up-fill"></i>Génère du HTML statique - pas de dépendance à maintenir sur le webserver
- <i class="bi bi-hand-thumbs-up-fill"></i>Séparation contenu/rendu avec système de templating
- <i class="bi bi-hand-thumbs-up-fill"></i>Tout est fichier texte donc facile à gérer avec Git (vs Wordpress où il y a une DB MySQL)
- <i class="bi bi-hand-thumbs-up-fill"></i>Multilingue
{.pros}
- <i class="bi bi-hand-thumbs-down-fill"></i>Demande un peu d'efforts pour avoir un site custom (CSS n'est pas mon ami)
{.cons}

## Articles
Comme dit plus tôt, chaque page du site correspond d'abord à un fichier markdown ou html dans _content/_. Je mettrais tous les articles dans un sous-dossier _blog/_, les fichers en français se terminent par .fr.md ou .fr.html, ceux en anglais par .en.md ou .en.html (on peut aussi créer un sous dossier par langue mais je préfère voir les articles dans les deux langues l'un à coté de l'autre).

Chaque fichier comporte deux sections, une première partie appelée _front matter_ qui sont les métadonnées de la page (titre, date de création, résumé...), et le contenu en lui même.

On trouve le template qui sera appliqué à ce contenu dans le dossier layouts, ou dans themes si on utilise un thème prédéfini ou qu'on crée le sien. Pour ma part j'ai tout fait de zero donc pour mes articles dans _content/blog_, le template sera _layouts/blog/single.html_. Les fichiers template utilise le templating de Go (`{{ }}`), et des fonctions prédéfinies de Hugo. On notera qu'en fait ce template défini un bloc (il commence par `{{ define "main" }}`), ce bloc est utilisé par le template de base _layouts/\_defaults/baseof.html_ :
```
<!DOCTYPE html>
<html lang="{{ .Site.Language }}">
    {{- partial "head.html" . -}}
    <body>
        <div class="container-xxl shadow-sm p-0">
            {{- partial "header.html" . -}}
            {{- block "main" . }}{{- end }}
            {{- partial "footer.html" . -}}
        </div>
    </body>
</html>
```
Ce découpage en partial/block permet de découper et réutiliser les templates, par exemple la page d'accueil ou mon CV ne seront pas présentés comme les articles de blog mais partageront la navigation et le pied de page par exemple.

## CV
Le CV est construit différement. Il y a toujours un fichier dans _content/_, c'est un point d'entrée obligatoire, mais il ne contient la partie _front matter_ et pas de contenu. Ce _front matter_ permet de sélectionner un template spécifique pour le rendu, _page/cv.html_. Les données proviennent de fichiers yaml contenus dans _data/_. En utilisant la même structure pour tous les fichiers yaml (même clés _title, intro, experience, skills, education_...), j'ai pu créer 4 fichiers de données différent (cv court et détaillé, en français et en anglais), et le template charge juste le bon fichier en fonction de la langue de la page et d'un paramètre dans le _front_matter_ du fichier content. Il suffit ensuite d'utiliser le moteur de templating de Go pour sélectionner ou itérer sur les différentes listes du fichier yaml, comme par exemple pour afficher les langues et le niveau de maitrise:
```
<h5>{{T "languages"}}</h5>
<ul class="list-unstyled">
    {{ range .skills.languages }}
        <li>{{ .name}}: {{ .level }}</li>
    {{ end }}
</ul>
```
On utilise la macro `{{T key}}` pour les traductions. Il y a deux fichiers _en.yaml_ et _fr.yaml_ dans _i18n/_ qui servent de dictionnaires en fonction de la langue de la page.
## Déploiement
J'utilise Github Actions pour lancer le build du site puis le déployer. Le build est assez trivial, il suffit d'installer Hugo, de récupérer le code, de lancer Hugo et d'archiver le résultat (_public/_):
```
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.123.7
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb          
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Build with Hugo
        env:
          # For maximum backward compatibility with Hugo modules
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "https://learn-make-teach.com"
      - name: Upload static site as artifact
        uses: actions/upload-artifact@v4
        with:
          name: public
          path: ./public
```

Le site est hébergé chez Hostinger qui autorise l'accès par SSH. Il est donc facile d'utiliser rsync over ssh pour pousser les fichiers générés:
```
  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: public
          path: ./public
      - name: Deploy to Hostinger
        uses: easingthemes/ssh-deploy@v5.0.3
        with:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          ARGS: "-avz -i --delete"
          SOURCE: "public/"
          REMOTE_HOST: ${{ secrets.REMOTE_HOST }}
          REMOTE_PORT : ${{ secrets.REMOTE_PORT }}
          REMOTE_USER: ${{ secrets.REMOTE_USER }}
          TARGET: ${{ secrets.REMOTE_TARGET }}
```
Je récupère les fichiers préalablement archivés, puis les envoie vers le serveur web. Il suffit d'échanger une clé SSH pour autoriser l'accès.

Ainsi, dès que des modifications sont poussés sur la branche _main_, le site est automatiquement déployé. Il n'y a pas de CI sur les autres branches, ce qui permet d'avoir des versions de travail sans impacter le site live.

Vous pouvez retrouver tout le code source de ce site sur [github](https://github.com/learn-make-teach/website). Hugo permet de faire bien d'autres choses que je vous invite à découvrir directement sur le [site officiel](https://gohugo.io/).