---
layout: post
title: Notes on OOP Coders course for Ngrx
---

Notes while working through [this course](https://www.youtube.com/watch?v=oHmreG1Sul0&list=PLV-DQnYj14bRFWMmuT6ptSL4v5fxMJnOS&index=1) on YouTube by OOP Coders.

The [source code is here](https://github.com/oopcoders/NGRX-Course)

click on commits
scroll to 7. begin and copy the commit id: 3efc22acd6e73ff41c84f70f770a548560de027b


In VS Projects C:\Users\rregan\VsProjects\  

```console
git clone https://github.com/oopcoders/NGRX-Course.git
git checkout 3efc22acd6e73ff41c84f70f770a548560de027b
```

open is vs code 

```console
cd NGRX-Course
code .
```

Install the ngrx store, open a terminal and type the following 

```console
ng add @ngrx/store@latest
```

see what it did:

in app.module.ts

```typescript
 import { StoreModel } from '@ngrx/store'
```
 and in the imports 
```typescript
 StoreModule.forRoot({},{})
```

In package.json under "dependencies"
 
```json
...
  "dependencies": {
  ...
    "@ngrx/store": "^10.1.2",
  ...
  }
...
}
```

Install the ngrx dev tools in a terminal type the following

```console
ng add @ngrx/store-devtools@latest
```

see what it did:

in app.module.ts
```typescript
    StoreDevtoolsModule.instrument({
      maxAge: 25,
      logOnly: environment.production,
    }),
    !environment.production ? StoreDevtoolsModule.instrument() : [],
```

you can configure the instrumentations, see https://ngrx.io/guide/store-devtools/config
 
In package.json under "dependencies"
 
```json
{
...
  "dependencies": {
  ...
    "@ngrx/store-devtools": "^10.1.2",
  ...
  }
...
}
```

run the application server and client in a terminal window

```console
npm run dev
```

In the browser go to the application
* http://localhost:4201/
* open the dev tools
* see the Redux tab
see https://ngrx.io/guide/store-devtools for more on dev tools

Install the schedmatics tools by typing the following in a terminal

```console
ng add @ngrx/schematics@latest
```

enter (Yes) for use default collection

see https://ngrx.io/guide/schematics

generate a store see https://ngrx.io/guide/schematics/store

```console
ng generate store store --module=app.module.ts --root=true --statePath=store --stateInterface=AppState
```

* see the index.ts file in the store folder (store was used because of --statePath)
* see empty interface AppState, this is becaues of --stateInterface
* see the app.module.ts file, in the imports

```typescript
StoreModule.forRoot(reducers, {
  metaReducers,
  runtimeChecks: {
    strictStateImmutability: true,
	strictActionImmutability: true,
  }
  })
```
* forRoot was used because of --root=true
* we can remove the original code injected:
```typescript
StoreModule.forRoot({},{})
```
that was generated when we first added the ngrx/store

* restart the app because we made changes to app.moddule.ts

Doing real stuff

Now add some actions https://ngrx.io/guide/store/actions 
Using the schematics https://ngrx.io/guide/schematics/action


