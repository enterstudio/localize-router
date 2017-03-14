# localize-router
[![Build Status](https://travis-ci.org/Greentube/localize-router.svg?branch=master)](https://travis-ci.org/Greentube/localize-router)
[![npm version](https://img.shields.io/npm/v/localize-router.svg)](https://www.npmjs.com/package/localize-router)
> An implementation of routes localization for Angular.

Based on and extension of [ngx-translate](https://github.com/ngx-translate/core).
Demo project can be found [here](https://github.com/meeroslav/localize-router-example).

# Table of contents:
- [Installation](#installation)
- [Usage](#usage)
    - [Initialize module](#initialize-module)
        - [Static initialization](#static-initialization)
        - [JSON config file](#json-config-file)
        - [Manual initialization](#manual-initialization)
        - [Server side initialization](#server-side-initialization)
    - [How it works](#how-it-works)
        - [ngx-translate integration](#ngx-translate-integration)
    - [Pipe](#pipe)
    - [Service](#service)
    - [AOT](#aot)
- [API](#api)
    - [LocalizeRouterService](#localizerouterservice)
    - [LocalizeParser](#localizeparser)
- [License](#license)

## Installation

```
npm install --save localize-router
```

## Usage

In order to use `localize-router` you must initialize it with following information:
* Available languages/locales
* Prefix for route segment translations
* Routes to be translated

### Initialize module
Module can be initialized either using static file or manually by passing necessary values.

#### Static initialization
```ts
import {BrowserModule} from "@angular/platform-browser";
import {NgModule} from '@angular/core';
import {HttpModule} from '@angular/http';
import {TranslateModule} from '@ngx-translate/core';
import {LocalizeRouterModule} from 'localize-router/localize-router';
import {RouterModule} from '@angular/router';

import {routes} from './app.routes';

@NgModule({
    imports: [
        BrowserModule,
        HttpModule,
        TranslateModule.forRoot(),
        LocalizeRouterModule.forRoot(routes),
        RouterModule.forRoot(routes)
    ],
    bootstrap: [AppComponent]
})
export class AppModule {
}
```

Static file's default path is `assets/locales.json`. You can override the path by calling `StaticParserLoader` on you own:
```ts
LocalizeRouterModule.forRoot(routes, {
    provide: LocalizeParser,
    useFactory: (translate, location, http) =>
        new StaticParserLoader(translate, location, http, 'your/path/to/config.json'),
    deps: [TranslateService, Location, Http]
})

```

If you are using child modules or routes you need to initialize them with `forChild` command:
```ts
@NgModule({
  imports: [
    TranslateModule,
    LocalizeRouterModule.forChild(routes),
    RouterModule.forChild(routes)
  ],
  declarations: [ChildComponent]
})
export class ChildModule { }
```

#### JSON config file
JSON config file has following structure:
```
{
    "locales": ["en", "de", ...],
    "prefix": "..."
}
```

```ts
interface ILocalizeRouteConfig {
    locales: Array<string>;
    prefix: string;
}
```

#### Manual initialization
With manual initialization you need to provide information directly:
```ts
LocalizeRouterModule.forRoot(routes, {
    provide: LocalizeParser,
    useFactory: (translate, location) =>
        new ManualParserLoader(translate, location, ['en','de',...], 'YOUR_PREFIX'),
    deps: [TranslateService, Location]
})

```

#### Server side initialization
In order to use server side initialization in isomorphic/universal projects you need to create loader similar to this:
```ts
export class LocalizeUniversalLoader extends LocalizeParser {
  /**
   * Gets config from the server
   * @param routes
   */
  public load(routes: Routes): Promise<any> {
    return new Promise((resolve: any) => {
      let data: any = JSON.parse(fs.readFileSync(`assets/locales.json`, 'utf8'));
      this.locales = data.locales;
      this.prefix = data.prefix;
      this.init(routes).then(resolve);
    });
  }
}

export function localizeLoaderFactory(translate: TranslateService, location: Location) {
  return new LocalizeUniversalLoader(translate, location);
}

```

Don't forget to create similar loader for `ngx-translate` as well:
```ts
export class TranslateUniversalLoader implements TranslateLoader {
  /**
   * Gets the translations from the server
   * @param lang
   * @returns {any}
   */
  public getTranslation(lang: string): Observable<any> {
    return Observable.create(observer => {
      observer.next(JSON.parse(fs.readFileSync(`src/assets/locales/${lang}.json`, 'utf8')));
      observer.complete();
    });
  }
}
export function translateLoaderFactory() {
  return new TranslateUniversalLoader();
}
```

Since node server expects to know which routes are allowed you can feed it like this:
```ts
let fs = require('fs');
let data: any = JSON.parse(fs.readFileSync(`src/assets/locales.json`, 'utf8'));

app.get('/', ngApp);
data.locales.forEach(route => {
  app.get(`/${route}`, ngApp);
  app.get(`/${route}/*`, ngApp);
});
```

Working example can be found [here](https://github.com/meeroslav/universal-localize-example).

### How it works

`Localize router` intercepts Router initialization and translates each `path` and `redirectTo` path of Routes.
The translation process is done with [ngx-translate](https://github.com/ngx-translate/core). In order to separate 
router translations from normal application translations we use `prefix`. Default value for prefix is `ROUTES.`.
```
'home' -> 'ROUTES.home'
```

Upon every route change `Localize router` kicks in to check if there was a change to language. Translated routes are prepended with two letter language code:
```
http://yourpath/home -> http://yourpath/en/home
```

If no language is provided in the url path, application uses: 
* cached language in LocalStorage (browser only) or
* current language of the browser (browser only) or 
* first locale in the config

Make sure you therefore place most common language (e.g. 'en') as a first string in the array of locales.

> Note that `localize-router` does not redirect routes like `my/route` to translated ones e.g. `en/my/route`. All routes are prepended by currently selected language so route without language is unknown to Router.

#### ngx-translate integration

`LocalizeRouter` depends on `ngx-translate` core service and automatically initializes it with selected locales.
Following code is run on `LocalizeParser` init:
```ts
this.translate.setDefaultLang(cachedLanguage || languageOfBrowser || firstLanguageFromConfig);
// ...
this.translate.use(languageFromUrl || cachedLanguage || languageOfBrowser || firstLanguageFromConfig);
```

Both `languageOfBrowser` and `languageFromUrl` are cross-checked with locales from config.

### Pipe

`LocalizeRouterPipe` is used to translate `routerLink` directive's content. Pipe can be appended to partial strings in the routerLink's definition or to entire array element:
```html
<a [routerLink]="['user', userId, 'profile'] | localize">{{'USER_PROFILE' | translate}}</a>
<a [routerLink]="['about' | localize]">{{'ABOUT' | translate}}</a>
```

Root routes work the same way with addition that in case of root links, link is prepended by language.
Example for german language and link to 'about':
```
'/about' | localize -> '/de/über'
```

### Service

Routes can be manually translated using `LocalizeRouterService`. This is important if you want to use `router.navigate` for dynamical routes.

```ts
class MyComponent {
    constructor(private localize: LocalizeRouterService) { }

    myMethod() {
        let translatedPath: any = this.localize.translateRoute('about/me');
       
        // do something with translated path
        // e.g. this.router.navigate([translatedPath]);
    }
}
```

### AOT

In order to use Ahead-Of-Time compilation any custom loaders must be exported as functions.
This is the implementation currently in the solution:

```ts
export function localizeLoaderFactory(translate: TranslateService, location: Location, http: Http) {
  return new StaticParserLoader(translate, location, http);
}
```

## API
### LocalizeRouterService
#### Properties:
- `routerEvents`: An EventEmitter to listen to language change event
```ts
localizeService.routerEvents.subscribe((language: string) => {
    // do something with language
});
```
- `parser`: Used instance of LocalizeParser
```ts
let selectedLanguage = localizeService.parser.currentLang;
```
#### Methods:
- `translateRoute(path: string | any[]): string | any[]`: Translates given path. If `path` starts with backslash then path is prepended with currently set language.
```ts
localizeService.translateRoute('/'); // -> e.g. '/en'
localizeService.translateRoute('/about'); // -> '/de/uber' (e.g. for German language)
localizeService.translateRoute('about'); // -> 'uber' (e.g. for German language)
```
- `changeLanguage(lang: string)`: Translates current url to given language and changes the application's language
For german language and route defined as `:lang/users/:user_name/profile`
```
yoursite.com/en/users/John%20Doe/profile -> yoursite.com/de/benutzer/John%20Doe/profil
```
### LocalizeParser
#### Properties:
- `locales`: Array of used language codes
- `currentLang`: Currently selected language
- `routes`: Active translated routes

#### Methods:
- `translateRoutes(language: string): Observable<any>`: Translates all the routes and sets language and current 
language across the application.
- `translateRoute(path: string): string`: Translates single path
- `getLocationLang(url?: string): string`: Extracts language from current url if matching defined locales

## License
Licensed under MIT
