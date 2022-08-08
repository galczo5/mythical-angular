---
title: "esbuild against TestBed"
date: 2022-07-24
draft: false
---

Isn't the unit testing the most basic technique and one of the more important aspects of programing these days?
During my days as a .NET developer, I learned a few things about unit testing:
1. Unit tests have to be **FAST**. If they are not fast, you're going to be irritated, and you'll waste a lot of time.
2. You have to **ISOLATE** your unit tests from the source code. If not, changes in the source code may affect tests and change the requirement behind the test.
3. Tests have to be **REPEATABLE**. No need to explain, random tests are not trustworthy.
4. You shouldn't have to check the code manually. Tests should be **SELF-VALIDATING**. If you have to check it manually even after a unit testing end, there is no reason to worry about unit tests.
5. Tests should be **THOROUGH**. If not, again, there is no reason to write them.

All of these rules are known as F.I.R.S.T. principles of unit testing.
These rules are pretty obvious and easy to understand. 

Unfortunately, I have a problem with unit testing in Angular.
The default stack is using Karma as a test runner,
so the test code is compiled, transpiled to JS,
after that is being sent to the new instance of the browser and then executed.
A lot of things can go wrong.
For example, the browser may crash during the tests, and boom, tests are not repeatable.
Another example, we have to compile the Angular code, and it's not fast at all.
Don't believe me...? 
Just compare how fun, and fast is having a project in Vite. 

We all know that Karma is not the only option.
For example, in NX projects, the default runner is Jest.
The test code looks almost the same, but instead of running in the browser it's executed in NodeJS.
In my opinion and experience, it's more stable.
There's still a problem with running time.
We still need to transpile TypeScript to JavaScript.
And without caches, with cold start, it's a very time-consuming process.

I don't want to discuss and describe in this article why the Angular building is slower than Vite, SWC or esbuild.
I just want to mention that for now, we cannot change it that easily,
and probably, we need to wait for the move from the Angular team. 

The situation is not very positive for us, Angular developers.
I'm still looking for the best way of not only writing but also running the unit tests.
Please remember, that probably, it's not that important when you have a few or even a dozens of tests.
As always, my perspective is based on the huge, enterprise scale apps,
where the Angular framework is a very good choice.
So instead of dozens of tests, we have tens of thousands of tests to run, and we want to run them as often as possible.
In the perfect world, we want to run all tests on every commit.

Why am I writing this article?

I think that there is a new possibility to avoid time-consuming tests.
I was able to run Angular tests in Jest using a custom transformer that is using esbuild under the hood.
It was much faster than the default configuration from NX, but there is a sacrifice too.
You cannot use the `TestBed` helper. 

Sounds scary.

I'm constantly thinking about it, and every second I'm closer to say that you want to get rid of `TestBed` anyway.
Unit testing is about testing "units", and it's not strictly defined term.
No one ever said that "unit" has to be 100 lines long file, containing only one class, etc.
In frontend, we very often assume that "unit" in our case is the whole component.
In other words, a component is the business logic attached to the view, that can be rendered and interact with a user.

`TestBed` helps us to create components and their dependencies.
If we stop using the `TestBed`, we have to handle this problem manually,
using constructors and what's more problematic we cannot test the views
(in Angular, might be different in other frameworks).
Without this helper, "unit" is smaller.
I assume that smaller units should be easier to test.
So at the end, maybe not having the `TestBed` is a good thing.
We can test visual part of the components using different types of tests, like e2e or storybook tests.
BTW. In Jest we don't have a real browser, so there is no actual rendering.

There is another problem that I have with `TestBed`.
It's very easy to mess it up, and very often are written more like integration tests than unit tests.
We can always say that it all depends on the developer,
and if the developer writes bad code, it's not the framework concern.
I strongly disagree with the sentences like this.
If it's very easy to make mistakes, and it was designed to be used by a lot of people, there always be messed-up code.
It's like creating two buttons, both red, without labels,
and information in the header says that left one is for deleting and the right one is cancel action.
If a framework ensures that you are not able to make simple mistakes, there is no way that you will make these mistakes.
It's a very intuitive way of thinking, so maybe I will show you what I did with the esbuild.

## What I did

Not so long ago, I was thinking about the duration of the tests in Jest,
and I realized that I have quite complicated tests, with multiple dependencies.

![tick](/images/esbuild.png)

For some time,
I knew that Angular CLI is not a very fast solution of bundling and newer bundlers like esbuild are much faster.
I even had a talk on Devoxx 2021 in Kraków about it.
I knew also
that there was [ngc-esbuild](https://github.com/cherryApp/ngc-esbuild)
that is trying using esbuild to run the Angular app.
I tried to use it, but it's not working with more complicated apps.
I had problems in the runtime in the browser, but the building process was really fast.
The author of ngc-esbuild says that it's 50 times faster than Angular CLI, and I think that it might be possible.  

It's maybe not ready to run the whole application,
but it looks that it's able to build a small part and run it as a unit test.

I tried to use esbuild transformer in Jest, but for some reason it had some problems with `@Injectable` decorator.
So, I decided to write my own, custom, transformer to solve the problem.
I thought that I could check how ngc-esbuild is solving the same difficulties,
but at the end there was no need to do the same.

All the code
you will find in the [ng-esbuild-transformer GitHub repository](https://github.com/galczo5/ng-esbuild-transformer).
It's published in the npm, and you can easily try it on your repository.
It's not production ready, so you do it at your own risk.  

```typescript
/**
 * https://jestjs.io/docs/next/code-transformation
 */

const {buildSync} = require('esbuild');
const {appendFileSync, readFileSync} = require('fs');
const {randomUUID} = require('crypto');
const {join, resolve} = require('path');

interface TransformerConfig {
    readonly outDir: string,
    readonly esbuildLogFilename: string,
    readonly verbose: boolean,
    readonly useCache: boolean,
    readonly esbuildConfig: any
}

const defaultConfig: TransformerConfig = {
    esbuildLogFilename: 'esbuild.log',
    outDir: './dist/ng-esbuild/',
    useCache: true,
    verbose: false,
    esbuildConfig: {}
}

interface TransformOptions {
    supportsDynamicImport: boolean;
    supportsExportNamespaceFrom: boolean;
    supportsStaticESM: boolean;
    supportsTopLevelAwait: boolean;
    instrument: boolean;
    cacheFS: Map<string, string>;
    configString: string;
    transformerConfig: TransformerConfig;
}

type TransformedSource = {
    code: string;
    map?: string | null;
};

module.exports = {
    process(sourceText: string, sourcePath: string, userOptions: TransformOptions): TransformedSource {
        const options = {
            ...defaultConfig,
            ...(userOptions && userOptions.transformerConfig ? userOptions.transformerConfig : {})
        };

        printVerbose(options, 'Processing file', sourcePath);

        const fileName = randomUUID().toString() + '.js';
        const outFile = resolve(join(options.outDir, fileName));

        printVerbose(options, 'Processed file will be saved as', outFile);

        const buildOptions = {
            ...options.esbuildConfig,
            entryPoints: [sourcePath],
            format: 'iife',
            platform: 'node',
            bundle: true,
            outfile: outFile,
        };

        printVerbose(options, 'Build options', JSON.stringify(buildOptions));
        printVerbose(options, 'Build started');
        const buildResult = buildSync(buildOptions);
        printVerbose(options, 'Build completed');

        logIfNecessary(options, buildResult);

        printVerbose(
            options,
            'Esbuild result',
            `errors: ${buildResult.errors.length}`,
            `warnings: ${buildResult.warnings.length}`
        );

        const result = readFileSync(outFile).toString();

        return {
            code: result
        };
    }
};

function logIfNecessary(options: TransformerConfig, buildResult: unknown) {
    if (options.esbuildLogFilename) {
        const buildLogFile = resolve(join(options.outDir, options.esbuildLogFilename));
        const buildResultJson = JSON.stringify(buildResult, null, 2);
        const logText = `${new Date().toISOString()}\n${buildResultJson}\n`;
        appendFileSync(buildLogFile, logText);

        printVerbose(options, 'Build log saved in file', buildLogFile);
    }
}

function printVerbose(options: TransformerConfig, ...msg: Array<string>): void {
    if (options.verbose) {
        console.log(new Date().toISOString(), '[ng-esbuild]', ...msg);
    }
}
```

My transformer is relatively easy. It's just starting a synchronous esbuild process and returns its results. 

I was able to run simple tests on it, and measure the time.
The results were very satisfying.
**Instead of 31s on ts-jest I had only 6s in the first test.
Both builds were not using cache etc.**

Remember that the tests cannot use `TestBed`.
I tried to find the alternative, and there is one think that you can use.
It's not super automatic, super intuitive or super developer friendly, but it works.

## The `TestBed` problem

I found
that I still can use part of the Angular framework as a helper to create necessary components and its dependencies. 

```typescript
// Needed by Injector
import '@angular/compiler';

import { AppComponent } from './app.component';
import {AppService} from "./app.service";
import {Injector} from "@angular/core";

describe('AppComponent', () => {

    it('Injector helper - instead of TestBed', async () => {
        const injector = Injector.create({
            providers: [
                // Have to provide deps directly into the provider because we are not transforming the constructor
                { provide: AppComponent, useClass: AppComponent, deps: [AppService] },
                { provide: AppService, useClass: AppService }
            ]
        });

        const service = injector.get(AppService);
        expect(service.isEnabled()).toBeFalsy();

        const component = injector.get(AppComponent);
        expect(component.isEnabled()).toBeFalsy();
    })

});
```

It looks that after importing the `@angular/compiler` into the test we are able to use `Injector` to manage our objects.
There is only one problem, we have to directly point dependencies using `deps` property.

We can omit this problem,
but it requires us to use `@Inject()` in components or other injectables instead of shorthands.
I don't like it, because it influences the code of the app, not only the test. 

```typescript
import {Component, Inject} from '@angular/core';
import {AppService} from "./app.service";

@Component({
  selector: 'jest-ngc-esbuild-transformer-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css'],
})
export class AppComponent {

  constructor(
      private readonly appService: AppService,
      // You don't have to use deps if you do:
      // @Inject(AppService) private readonly appService: AppService,
  ) {
  }

  isEnabled(): boolean {
    return this.appService.isEnabled();
  }
}
```

You can use `Injector`. It's not very helpful.
BTW it prevents you from writing "too big" tests, so it has the bright side.

## Summary

I'm still not sure what to think about this way of testing.
On the one hand, I have fast tests.
On another, I have the `TestBed`.

I just wanted to show you the possibilities and the profits of using esbuild in Angular unit testing.
Maybe we can evolve this idea, and describe this way well.

You know my opinion, it's the time for your reflections.  
