---
title: "Component styles cleanup in Angular 17"
date: 2024-04-02
draft: false
---

Hello everyone!

I noticed interesting behavior that hasn't been in the release notes. In this article I will show you what is it, and why I think it's a quite interesting change.

## Styles cleanup problem

So, I noticed that Angular was not removing styles after removing components that were dynamically created. I mean, once you create a dynamic component (you don't even need to attach it to the DOM), Angular will add styles and then **not remove it when a component is destroyed.**

It's very easy to check and test.

## The test

As always, all code is available to you on my GitHub. Look for [experiment-styles-cleanup repository.](https://github.com/galczo5/experiment-styles-cleanup)

I'm starting my test with a simple component that does nothing but add styles to the application.

```typescript
import {Component, ViewEncapsulation} from '@angular/core';

@Component({
  selector: 'app-dynamic',
  template: '',
  encapsulation: ViewEncapsulation.None,
  styles: [`
    .test {
      background: red;
      color: white;
      border: 4px dashed black;
      font-weight: bold;
    }
  `]
})
export class DynamicComponent {
}
```

As you can see, it's fairly simple. No template, no logic, just styles. What is important is that I've set encapsulation to `ViewEncapsulation.None` so these styles can be applied outside the component.

Then, it's time to modify the root `AppComponent.`

```typescript
import {Component, ComponentFactoryResolver, inject, Injector, OnDestroy} from '@angular/core';
import {DynamicComponent} from "./dynamic.component";

@Component({
  selector: 'app-root',
  template: `
    <div class="test">
      Styles applied when this box is red
    </div>

    <button (click)="createComponent()">Click me!</button>
  `
})
export class AppComponent {
  private readonly componentFactoryResolver = inject(ComponentFactoryResolver);
  private readonly injector = inject(Injector);

  createComponent(): void {
    const factory = this.componentFactoryResolver.resolveComponentFactory(DynamicComponent);
    const componentRef = factory.create(this.injector);

    setTimeout(() => componentRef.destroy(), 5000);
  }
}
```

In the `AppComponent`, we have two things.

The first is `div` with `test` class. It will be styled when Angular adds styles from `DynamicComponent.` That's why I've set encapsulation to `ViewEncapsulation.None`.

The second, we have a button that, on click, will use `ComponentFactoryResolver` to dynamically create `DynamicComponent,` then after 5 seconds, it will destroy it. I know, there's a memory leak in `setTimeout`, but it's only a test, ok?

The whole test is straightforward. Click on the button, and the `div` should be styled.

![trust-the-code](/images/style-applied.png)

Wait for 5 seconds, and there are two possible endings:

1. `div` has styles - it means that **styles were not removed with the component**
2. `div` has no styles - it means Angular is doing a cleanup after component removal

### Angular 15 & 16

As you probably expected, in Angular 15 and 16, styles **are not removed with components.** You can try it yourself using my code from [experiment-styles-cleanup repository](https://github.com/galczo5/experiment-styles-cleanup), branches [main](https://github.com/galczo5/experiment-styles-cleanup) and [v16](https://github.com/galczo5/experiment-styles-cleanup/tree/v16).

![trust-the-code](/images/style-applied-v15.png)

PS. If you don't have time to test it on your machine, I deployed the code via GitHub Pages. Here are the links:
- [galczo5.github.io/experiment-styles-cleanup/v15](https://galczo5.github.io/experiment-styles-cleanup/v15/)
- [galczo5.github.io/experiment-styles-cleanup/v16](https://galczo5.github.io/experiment-styles-cleanup/v16/)

### Angular 17

After the upgrade to Angular 17, the behavior has changed. Now **styles are removed with components.**

Check out `v17` branch in my repository or use [galczo5.github.io/experiment-styles-cleanup/v17](https://galczo5.github.io/experiment-styles-cleanup/v17/) application.

![trust-the-code](/images/style-applied-v17.png)

## How does it work with non-dynamic components?

You may say that this case is a border case, and it's not important because we don't create a lot of dynamic components.

Let's check how it behaves with a standard `ngIf` statement.

To check it, I had to apply the necessary changes in the code of `AppComponent`.

```typescript
import {Component, ComponentFactoryResolver, inject, Injector, OnDestroy} from '@angular/core';
import {DynamicComponent} from "./dynamic.component";

@Component({
  selector: 'app-root',
  template: `
    <div class="test">
      Styles applied when this box is red
    </div>

    <app-dynamic *ngIf="visible"/>

    <button (click)="createComponent()">Click me!</button>
  `
})
export class AppComponent {
  visible = false;

  createComponent(): void {
    this.visible = true;
    setTimeout(() => this.visible = false, 5000);
  }
}
```

The results are not surprising.

Angular 15 & 16 - **styles are not removed**
- Branch `v15-ngIf` and [galczo5.github.io/experiment-styles-cleanup/v15-ngIf/](https://galczo5.github.io/experiment-styles-cleanup/v15-ngIf/)
- Branch `v16-ngIf` and [galczo5.github.io/experiment-styles-cleanup/v16-ngIf/](https://galczo5.github.io/experiment-styles-cleanup/v16-ngIf/)

Angular 17 - **styles are removed after a component is destroyed**
- Branch `17-ngIf` and [galczo5.github.io/experiment-styles-cleanup/v17-ngIf/](https://galczo5.github.io/experiment-styles-cleanup/v17-ngIf/)

So, it works exactly the same as for dynamic components. I'm not sure how heavy it is to have additional not-removed styles for the browser, but I assume that fewer styles should be easier to process. Maybe next time, I'll measure it.

## What about encapsulated styles?

All my tests used `ViewEncapsulation.None` to visually show how it works. To check how it works with default emulated encapsulation, I used dev tools to check DOM after component removal. Observe what is happening in `<head>` part of your document.

The results for me were:
- Angular 15 & 16 - **styles are not removed**
- Angular 17 - **styles are removed**

It works the same for the default encapsulation.

## Summary

To be honest, I was a bit surprised when I realized that Angular does not remove styles by default when you destroy a component. If you had asked me about it a few months ago, I would have bet that Angular is removing them.

I'm sure that the change introduced in Angular 17 is more intuitive for every dev.

To sum up, my tests show that before Angular 17, **all styles added by components were not removed after component was destroyed.** No matter if it's a dynamic component or conditionally added component using `ngIf`. Angular 17 has changed this behaviour and now unnecessary styles are cleaned after components.  
