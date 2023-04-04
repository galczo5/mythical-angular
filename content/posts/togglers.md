---
title: "What you can do with Angular 14 inject function"
date: 2022-06-26
draft: false
---

Since last article, I was wondering what to do with the `inject` function.
For sure, it's quite limited.
We can use it only during construction of the component. 

Today I want to show you something, what I think, might be pretty useful.
Especially if you're working on a bigger project, and you care about the runtime performance. 

## The problem

What's the problem with the `@HostBinding`...?
You need to trigger change detection to apply the effect.
So, when you want to toggle css class on the host element,
you have to execute additional, unnecessary code from the framework. 

It's not very optimised, so we use `Renderer2` instead.
But 'Renderer2' requires more code to achieve the same effect.
It's not very 'developer friendly'
because we always want to keep our code as simple as possible, and as short as possible.

```typescript
// Host binding
@HostBinding('class.background') hostBinding: boolean = false;

// Renderer2

hostBinding: boolean = false;

// ...
// method toggleBackgroundClass
if (this.hostBinding) {
    this.renderer.addClass(this.elementRef.nativeElement, 'background');
} else {
    this.renderer.removeClass(this.elementRef.nativeElement, 'background');
}
```

It's very easy to understand just using the code from above.
I don't like to repeat these conditions over and over.
It makes my code less readable.
On the other hand, I prefer to avoid unnecessary actions in runtime.

## How `inject()` can help to solve this problem

I stared my idea with creating a type of 'toggler'.
You may notice a little of inspiration from the React hooks.
My type contains two methods, one to return the current state (disabled/enabled) and method to change the current state.

```typescript
export type NgxToggler = {
  getState(): boolean;
  toggle: () => boolean;
};

/**
 * In react it's like:
 * const [enabled, setEnabled] = useState(false);
 */
```

Now it's the time to create implementation of the togglers.
You may ask me how I'm going to encapsulate `Renderer2` in `NgxToggler`.
It's quite simple, I'm going to use a factory function pattern.

Let's start with the css class equivalent to `@HostBinding`.

```typescript
import {ElementRef, inject, Renderer2} from "@angular/core";
import {NgxToggler} from "./ngx-toggler";

export type ngxCssClassTogglerOpts = {
    defaultValue: boolean
};

export function ngxCssClassToggler(cssClass: string, 
                                   opts?: Partial<ngxCssClassTogglerOpts>): NgxToggler {

    const renderer = inject(Renderer2);
    const elementRef = inject(ElementRef);

    const element = elementRef.nativeElement;
    let enabled = !!opts?.defaultValue;

    if (enabled) {
        renderer.addClass(element, cssClass);
    }

    return {
        getState: () => {
            return enabled;
        },
        toggle: () => {
            enabled = !enabled;

            if (enabled) {
                renderer.addClass(element, cssClass);
            } else {
                renderer.removeClass(element, cssClass);
            }

            return enabled;
        }
    };
}
```

I'm using `inject()` to get the `Renderer2` and `ElementRef`,
later I'm using them to return the instance of `NgxToggler`.
I think that the code is quite simple and there's no need to explain it.

How to use it? Simple!

```typescript
import {Component} from '@angular/core';
import {ngxCssClassToggler} from "./ngx-css-class-toggler";

@Component({
  selector: 'app-root',
  template: `
    <button (click)="background.toggle()">Toggle background css class</button>
  `
})
export class AppComponent {
  background = ngxCssClassToggler('background');
}
```

It's simple to use.
It's reusable.
It was implemented using the `Renderer2` so there's no need to trigger the change detection.
It can be used because my `ngxCssClassToggler` is executed during the creation of a component.

## Summary

Personally, I think that `inject()` might change the way we implement utils.
It's quite new, so we have to explore the possibilities.
Probably, not every solution will be perfect or even good,
and for now, I would avoid using it for a very complicated code.

`ngxCssClassToggler` in my opinion is a great example of how much fun and useful `inject()` might be.

PS. You can also use the same idea to create togglers for styles, attrs etc.
