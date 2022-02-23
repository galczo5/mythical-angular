---
title: "Myth: Change Detection Strategies"
date: 2022-02-20
draft: false
---

Not so long ago I noticed that the change detection subject in Angular is very mythical.
For the first sight, it looks very simple, and it is.

## Docs

So, we have two change detection strategies: `Default` and `OnPush`. If you have not specified which one you want to use in your component, it's going to use `Default` one.
Our experiment we should start with the [docs](https://angular.io/api/core/ChangeDetectionStrategy).

![docs](/mythical-angular/images/cd-docs.png)

As you can see, the documentation is not that easy to base here only on the docs. Even if you follow the [Change detection usage](https://angular.io/api/core/ChangeDetectorRef#usage-notes) links, it's not enough. 
In this case, it's good to try using external sources to learn how it works,
and we all know that it's not the best source of truth.

## Testing app

I've created for you an app that will let you try everything yourself. 
You can find it here: [github.com/galczo5/experiment-change-detection-strategy](https://github.com/galczo5/experiment-change-detection-strategy).
The app is hosted below in the iframe, so it's fully interactive, and you can do every test I did.

{{< rawhtml >}}
<div style="height: 800px; width: 1050px; border: 1px solid lightgray;">
    <iframe style="border: 0; width: 100%; height: 100%;" src="https://galczo5.github.io/experiment-change-detection-strategy/"></iframe>
</div>
{{< /rawhtml >}}

It's a very simple app. 
Basically there are two branches of almost identical components, one branch with `Default` strategy and the second one with the `OnPush`.
Every branch contains three components: Parent and two children.

![docs](/mythical-angular/images/cd-app.png)

Every component of my app implements `OnInit`, `OnChanges` and `DoCheck` hooks to log what is called by Angular.
Today we are not using it to do anything. I just want to know which component was checked etc.

``` typescript
import {
  ChangeDetectionStrategy,
  ChangeDetectorRef,
  Component,
  DoCheck,
  OnChanges,
  OnInit,
  SimpleChanges
} from '@angular/core';
import {getConsoleStyle} from "../consoleStyleFactory";

@Component({
  selector: 'app-on-push-parent',
  template: `
    // ...
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnPushParentComponent implements OnInit, OnChanges, DoCheck {

  /* ... */

  ngOnInit() {
    console.log('%cOnPushParentComponent', getConsoleStyle('green'), 'ngOnInit');
  }

  ngDoCheck() {
    console.log('%cOnPushParentComponent', getConsoleStyle('green'), 'ngDoCheck');
  }

  ngOnChanges(changes: SimpleChanges) {
    console.log('%cOnPushParentComponent', getConsoleStyle('green'), 'ngOnChanges', changes);
  }

  /* ... */
}

```

In addition, I've added a button to disable/enable NgZones. NgZones is not a part of change detection strategies, but it's strongly connected, and I decided that it's worth showing it.

## Facts

### Fact 1. `Default` strategy checks a lot more than `OnPush`

It's easy to test it. Just click on `set value` button or follow the instructions bellow"

1. Click on the button "set value" in Default Strategy parent
2. Value should be displayed below Child 1 in Default Strategy
3. Click on the button "set value" in OnPush Strategy parent
4. Observe value not being displayed below Child 1 in OnPush Strategy

Repeat the test clicking "set new value".

This button is executing very simple code:

```typescript
import {ChangeDetectorRef, Component, DoCheck, OnChanges, OnInit, SimpleChanges} from '@angular/core';
import {getConsoleStyle} from "../consoleStyleFactory";

@Component({
    selector: 'app-default-parent',
    template: `
    // ...
      <button (click)="setValue()">Set value</button>
      <button (click)="setNewValue()">Set new value</button>
    // ...
  `
})
export class DefaultParentComponent implements OnInit, OnChanges, DoCheck {

    obj = { value: 0 };

    /* ... */

    setValue() {
        this.obj.value = new Date().getTime();
    }

    setNewValue() {
        this.obj = { value: new Date().getTime() };
    }

    /* ... */
}
```

In case of `Default` strategy you'll see that value is rendered.
When you repeat that test in `OnPush` branch value will not be rendered.
In both cases in the console, you should get a similar output.

Default:
```text
default-parent.component.ts:44 DefaultParentComponent ngDoCheck
on-push-parent.component.ts:52 OnPushParentComponent ngDoCheck
default-child1.component.ts:30 DefaultChild1Component ngDoCheck
default-child2.component.ts:26 DefaultChild2Component ngDoCheck
```

OnPush:
```text
default-parent.component.ts:44 DefaultParentComponent ngDoCheck
on-push-parent.component.ts:52 OnPushParentComponent ngDoCheck
default-child1.component.ts:30 DefaultChild1Component ngDoCheck
default-child2.component.ts:26 DefaultChild2Component ngDoCheck
on-push-child1.component.ts:39 OnPushChild1Component ngDoCheck
on-push-child2.component.ts:35 OnPushChild2Component ngDoCheck
```

So if you keep your object immutable, it should work like in method `setNewValue`.
In both strategies value will be rendered.

**This fact shows us why `OnPush` strategy is considered
to be better for performance.** Comparing values recursively is not easy and complex.
Not doing it is great.

### Fact 2. Using `OnPush` reduce the number of components to check

The test is even simpler than the previous one.


1. Open the dev tools of your browser.
2. Clear the console.
3. Click on `console.log` button from `AppComponent`.

On the output you'll see:

```text
app.component.ts:52 Click!
default-parent.component.ts:44 DefaultParentComponent ngDoCheck
on-push-parent.component.ts:52 OnPushParentComponent ngDoCheck
default-child1.component.ts:30 DefaultChild1Component ngDoCheck
default-child2.component.ts:26 DefaultChild2Component ngDoCheck
```

Here we see
that Angular is executing `DoCheck` hook for every component with the `Default` strategy and only for the direct child with `OnPush` strategy.

This button is not changing anything in my view.
There is only a side effect in form of console.log.
In my opinion, there is no reason to check any of the components and Angular will execute some code.
When you use `OnPush` it executes less code.

When you turn off the `NgZone`, it will print only the console.log and no checks will be executed.

Sounds like optimization, right? 

### Fact 3. Change detection is strongly connected to NgZones

This test I'm starting with pushing the `toggle ngzone` button. The header `Noop Zone Provided` should appear.

From this point, I'm repeating the test from the first fact.
In every case, no matter if I click on `set value` or `set new value` there is nothing.
Value is not rendered.
The console is clear.
It looks like the app is broken. 

To fix that issue, we have to execute change detection manually.
In our case we can use `detect changes` button.
After that, it's working again!

Now, you have full control of the change detection in Angular.
When you work with performance, it's very important to have it,
 because you don't need to care about external bottlenecks.

**Important!** Check out the output in the console.
In this case, it's executing a lot less `DoCheck` hooks.
For button in `Default` branch it'll execute only the hooks for components from `Default` subtree.
Analogical for the `OnPush` strategy.

This test proves one very important thing. 
`NgZone` is the mechanism responsible for triggering the change detection cycle.
No matter if you use `Default` or `OnPush`, at the end value was rendered because `zone.js` reacted to click event. 

### Fact 4. Only events from inside the app are triggering change detection

Click on the button from section in the red box bellow apps.

Nothing will happen.
No hook was executed in this test.

Code outside your app will not affect your performance or trigger the change detection, and it's great.

## Summary

I'm not sure why there are a lot of myths about this subject.
I heard just too many very strange answers for a question "what is the difference between change detection strategies in Angular".

At the end, it's pretty simple:
- `Default` - check recursively
- `OnPush` - check only the reference to an object.

Change detection cycle is started by click event, or anything else caught by `NgZone`.

In my opinion, If you care about performance,
you may want to use `OnPush` strategy in every component, and you may consider switching to zone-less approach.
It's just faster when your app is not executing code that is not necessary to render the value in one place.

I think that I proved it in this article. 
