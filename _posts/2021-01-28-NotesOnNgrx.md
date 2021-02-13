---
layout: post
title: Notes on OOP Coders course for Ngrx
---

This is a work in progress.

Notes while working through [this course](https://www.youtube.com/watch?v=oHmreG1Sul0&list=PLV-DQnYj14bRFWMmuT6ptSL4v5fxMJnOS&index=1) on YouTube by OOP Coders.

The [source code is here](https://github.com/oopcoders/NGRX-Course)

click on commits
scroll to "7. begin" and copy the commit id: 3efc22acd6e73ff41c84f70f770a548560de027b

## Create Project is VS Code

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

## Add Ngrx to the Project

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

## Add Ngrx Dev Tools to Project

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

# Add Ngrx Schematics

This adds Nrgx support to the angular cli for adding store, actions, etc.

Install the schematics tools by typing the following in a terminal

```console
ng add @ngrx/schematics@latest
```

enter (Yes) for use default collection.

this makes ngrx the defaultCollection so when 

```json
{
...
"cli": {
    "defaultCollection": "@ngrx/schematics",
    "analytics": "46d8b363-06f6-4bbf-93b8-be84098b2f8f"
  }
...
}
```

this makes ngrx the defaultCollection so when you use the cli to create ngrx components 
you don't need to prepend what your generating with the schematic collection name:

i.e. you can type this 
```console
ng g action
```

instead of having to type this

```console
ng g @ngrx/schematics:action
```

see https://ngrx.io/guide/schematics

## Generate a Store

generate a store see https://ngrx.io/guide/schematics/store

```console
ng generate store store --module=app.module.ts --root=true --statePath=store --stateInterface=AppState
```
* --module tells it to add the store to the specified module, you most likely want to use app.module.ts  
* --root=true configures the store to be shared, and uses forRoot see the imports section in app.module.ts

```typescript
...
imports: [
...
StoreModule.forRoot(reducers, {
  metaReducers,
  runtimeChecks: {
    strictStateImmutability: true,
	strictActionImmutability: true,
  }
  })
...
  ]
```

* --statePath tells it what to name the folder for the store files, see the index.ts file in the store folder
```typescript
import {
  ActionReducer,
  ActionReducerMap,
  createFeatureSelector,
  createSelector,
  MetaReducer
} from '@ngrx/store';
import { environment } from '../../environments/environment';

export interface AppState {
}

export const reducers: ActionReducerMap<AppState> = {
};


export const metaReducers: MetaReducer<AppState>[] = !environment.production ? [] : [];
```
 
* --stateInterface tells it what to name the class used for holding the state. 
see empty interface AppState in the index.ts file in the store folder.
```typescript
export interface AppState {
}
```


* we can remove the original code added to the imports in app.module.ts when we created the store:
```typescript
...
imports: [
...
StoreModule.forRoot({},{})
...
  ]

```

* restart the app because we made changes to app.moddule.ts

```console
npm run dev
```

##Actions

Actions communicate store related activity in your application. For example, user logs in, 
user buys something, user lists products. It also identifies activity from the application, user
login succeeded, user login failed. If you use use cases (not a use case diagram), these are 
basically the messages that get passed around.

Now add some actions https://ngrx.io/guide/store/actions 
Using the schematics https://ngrx.io/guide/schematics/action

```commandline
ng generate action store/actions/customer-support --skipTests=true --api=true

Do you want to use the create function? (Y/n) Y
```

This generates the file store/actions/customer-support.actions.ts

* --skipTests=true says not to generate a test spec for the action
* --api=true says to generate stubs for Load/Load Success/Load Failure
* entering Y for using the create function makes the schematic generate the action classes
using the createAction function like this:

```typescript
import { createAction, props } from '@ngrx/store';

export const loadCustomerSupports = createAction(
  '[CustomerSupport] Load CustomerSupports'
);

export const loadCustomerSupportsSuccess = createAction(
  '[CustomerSupport] Load CustomerSupports Success',
  props<{ data: any }>()
);

export const loadCustomerSupportsFailure = createAction(
  '[CustomerSupport] Load CustomerSupports Failure',
  props<{ error: any }>()
);
```

* entering N for using the create function makes the schematic generate the action classes
explicitly like this:

```typescript
import { Action } from '@ngrx/store';

export enum CustomerSupportActionTypes {
  LoadCustomerSupports = '[CustomerSupport] Load CustomerSupports',
  LoadCustomerSupportsSuccess = '[CustomerSupport] Load CustomerSupports Success',
  LoadCustomerSupportsFailure = '[CustomerSupport] Load CustomerSupports Failure',
}

export class LoadCustomerSupports implements Action {
  readonly type = CustomerSupportActionTypes.LoadCustomerSupports;
}

export class LoadCustomerSupportsSuccess implements Action {
  readonly type = CustomerSupportActionTypes.LoadCustomerSupportsSuccess;
  constructor(public payload: { data: any }) { }
}

export class LoadCustomerSupportsFailure implements Action {
  readonly type = CustomerSupportActionTypes.LoadCustomerSupportsFailure;
  constructor(public payload: { error: any }) { }
}

export type CustomerSupportActions = LoadCustomerSupports | LoadCustomerSupportsSuccess | LoadCustomerSupportsFailure;

```

The type is a string that identifies the action for example
'[CustomerSupport] Load CustomerSupports Success' the part in the brackets is the source of the
action '[CustomerSupport]' and the part outside is what is happening 'Load CustomerSupports Success'

* In the video the instructor says to use a very specific sources, he suggests the name of the
component generating it. This is very helpful for debugging.

In customer-support.actions.ts add an action for sending a message and import the CustomerMessage class

```typescript
...
import { CustomerMessage } from 'src/app/shared/models/customer-message';
...
export const sendingCustomerSupportMessage = createAction(
  '[Customer Support Component] Sending Customer Support Message',
  props<{ data: CustomerMessage }>()
);

```

In the customer-support.component.ts add the store to the constructor The type in Store<AppState> comes
from the type defined in store/index.ts

```typescript
constructor(private customerSupportService: CustomerSupportService, private store: Store<AppState>) {}
```

Then use the store to dispatch the action, replacing the customerSupportService.sendMessage()

```typescript
 onSubmit(f: NgForm) {
    this.customerSupportService.sendMessage(f.value).subscribe((success) => {
      console.log(success);
      this.isSendSuccess = success;
    });
  }
```

Use sendingCustomerSupportMessage() from customer-support.actions.ts passing the message
data from the form

```typescript
 onSubmit(f: NgForm) {
    this.store.dispatch(sendingCustomerSupportMessage({data: f.value}));
  }
```

Restart the application

```console
npm run dev
```

go to the [support page](http://localhost:4200/support) 

open the dev tools and go to the Redux tab

enter a message in the form and watch the left side you should see the event type

```
[Customer Support Component] Sending Customer Support Message
```

click on the Action button on the right side to see the contents of the message, the Raw
tab shows it as an object. the message is the data as defined in the action

```javascript
{
  data: {
    name: 'Bob',
    email: 'bbb@bbb.com',
    message: 'This is the message'
  },
  type: '[Customer Support Component] Sending Customer Support Message'
}
```

## Reducers

Reducers "change" state by taking an action and the current state and producing a new state.

There are four steps:


### Create the reducer

```commandline
ng generate reducer store/reducers/customer-support --api=true --skipTests=true --reducers=../index.ts
? Do you want to use the create function? Yes

CREATE src/app/store/reducers/customer-support.reducer.ts (251 bytes)
UPDATE src/app/store/index.ts (623 bytes)

```
* --reducers=../index.ts makes it add the reducer info to the index.ts file
  * adds "import * as fromCustomerSupport from './reducers/customer-support.reducer';"
  * adds "[fromCustomerSupport.customerSupportFeatureKey]: fromCustomerSupport.State;" in the AppState
    * this is adding a property to the AppState interface with key 'customerSupport' that refers to the current State.
  * adds "[fromCustomerSupport.customerSupportFeatureKey]: fromCustomerSupport.reducer," to the reducers
    * this is mapping the customer support reducer to 'customerSupport' key in the map of all the reducers.

```typescript
import {
  ActionReducer,
  ActionReducerMap,
  createFeatureSelector,
  createSelector,
  MetaReducer
} from '@ngrx/store';
import { environment } from '../../environments/environment';
import * as fromCustomerSupport from './reducers/customer-support.reducer';


export interface AppState {

  [fromCustomerSupport.customerSupportFeatureKey]: fromCustomerSupport.State;
}

export const reducers: ActionReducerMap<AppState> = {

  [fromCustomerSupport.customerSupportFeatureKey]: fromCustomerSupport.reducer,
};


export const metaReducers: MetaReducer<AppState>[] = !environment.production ? [] : [];
```
* --api=true defaults to adding success and failure actions to the reducer, if you leave it off it asks

```commandline
? Should we add success and failure actions to the reducer? No
```
* **note** I don't see any difference, but that is probably because the success and failure actions were
removed from the customer-support.actions.ts in an earlier step.

* Answering Yes to "Do you want to use the create function?" uses the createReducer()

```typescript
import { Action, createReducer, on } from '@ngrx/store';


export const customerSupportFeatureKey = 'customerSupport';

export interface State {

}

export const initialState: State = {

};

export const reducer = createReducer(
  initialState,

);
```

* Answering No to "Do you want to use the create function?" creates the reducer function

```typescript
import { Action } from '@ngrx/store';


export const customerSupportFeatureKey = 'customerSupport';

export interface State {

}

export const initialState: State = {

};

export function reducer(state = initialState, action: Action): State {
  switch (action.type) {

    default:
      return state;
  }
}
```

### Define the State

add properties to State in the reducers file

```typescript
export interface State {
  name: string;
  email: string;
  message: string;
}
```

###Define the values of the initial state

```typescript
export const initialState: State = {
  name: null,
  email: null,
  message: null
}
```

###Add handlers for the Actions to the reducer

In createReducer add on() that takes the action and a function that takes the previous state
and the action and returns a new state.

```typescript
export const reducer = createReducer(
...
on(sendingCustomerSupportMessage, (state, action) => {
    return {
      ...state,
      ...action.data
    };
    };
  })
...
```
* sendingCustomerSupportMessage is the action defined in ../actions/customer-support.actions
* ...state uses the spread operator to copy all the properties of the previous state into the
new object being created
* ...action.data uses the spread operator to copy all the properties in the action data into the
new object, adding to or replacing the values from the previous state

run the app and go to the [support form](http://localhost:4200/support) open the dev tools and
go to the Redux tab.

fill in the form and submit.

* see the action on Action tab
* click on the State tab and see that the customerSupport has the values from the form

##Selectors

Pull data from the store

### Create selector

```comandline
ng generate selector store/selectors/customer-support --skipTests=true
```

this creates /store/selectors/customer-support.selectors.ts

```typescript
import { createFeatureSelector, createSelector } from '@ngrx/store';
```

* importing createSelector is the function for creating the selector to get the name out of the State
** selectName uses the selectCustomerSupportFeature to get the State and a function to get name out of the state
** you can pass in multiple features to combine data from different source in the fuction
* importing createFeatureSelector is used for getting the customer support data from the store
** selectCustomerSupportFeature uses the customerSupportFeatureKey to get the State defined by the reducer

```typescript
import { createFeatureSelector, createSelector } from '@ngrx/store';
import { State, customerSupportFeatureKey } from '../reducers/customer-support.reducer';
  
  //Get feature from store
  export const selectCustomerSupportFeature = createFeatureSelector<State>(
    customerSupportFeatureKey
  );
  
  //Return name from feature
  export const selectName = createSelector(
    selectCustomerSupportFeature,
    (state: State) => state.name
  );
```

### Use the Selector in the Support Form

edit customer-support.component.ts to use the selector to get the name of the user after submitting
* add a variable to hold the Observable for the name
* in ngOnInit create the Observable
* When the form is submitted, set isSendSuccess to true

```typescript
...
  name$: Observable<string>;
...
  ngOnInit(): void {
    this.name$ = this.store.pipe(select(selectName));
  }
...
  onSubmit(f: NgForm) {
    this.store.dispatch(sendingCustomerSupportMessage({ data: f.value }));
    this.isSendSuccess = true;
```

edit customer-support.component.html to include the name in the success message
* use the name$ Observable with interpolation and the async pipe

```html
...
    <ng-container *ngIf="isSendSuccess != null">
        <div class="alert alert-dismissible alert-success" *ngIf="isSendSuccess">
            <button type="button" class="close" data-dismiss="alert" (click)="clearFeedback()">&times;</button>
            <strong>We got your message {{name$ | async}}!</strong> We will be in touch.
        </div>
...
    </ng-container>
```