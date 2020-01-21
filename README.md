# Angular-Configurations-
Where To Store Angular Configurations 

Because this is a frequent problem, because it is so often done incorrectly and because there is a great alternative, today I want to discuss where to store Angular configurations. You know, all that information that changes as you move from your local development environment to the Development, QA and Production servers?

There’s a place for that!

Wrong Place

I know it is tempting, but the environment.ts and environment.prod.ts files were never meant for configuration information other than to tell the run-time you are running a production version of the code instead of developing the code locally. Yes, I know it is possible to create a file for your different environments and you can effectively use the file for your configuration information. But, just because you can, doesn’t mean you should.

In an ideal world, you would build a release candidate and place it on your Development server and then move it from there to QA and then to Production. You would never rebuild the application. You want to be absolutely sure that the code you tested in the Development environment is the code you ran in the QA environment and that the code you ran in the QA environment is the code that is running in the Production environment. You want to know for sure that the only possible reason why something isn’t working is because the configuration information is incorrect.

There are other ways to mitigate the risk, but they all involve recompiling the code and tagging your repository so you can get the same code back. This works when there isn’t a better way. But, there is a better way!
Where Instead?

If we can’t put our configuration information in our code, where do we put it? Obviously external to your code. This leaves us several solutions. One is to create a static json file that gets copied into your dist directory when the code is deployed to each environment. Another place that I’ve see work is to place the code in a database. The advantage to the database method is that you can have one database that handles the configuration information for all of your applications and even all of your environments. Put a good administration GUI on top of it and you can change the configuration easily without having to deploy even a file.
Jumping The Hurdle

Now that you’ve put your configuration information in an external location, you realize that you’ll need to make a GET request to retrieve that information. You may also quickly realize that you need that configuration information as soon as your application starts up. Maybe putting that information in an external file wasn’t such a great idea after all?

Well, not so fast!

There is a little known API feature in Angular that lets us load stuff up front and will actually wait until the loading has completed before continuing on with our code.
APP_INITIALIZER

APP_INITIALIZER is a multi provider type that lets you specify a factory that returns a promise. When the promise completes, the application will continue on. So when you get to the place in your code where you need the configuration information, you can be sure it has been loaded. It’s pretty slick.
```javascript

import { APP_INITIALIZER } from '@angular/core'

@NgModule({
    ....
    providers: [
        ...
        {
            provide: APP_INITIALIZER,
            useFactory: load,
            multi: true
        }
    ]
)
```
where load is a function that returns a function that returns a Promise<boolean>. The promise function loads your configuration information and stores it in your application. Once your configuration has been loaded, you resolve the promise using resolve(true).

This last point is really important. If you get this wrong, the code won’t wait for this to finish loading before moving on. useFactory points to a function that returns a function that returns a Promise<boolean>!

The multi: true thing is because APP_INITIALIZER allows multiple instances of this provider. They all run simultaneously, but the code will not continue beyond APP_INTITIALIZER until all of the Promises have resolved.
An example.

Now, as a discussion point, let’s assume that you have a regular Angular CLI based project and you need to load in the base location of your REST endpoints. You might have a config.json file that looks something like this:
```javascript
{
    "baseUrl": "https://davembush.github.io/api"
}
```
You would create a different one of these for each of the environments you wanted to deploy to, and, as part of your deployment process, you would copy the appropriate file to config.json in the same location that you deploy all of your Angular CLI generated static files.
Basic App Initializer

Now, the thing we want to do is to load this config file at runtime using APP_INITIALIZER. To do that, let’s add an APP_INITIALIZER provider to our app.module.ts file.
	
```javascript
import { APP_INITIALIZER } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';

function load() {

}

@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule
  ],
  providers: [{
    provide: APP_INITIALIZER,
    useFactory: load,
    multi: true
  }],
  bootstrap: [AppComponent]
})
export class AppModule { }
```
For now, you’ll notice, we are ignoring the implementation of the load() function. Typically, I add load to another file and import it, but for the purposes of explaining how this works, we will leave it as part of this code.
Type the load() function

Since I made such a big deal about the typing of the load function, we should type the load function to make sure it actually returns what we have in mind. The following is skeleton code that we will embellish.

```javascript
function load(): (() => Promise<boolean>) {
  return (): Promise<boolean> => {
    return new Promise<boolean>((resolve: (a: boolean) => void): void => {
      resolve(true);
    });
  };
}
```
At this point, the code doesn’t do anthing useful, but you should be able to compile and run the code at this point without seeing any errors in the console.
Proof that it waits

Next, we want to prove that this really is stopping the rest of our code from running. To do this, we are going to use a simple setTimeout() to simulate a long running process that we need to wait on to complete.
```javascript
function load(): (() => Promise<boolean>) {
  return (): Promise<boolean> => {
    return new Promise<boolean>((resolve: (a: boolean) => void): void => {
      setTimeout(() => resolve(true), 10000);
    });
  };
}
```
This code waits for 10 seconds before resolving. You can play with the timeout value to prove to yourself that this code really is waiting.
The fun begins

Now, the purpose of this code is to load the config.json file. For that we are going to need to get access to the HttpClient service. Normally, we use dependency injection to access this, and this code is no different.

Back in the provider, add a deps: [] section. To inject HttpClient, use:


deps: [HttpClient]

And change the load function to take HttpClient as a parameter.	
```javascript
function load(http: HttpClient): (() => Promise<boolean>) {
  return (): Promise<boolean> => {
    return new Promise<boolean>((resolve: (a: boolean) => void): void => {
      setTimeout(() => resolve(true), 10000);
    });
  };
}
```
Don’t forget to import HttpClientModule into your module.

And, once we load the configuration file, we will want to save the information some place. There are a couple of ways that you might do this, but the most Angular like would be to create a service. In our case our service will have one field named baseUrl.
```javascript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class ConfigService {
  baseUrl: string;
  constructor() { }
}
```
Since our service is using the new provideIn flag, we don’t need to worry about adding it to a module.

In order to set this, we will need to add ConfigService to the deps: array.
	
```javascript

deps: [
  HttpClient,
  ConfigService
],
  ```
And make it a parameter of our load() function.
	
```javascript
function load(http: HttpClient, config: ConfigService): (() => Promise<boolean>) {
  return (): Promise<boolean> => {
    return new Promise<boolean>((resolve: (a: boolean) => void): void => {
      setTimeout(() => resolve(true), 10000);
    });
  };
}
```
Loading config.json

The rest is pretty straight forward Angular code. We will
```javascript
    http.get(‘.config.json’)
    if it gives us a 404 error, we set default values
    if it succeeds we set the values from the data that got loaded.
```
Now, why would we get a 404 error?

Well, the one place where this file won’t be available is when we are using ng serve during development. But when we load this with a regular server like Apache, IIS, or Nginx, the file will be available.

Now, where we have the setTimeout() code, replace it with http.get() code.

```javascript
http.get('./config.json')
 .pipe(
   map((x: ConfigService) => {
     config.baseUrl = x.baseUrl;
     resolve(true);
   }),
   catchError((x: { status: number }, caught: Observable<void>): ObservableInput<{}> => {
     if (x.status !== 404) {
       resolve(false);
     }
     config.baseUrl = 'http://localhost:8080/api';
     resolve(true);
     return of({});
   })
 ).subscribe();
```
I’m not going to bother explaining all of the details of RxJS 6, you can look that all up if you are interested.
Caveats

Regardless of which of the two methods you use, you’ll need to make sure that your code will not need to change to access the database or the config file. If you are going to use a database, even then, I would not hard-code anything other than a relative URL. If the database is on some external server, make sure that you can proxy the code to is from your application’s server so that if and when the database location changes, you won’t need to recompile your code.
Full resulting code

Just in case I left out a step or you are otherwise lost or maybe all you really care about is the code so you can copy and paste it.

Here it is:

```javascript

import { APP_INITIALIZER } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { HttpClientModule, HttpClient } from '@angular/common/http';

import { AppComponent } from './app.component';
import { ConfigService } from './config.service';
import { of, Observable, ObservableInput } from '../../node_modules/rxjs';
import { map, catchError } from 'rxjs/operators';

function load(http: HttpClient, config: ConfigService): (() => Promise<boolean>) {
  return (): Promise<boolean> => {
    return new Promise<boolean>((resolve: (a: boolean) => void): void => {
       http.get('./config.json')
         .pipe(
           map((x: ConfigService) => {
             config.baseUrl = x.baseUrl;
             resolve(true);
           }),
           catchError((x: { status: number }, caught: Observable<void>): ObservableInput<{}> => {
             if (x.status !== 404) {
               resolve(false);
             }
             config.baseUrl = 'http://localhost:8080/api';
             resolve(true);
             return of({});
           })
         ).subscribe();
    });
  };
}
@NgModule({
  declarations: [
    AppComponent
  ],
  imports: [
    BrowserModule,
    HttpClientModule
  ],
  providers: [{
      provide: APP_INITIALIZER,
      useFactory: load,
      deps: [
        HttpClient,
        ConfigService
      ],
      multi: true
    }
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```
