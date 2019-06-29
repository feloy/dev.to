
I will explain in this article how to create from scratch an internationalized (i18n) Angular app with the use of the Angular CLI and how to deploy it on an Apache or NGINX web server.

The following versions are used:
- angular-cli: 8.0.4
- angular: 8.0.4
- Apache 2.4
- NGINX 1.17

The described sample app is available at: https://github.com/feloy/angular-cli-i18n-sample

## A fresh i18n app

We first create a fresh Angular app with the help of the Angular CLI:

```
$ ng new angular-cli-i18n-sample
```

We make some changes to add some translatable text, in 
`app.component.html`:

```
<h1 i18n>Hello world!</h1>
```

We need now to create an xlf file with the translatable strings. We can generate the file `src/i18n/messages.xlf` with the following command:

```
$ ng xi18n --output-path src/i18n
```

We now create translations for different languages, here in english with a fresh file `src/i18n/messages.en.xlf` copied from `src/i18n/messages.xlf`:

```
[...]
      <trans-unit id="[...]" datatype="html">
        <source>Hello World!</source>
        <target>Hello World!</target>
        [...]
      </trans-unit>
[...]
```

in french with `src/i18n/messages.fr.xlf`:

```
[...]
      <trans-unit id="[...]" datatype="html">
        <source>Hello World!</source>
        <target>Salut la foule !</target>
        [...]
      </trans-unit>
[...]
```

and in spanish with `src/i18n/messages.es.xlf`:

```
[...]
      <trans-unit id="[...]" datatype="html">
        <source>Hello World!</source>
        <target>¿hola, qué tal?</target>
        [...]
      </trans-unit>
[...]
```

It is now possible to make Angular CLI build the app with the language of your choice, here in spanish:

```
$ ng build --aot \
           --i18n-file=src/i18n/messages.es.xlf \
           --i18n-locale=es \
           --i18n-format=xlf
```

## Prepare the app for production

In production, we would like the app to be accessible in different subdirectories, depending on the language; for example the spanish version would be accessible at http://myapp.com/es/ and the french one at http://myapp.com/fr/. We also would like to be redirected from the base url http://myapp.com/ to the url of our preferred language.

For this, we guess that we need to change the base href to **es**, **en** or **fr**, depending on the target language. Angular CLI has a special command-line option for this, `--base-href` which permits to declare the base href at compile time from command line.

### Linux/macOS users

Here is the shell command we can use to create the different bundles for the different languages:

```
$ for lang in es en fr; do \
    ng build --output-path=dist/$lang \
             --aot \
             --prod \
             --base-href /$lang/ \
             --i18n-file=src/i18n/messages.$lang.xlf \
             --i18n-format=xlf \
             --i18n-locale=$lang; \
  done
```

We can create a script definition in package.json for this command and execute it with npm run build-i18n:

```
{
  [...]
  "scripts": {
    [...]
    "build-i18n": "for lang in en es fr; do ng build --output-path=dist/$lang --aot --prod --base-href /$lang/ --i18n-file=src/i18n/messages.$lang.xlf --i18n-format=xlf --i18n-locale=$lang; done"
  }
  [...]
}
```

At this point we get three directories `en/`, `es/` and `fr/` into the `dist/` directory, containing the different bundles.

### Windows users

As a Windows user, you can use these commands to build your different bundles for different languages:

```
> ng build --output-path=dist/fr --aot --prod --base-href /fr/ --i18n-file=src/i18n/messages.fr.xlf --i18n-format=xlf --i18n-locale=fr

> ng build --output-path=dist/es --aot --prod --base-href /es/ --i18n-file=src/i18n/messages.es.xlf --i18n-format=xlf --i18n-locale=es

> ng build --output-path=dist/en --aot --prod --base-href /en/ --i18n-file=src/i18n/messages.en.xlf --i18n-format=xlf --i18n-locale=en
```

We can create script definitions in `package.json` for these commands and a supplementary one to run all these commands at once and execute the last one with `npm run build-i18n`:

```
"scripts": {
    "build-i18n:fr": "ng build --output-path=dist/fr --aot --prod --base-href /fr/ --i18n-file=src/i18n/messages.fr.xlf --i18n-format=xlf --i18n-locale=fr",
    "build-i18n:es": "ng build --output-path=dist/es --aot --prod --base-href /es/ --i18n-file=src/i18n/messages.es.xlf --i18n-format=xlf --i18n-locale=es",
    "build-i18n:en": "ng build --output-path=dist/en --aot --prod --base-href /en/ --i18n-file=src/i18n/messages.en.xlf --i18n-format=xlf --i18n-locale=en",
    "build-i18n": "npm run build-i18n:en && npm run build-i18n:es && npm run build-i18n:fr"
  }
```

## Apache2 configuration

Here is a virtual host configuration which will serve your different bundles from the `/var/www` directory: you will have to copy in this directory the three directories `en/`, `es/` and `fr/` previously generated.

With this configuration, the url http://www.myapp.com is redirected to the subdirectory of the preferred language defined in your browser configuration (or `en` if your preferred language is not found) and you still have access to the other languages by accessing the other subdirectories.

```
<VirtualHost *:80>
  ServerName www.myapp.com
  DocumentRoot /var/www
  <Directory "/var/www">
    RewriteEngine on
    RewriteBase /
    RewriteRule ^../index\.html$ - [L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule (..) $1/index.html [L]
    RewriteCond %{HTTP:Accept-Language} ^fr [NC]
    RewriteRule ^$ /fr/ [R]
    RewriteCond %{HTTP:Accept-Language} ^es [NC]
    RewriteRule ^$ /es/ [R]
    RewriteCond %{HTTP:Accept-Language} !^es [NC]
    RewriteCond %{HTTP:Accept-Language} !^fr [NC]
    RewriteRule ^$ /en/ [R]
  </Directory>
</VirtualHost>
```

## NGINX configuration

Here is an NGINX configuration that will give you the same behaviour: an access to http://www.myapp.com will redirect to the preferred language defined in the browser (or `en` if your preferred language is not found) and the other languages are still accessible.

```
server {
    listen       80;
    server_name  localhost;

    location /en/ {
        alias   /usr/share/nginx/html/en/;
        try_files $uri$args $uri$args/ /en/index.html;
    }
    location /es/ {
        alias   /usr/share/nginx/html/es/;
        try_files $uri$args $uri$args/ /es/index.html;
    }
    location /fr/ {
        alias   /usr/share/nginx/html/fr/;
        try_files $uri$args $uri$args/ /fr/index.html;
    }

    set $first_language $http_accept_language;
    if ($http_accept_language ~* '^(.+?),') {
        set $first_language $1;
    }

    set $language_suffix 'en';
    if ($first_language ~* 'es') {
        set $language_suffix 'es';
    }
    if ($first_language ~* 'fr') {
        set $language_suffix 'fr';
    }

    location / {
        rewrite ^/$ http://localhost:4200/$language_suffix/index.html permanent;
    }
}
```

## Bonus: add links to the different languages

It would be interesting to have some links in the app so the user can navigate to another languages by clicking these links. The links will point to `/en/`, `/es/` and `/fr/`.

One trick to know, the current language is available in the `LOCALE_ID` token.

Here is how you can get the `LOCALE_ID` value and display the list of languages, differentiating the current language:

```
// app.component.ts
import { Component, LOCALE_ID, Inject } from '@angular/core';
@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  languages = [
    { code: 'en', label: 'English'},
    { code: 'es', label: 'Español'},
    { code: 'fr', label: 'Français'}
  ];
  constructor(@Inject(LOCALE_ID) protected localeId: string) {}
}
<!-- app.component.html -->
<h1 i18n>Hello World!</h1>
<ng-template ngFor let-lang [ngForOf]="languages">
  <span *ngIf="lang.code !== localeId">
    <a href="/{{lang.code}}/">{{lang.label}}</a> </span>
  <span *ngIf="lang.code === localeId">{{lang.label}} </span>
</ng-template>
```


### Bonus 2: Angular Translator application

For the 2017 AngularAttack, I've created an application that can definitely help you translate your Angular applications. It is still in development, feedbacks are welcome: http://angular-translator.elol.fr/.

Good translations!
