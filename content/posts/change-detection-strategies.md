---
title: "Myth: Change Detection Strategies"
date: 2022-02-20T11:17:36+01:00
draft: true
---

Not so long ago I noticed that change detection subject in Angular is very mythical.
For the first sight it looks very simple. But it's not.

Previously I wrote about async pipe and how it affects the change detection. I mentioned that it's not working without NgZones.
NgZones are quite complicated in concept, so I decided to demystify two available strategies for change detection.

## Docs

So, we have two change detection strategies: `Default` and `OnPush`. If you have not specified which one you want to use in your component, it's going to use `Default` one.
Our experiment we should start with the [docs](https://angular.io/api/core/ChangeDetectionStrategy).

![docs](/mythical-angular/images/cd-docs.png)

As you can see the documentation is not helpful at all. Even if you follow the [Change detection usage](https://angular.io/api/core/ChangeDetectorRef#usage-notes) links still it's not enough. In this case it's good to try using external sources to learn how it works.

## Testing app

I've created for you an app that will let you try everything yourself. It's a very simple app. Basically there are two branches of almost identical components, one branch with `Default` strategy and the second one with the `OnPush`.  
Every branch contains three components: Parent and two children.

![docs](/mythical-angular/images/cd-app.png)

Every component of my app implements `OnInit`, `OnChanges` and `DoCheck` hooks to log what is called by Angular.
Today we are not using it to do anything, I just want to know which component was checked etc.

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

In addition, I've added a button to disable/enable NgZones. NgZones is not a part of change detections strategies, but it's strongly connected, and I decided that it's worth to show it.

