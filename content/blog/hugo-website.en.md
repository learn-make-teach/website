+++
title = 'Hugo Website'
date = 2024-03-25T10:48:05+01:00
summary = "How I use Hugo to generate this website and Github Action for continuous deployment"
tags = ['Hugo', 'Github Actions', 'Website']
draft = false
[params]
  image = 'hugo-logo-square.svg'
+++
When I decided to create this blog, I wondered which technology I was going to use. A few months back I had created [a landing page with Wordpress](https://agnes-coaching.fr) so I wanted to use something different, mainly as an opportunity to learn something new. I chose Hugo for it's nice separation of content and presentation which was close to how I usually design my softwares.

The purpose of this article is not to deep dive into Hugo or copy the [tutorial](https://gohugo.io/getting-started/) but to give you an overview of this tool and some of the specificities used to create this website.

The source code for this site is available on [Github](https://github.com/learn-make-teach/website).

## Hugo in a nutshell
[Hugo](https://gohugo.io/) is a static site generator. This means that there is a build phase which creates a website and once built this website is free from any dependency on Hugo, you get a bunch of html files which can be served by any webserver (nginx, apache...). You can work on the source files, push them in Git and have a CI build and publish the site. Sounds familiar 👌

To render the pages, Hugo uses templates. Each page is a content file (in markdown or html format) and is processed by a Go template to generate an html file. You can focus on the content (in the markdown files) knowing that all those pages will look exactly the same since the same template will be used to generate them. If later you want to change how the pages look, you just have to modify the template and everything is generated again at build time. There are predefined themes that you can switch using the main configuration file. Or you can build your own of course!

Hugo also support multilingual sites. Since I wanted this blog to be in french and in english, this feature was a must-have. You can write articles in different languages, each in its own file, and use dictionnaries to translate the text present in the templates.

In summary:
- <i class="bi bi-hand-thumbs-up-fill"></i>Static HTML generation - no dependency to manage on the webserver (no PHP/MySQL for a simple website)
- <i class="bi bi-hand-thumbs-up-fill"></i>Content/Rendering separation using templates
- <i class="bi bi-hand-thumbs-up-fill"></i>Only text files = easy to Git (compared to Wordpress which relies on a database)
- <i class="bi bi-hand-thumbs-up-fill"></i>Multilingual
{.pros}
- <i class="bi bi-hand-thumbs-down-fill"></i>Learning curve and some elbow grease before you have your first nice looking page (especially if you're not fluent in CSS!)
{.cons}

## Articles
As already mentioned, for each page of the site there is a markdown or html file in _content/_. All my articles are in a _blog/_ subdirectory, with french files ending with _.fr.md_ or _.fr.html_ and english files ending with _.en.md_ or _.en.html_ (you can put each language in its own folder if you prefer but I like seeing both versions of the files in the same place).

Each file starts with a section called _front matter_ which containing metadata like the title, summary, date and so on.

Templates are in the _layouts_ folder, either at the root of the project or in the _themes_ folder if you use themes. For this site the articles found in _content/blog/*.md_ will be rendered using the template in _layouts/blog/single.html_. Notice the parallel in the structure, that's a convention in Hugo (content in _blog_ -> template in _blog_). The template files use Go template notation (`{{ }}`) and Hugo provides utility functions/methods and shortcodes to manipulate some data (range, sort, filter, translate, ...). If you look into the _layouts/blog/single.html_ template for this file, you'll notice that it starts with `{{ define "main" }}`. This defines a block that can then be used by other templates to compose a full page. For example the _layouts/\_defaults/baseof.html_ template will add a header and a footer around this block:
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
This is how you define the main layout of your website. For example the homepage, my resume and the blog articles all share this header/main/footer layout but the main section is completely different.

## Résumé
The résumé is organized differently. There is still a file in _content/_ (mandatory entry point), but it only contains the front matter part and no content. The metadata will specify which template to use (_page/cv.html_) and contain a custom _format_ parameter indicating if it's the short or complete version of the résumé. The data itself comes from 4 different yaml files (en/fr and short/long) located in the _data/_ folder. The structure of these yaml files is always the same, the template just loads the correct yaml file and render the content using explicit loops (range) over some sections. For example to render the list of language I speak and the associated level, the render contains:
```
<h5>{{T "languages"}}</h5>
<ul class="list-unstyled">
    {{ range .skills.languages }}
        <li>{{ .name}}: {{ .level }}</li>
    {{ end }}
</ul>
```
while the yaml files will contain an array of { name, level } entries under the skills/languages key:
```
skills:
  languages:
    - level: native
      name: French
    - level: fluent
      name: English
```

Hugo provide the `{{T key}}` macro for internationalization. There is a yaml file per language in the _i18n/_ folder containing keys and translations, the macro will select the correct file depending on the current language.

## Deployment
I use Github Actions to build and deploy the website. Coming from Gitlab CI, it's my first time using Github CI. I'm using the free runners for this build. 

The build it pretty trivial : install Hugo, checkout the code, run Hugo and save the generated site (the _public/_ folder) as an artifact:
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
The website is hosted at [Hostinger](https://hostinger.com/) and they provide an SSH access. You can use rsync over SSH to copy the generated files. For that I use the _easingthemes/ssh-deploy_ action:
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
I download the artifact saved during the build job and send the files to the webserver. The authentication relies on a private key which is added as a secret in Github.

As soon as a commit is pushed on the _main_ branch, the website is build by the CI and deployed. Only the main branch is considered for the build, I can use other branches without the risk of breaking the website in production.

I hope you learned something, so far I have fun learning and using Hugo. You can find the complete source code for this website on [Github](https://github.com/learn-make-teach/website). I've only scratched the surface of what Hugo can do, don't hesitate to look at the [official Hugo website](https://gohugo.io/) for more information (even if I must confess the doc is not always helpful for newcomers, but the forum contains a lot of knowledge!).
