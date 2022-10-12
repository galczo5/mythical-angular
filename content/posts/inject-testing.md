---
title: "How to create unit tests for code using `inject` without `TestBed`"
date: 2022-10-12
draft: false
---

`inject` function introduced in Angular 14, even with limitations is a very cool feature. But, where's light there's dark. When we inject everything into a constructor of the class, it's easy to test it when you're not using `TestBed`. After using inject it might be a bit complicated.

Luckily for us, it's not impossible. What's more, it's very easy to make code with `inject` testable.

## Repository

As always, if you need my code, you'll find it in the repository on GitHub:
[github.com/galczo5/testable-inject](https://github.com/galczo5/testable-inject).
To make testing my code easier in your cases, I published it on NPM.
Here is the link: [npmjs.com/package/testable-inject](https://www.npmjs.com/package/testable-inject).

## Problem
The main problem with `inject` is that, it's a function.
Many test runners provide solutions to mock functions, etc.
Personally, I like my code to be agnostic from the test runners,
so I can migrate it from Karma to Jest or maybe even something different.

To solve this problem, we can just wrap the `inject` function.
Wrapper should have some kind of context.
For the developer experience reasons, I will make it static.

```typescript
import {isDevMode, inject as angularInject, ProviderToken} from "@angular/core";

export class Inject {

  private static map: Map<ProviderToken<any>, any> = new Map<ProviderToken<unknown>, unknown>();

  static mock<T>(token: ProviderToken<T>, value: T) {
    this.map.set(token, value);
  }

  static get<T>(token: ProviderToken<T>): T {
    return this.map.get(token);
  }

  static has<T>(token: ProviderToken<T>): boolean {
    return this.map.has(token);
  }

}

export function inject<T>(token: ProviderToken<T>): T {
  if (isDevMode() && Inject.has<T>(token)) {
      return Inject.get(token);
  }

  return angularInject<T>(token);
}
```

`Inject` class is my context here. With this class, it is easy to mock anything you want. In addition to the context, I've added my `inject` function that has the same API that original function from the `@angular/core` package. This function is my wrapper. From the outside, all you have to do is to change import in your code, and it becomes testable! 

BTW. It's using the mocked values only in dev mode, so it shouldn't affect production build. Even if it is affecting, the difference will be close to zero milliseconds.

## How to use

Let's say that I have a very simple component to test.

```typescript
import {Component, ElementRef, Renderer2} from "@angular/core";
import {inject} from "testable-inject";

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {

  constructor(elementRef: ElementRef) {
    const renderer = inject(Renderer2);
    renderer.addClass(elementRef.nativeElement, 'class');
  }

}
```

The only thing to test here is that `renderer.addClass` call, to check if class was added, or not.
Notice that `inject` reference comes from the `testable-inject` package, not from `@angular/core`. 

It's only an example, so I decided not to waste any time here and unit test code is simplified.

```typescript
import {AppComponent} from "./app.component";
import {Inject} from "testable-inject";
import {Renderer2} from "@angular/core";

describe('AppComponent', () => {

  it('Should mock inject', () => {

    let itWorked = false;

    Inject.mock(Renderer2, {
      addClass() {
        itWorked = true;
      }
    } as unknown as Renderer2)

    new AppComponent({ nativeElement: null });

    expect(itWorked).toBeTruthy();
  });

});
```

## Conclusion

You can (and you should) test code with `inject` function calls.
It's easy and looks the same as Angular function call.

All code for this is published in this article.
The same code you'll find in the repository and in the NPM package.
