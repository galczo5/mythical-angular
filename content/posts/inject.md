---
title: "Angular 14 `inject()` implementation analysis"
date: 2022-06-03
draft: false
---

Angular 14 brings us a very cool feature, function which we can use to inject our dependencies.
First time,
I read about it in [an article
written by Netanel Basal](https://netbasal.com/unleash-the-power-of-di-functions-in-angular-2eb9f2697d66).
It looked very cool and a little mysterious.
Now, when Angular 14 is officially released, we can check how it works and how they implemented it.

In the article, Netanel described `inject()` interesting behaviour:

> First, inject() can only be called in the constructor, so we can’t leverage a component’s inputs. We can work around it by using closures, but did we gain anything?

I'll try to explain why it works only during component construction.

## Investigation

My work today I'm starting with generating a basic app, adding new injectable to it and modifying `AppComponent`.

```typescript
// app.component.ts

import {Component, inject} from '@angular/core';
import {TestService} from "./test.service";

@Component({
  selector: 'app-root',
  template: `
    <h1>Parent [{{ service.method() }}]</h1>
  `
})
export class AppComponent {

  service!: TestService;

  constructor() {
    this.service = inject(TestService);
  }

}

// test.service.ts

import { Injectable } from '@angular/core';

@Injectable({
    providedIn: 'root'
})
export class TestService {
    constructor() {
    }

    method(): string {
        return 'OK';
    }
}
```

Nothing special, but it's enough to check how it works.

Before you start, I would recommend disabling the javascript source maps in your browser;
for me, it was a lot easier to mess around with the code without maps.
If you don't want to do it, I'm going to paste the source code anyway, so no worries.

To investigate `inject()` function I set a breakpoint in constructor of my `AppComponent` and I stepped
into using debugger.
I found this:

```typescript
function inject(token, flags=InjectFlags.Default) {
    return ɵɵinject(token, flags);
}
```

So to check it, we have to follow `ɵɵinject` function.
It's not a very complicated code,
we have a variable that contains current inject implementation and getter/setter for it. 

```typescript
function ɵɵinject(token, flags=InjectFlags.Default) {
    return (getInjectImplementation() || injectInjectorOnly)(resolveForwardRef(token), flags);
}

// ...

/**
 * Current implementation of inject.
 *
 * By default, it is `injectInjectorOnly`, which makes it `Injector`-only aware. It can be changed
 * to `directiveInject`, which brings in the `NodeInjector` system of ivy. It is designed this
 * way for two reasons:
 *  1. `Injector` should not depend on ivy logic.
 *  2. To maintain tree shake-ability we don't want to bring in unnecessary code.
 */

let _injectImplementation;

function getInjectImplementation() {
    return _injectImplementation;
}

/**
 * Sets the current inject implementation.
 */
function setInjectImplementation(impl) {
    const previous = _injectImplementation;
    _injectImplementation = impl;
    return previous;
}
```

In my case I got the `ɵɵdirectiveInject` implementation,
and it looks ok, because component in Angular extends directive.
This function is very important, so please try to understand it.

```typescript
function ɵɵdirectiveInject(token, flags=InjectFlags.Default) {
    const lView = getLView();
    // Fall back to inject() if view hasn't been created. This situation can happen in tests
    // if inject utilities are used before bootstrapping.

    if (lView === null) {
        // Verify that we will not get into infinite loop.
        ngDevMode && assertInjectImplementationNotEqual(ɵɵdirectiveInject);
        return ɵɵinject(token, flags);
    }

    const tNode = getCurrentTNode();
    return getOrCreateInjectable(tNode, lView, resolveForwardRef(token), flags);
}
```

First, `ɵɵdirectiveInject` reach for two values `lView` and `tNode`.
These properties are necessary to execute `getOrCreateInjectable` function.
Both values are strongly connected the currently processed view. 
It creates a context for DI, so Angular knows in what place in view tree we are trying to inject something.
And both `getLView()` and `getCurrentTNode()` functions are quite simple.
Both of them get the necessary values from `instructionState.lFrame.`,
and this value is set by `enterView()` function
you might know from my previous post about differences between `ApplicationRef.tick()` and `ChangeDetectorRef.detectChanges()`.

```typescript
function getLView() {
    return instructionState.lFrame.lView;
}

// ...

function getCurrentTNode() {
    let currentTNode = getCurrentTNodePlaceholderOk();

    while (currentTNode !== null && currentTNode.type === 64 /* TNodeType.Placeholder */
        ) {
        currentTNode = currentTNode.parent;
    }

    return currentTNode;
}

function getCurrentTNodePlaceholderOk() {
    return instructionState.lFrame.currentTNode;
}
```

Second. The only missing part is the code of `getOrCreateInjectable()` function.

```typescript
function getOrCreateInjectable(tNode, lView, token, flags=InjectFlags.Default, notFoundValue) {
    if (tNode !== null) {
        // If the view or any of its ancestors have an embedded
        // view injector, we have to look it up there first.
        if (lView[FLAGS] & 1024 /* LViewFlags.HasEmbeddedViewInjector */
        ) {
            const embeddedInjectorValue = lookupTokenUsingEmbeddedInjector(tNode, lView, token, flags, NOT_FOUND);

            if (embeddedInjectorValue !== NOT_FOUND) {
                return embeddedInjectorValue;
            }
        }
        // Otherwise try the node injector.

        const value = lookupTokenUsingNodeInjector(tNode, lView, token, flags, NOT_FOUND);

        if (value !== NOT_FOUND) {
            return value;
        }
    }
    // Finally, fall back to the module injector.

    return lookupTokenUsingModuleInjector(lView, token, flags, notFoundValue);
}
```

I bet you know what this code does. It's trying to find a `Injector` instance and get the desired Injectable.

I know that it's crazy and complicated. I decided to use here a huge simplification to describe it:
1. When Angular is building a view, it enters it 
2. This operation sets the current "context"
3. Having that context, Angular is able to access a `Injector` instance
4. From `Injector` we can get the service

## Summary

If you don't know it already, we can use `inject()` function because the "context" has to be set.
Something has to "enter" the component, and when we do other operations like event listeners that part is missing.
That's why for now, `inject()` can only be called in the constructor. 

Personally, event with this limitation, I think that it's a very cool feature.
It simplifies visual style of the code and makes it more readable.
