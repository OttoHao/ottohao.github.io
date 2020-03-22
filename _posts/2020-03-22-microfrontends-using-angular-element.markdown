---
layout: post
title:  "Micro Frontends Using Angular Element"
date:   2020-03-22 18:03:18 +0800
categories: microfrontends angular
---

There are many ways to implement Micro Frontends. One of the approaches is using web component. Luckily, Angular provides an Angular Elements library ([@angular/elements](https://angular.io/guide/elements)) to support packging Angular components as web components and defining new HTML elements in a framework-agnostic way.

This post describes how to implement Micro Frontends using Angular Element. First, I will talk about how to build each micro frontend, and then the integration of all micro frontends into a container application will be addressed. In the end, I will discuss how to share common dependencies among micro frontends.

## 1. Building Micro Frontends

### 1.1 Define a custom element

```
ng add @angular/elements
```

[@angular/elements](https://angular.io/guide/elements) provides support for Angular elements.

```typescript
export class AppModule implements DoBootstrap {

 constructor(private injector: Injector) {
 }

 ngDoBootstrap() {
   const customElement = createCustomElement(MicroFrontendSampleComponent, { injector: this.injector });
   customElements.define('micro-frontend-sample', customElement);
 }
}
```

Angular provides the `createCustomElement()` function for converting an Angular component, together with its dependencies, to a custom element (also called web component). Use a JavaScript function, `customElements.define()`, to register the configured constructor and its associated custom-element tag with the browser's CustomElementRegistry. When the browser encounters the tag for the registered element, it uses the constructor to create a custom-element instance.

```typescript
 bootstrap: []
 entryComponents: [MicroFrontendSampleComponent]
```

Remember to empty the bootstrap array in app module in order to avoid conflicts between `ngDoBootstrap()`, and add the `MicroFrontendSampleComponent` into `entryComponents` array.

### 1.2 Webpack as a single bundle

```
ng add ngx-build-plus
ng build --prod --output-hashing none --single-bundle true
```

`--output-hashing none` will avoid hashing the output file names.

`--single-bundle true` will bundle all the compiled files into a single JS file. We get one bundle called main instad of `main`, `vendor` and `runtime`.
However, `scripts`, `styles`, and `polyfills` are still put into a bundle of their own. The reason behind this is, that it's very likely that the container application already loads these aspects and you don't want to load them again.

## 2. Integration in Container Application

The integration is easy. The first step is to add the micro frontend bundled `main.js` into script and then add the custom element in the container application.

```javascript
// add script
const script = document.createElement('script');
script.src = '[...]/main.js';
script.onload = ()=> { this.renderMicroFrontend(); };
document.body.appendChild(script);

// add custom element
renderMicroFrontend(){
  const root = document.getElementById('root');
  const microFrontend = document.createElement('micro-frontend-sample');
  root.appendChild(microFrontend);  
}
```

Here the src of the script `[...]/main.js` should be replaced with real url, and the `micro-frontend-sample` is the custom element in the browser's CustomElementRegistry registed by the micro frontend.

## 3 Share common dependencies between micro frontends

### 3.1 Sharing common dependencies is a trade off

Until now, the implementation of Micro Frontends using Angular Element is almost done. The container application can run successfuly and render the custom element registed by the micro frontend.

However, there will be code duplication within the micro frontends bundles, because each of them has its own dependencies. If all the micro frontends are implemented using Angular, they will have duplicate dependencies, such as Angular itself, RxJS. Therefore, it would be better to share common dependencies.

**One thing I want to make it clear is that sharing common dependencies is a trade off bewteen performance and coupling. On the one side, sharing common dependencies will reduce the size and loading time, and improve the performance. On the other side, it will bring couplings because the container application need to know the details of libraries or frameworks about the micro frontends, and the micro frontends are better to choose the same libraries and same framework even in the same version.**

### 3.2 Webpack externals

One solution for sharing commin dependencies is Webpack externals.

Externals allow us to share common libraries. For this, they are just loaded so that they can be referenced via the global namespace. In the case of most libraries we can use UMD bundles which do exactly this besides other things. Then, we have to tell webpack to not bundle them together with every micro frontend but to reference it within the global namespace instead.

```
ng g ngx-build-plus:externals
```

This script will:

 1. Introduces new npm scripts: `build:externals`, `npx-build-plus:copy-assets`
 2. Generate a new js file: `copy-bundles.js`, which copies over UMD bundles to the project's assets folder
 3. Import bundles from assets folder into `index.html`
 4. Generate a new js file: `webpack.externals.js`

 ```javascript
const webpack = require('webpack');

module.exports = {
    "externals": {
        "rxjs": "rxjs",
        "@angular/core": "ng.core",
        "@angular/common": "ng.common",
        "@angular/common/http": "ng.common.http",
        "@angular/platform-browser": "ng.platformBrowser",
        "@angular/platform-browser-dynamic": "ng.platformBrowserDynamic",
        "@angular/compiler": "ng.compiler",
        "@angular/elements": "ng.elements",
        "@angular/router": "ng.router",
        "@angular/forms": "ng.forms"
    }
}
```

This, for instance, makes the produced bundle to reference the global variable `ng.core` when it needs @angular/core. Hence, @angular/core does not need to be part of the bundle anymore.

In the container application, scripts need to be added in `angular.json`.

```json
"scripts": [
  "node_modules/rxjs/bundles/rxjs.umd.js",
  "node_modules/@angular/core/bundles/core.umd.js",
  "node_modules/@angular/common/bundles/common.umd.js",
  "node_modules/@angular/common/bundles/common-http.umd.js",
  "node_modules/@angular/compiler/bundles/compiler.umd.js",
  "node_modules/@angular/elements/bundles/elements.umd.js",
  "node_modules/@angular/platform-browser/bundles/platform-browser.umd.js",
  "node_modules/@angular/platform-browser-dynamic/bundles/platform-browser-dynamic.umd.js"
]
```

These UMD bundles put Angular into window.ng where it can be reused by several separately compiled micro frontends.

The name of global variable can be found in the UMD javascript file. A code snippet in `core.umd.js` of @angular/core is shown as an example.

```javascript
function (global, factory) {
    typeof exports === 'object' && typeof module !== 'undefined' ? factory(exports, require('rxjs'), require('rxjs/operators')) :
    typeof define === 'function' && define.amd ? define('@angular/core', ['exports', 'rxjs', 'rxjs/operators'], factory) :
    (global = global || self, factory((global.ng = global.ng || {}, global.ng.core = {}), global.rxjs, global.rxjs.operators));
}
```
