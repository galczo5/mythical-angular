---
title: "Myth: Using set instead ngOnChanges improves the performance"
date: 2022-02-06
draft: false
---

## Introduction

Why using sets instead of `ngOnChanges` is considered as a good practice...?
I'm not sure, but I heard it so many times that I decided to check it.
Thanks to luck, I found the root of this myth.

## The myth

It all started
when I asked a frontend developer about practices
that are helpful with dealing with the Angular performance problems.
One of the things he said was that using sets on inputs is a performance tweak.
After that conversation, few other developers said the same.
So, myth I'm going to verify is "Sets are faster than ngOnChanges".

## Verification

To verify this case, I created a GitHub repo. [Here it is.](https://github.com/galczo5/experiment-setters)
The App contains those routes, one with a component using setter on input.

``` typescript
import { ChangeDetectionStrategy, Component, Input } from '@angular/core';

@Component({
  selector: 'app-setter-counter',
  template: '<p>{{ displayValue }}</p>',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class SetterCounterComponent {

  @Input()
  set value(newValue: number) {
    this.displayValue = newValue
  }

  displayValue: number = 0; 

}
```

The second component is almost the same, only change is that it's using `ngOnChanges` method to set `displayValue`.

``` typescript
import { ChangeDetectionStrategy, Component, Input, OnChanges, SimpleChanges } from '@angular/core';

@Component({
    selector: 'app-on-changes-counter',
    template: '<p>{{ displayValue }}</p>',
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnChangesCounterComponent implements OnChanges {

    @Input()
    value: number = 0;

    displayValue: number = 0;

    ngOnChanges(changes: SimpleChanges): void {
        if (changes.value) {
            this.displayValue = this.value;
        }
    }
}
```

Every component in the whole app is using `ChangeDetectionStrategy.OnPush` strategy.
What's more I decided to provide `noop` zone to minimize the overhead of the framework.

Each test changes the input 100 000 times and triggers the change detection between each change.

``` typescript
import { ChangeDetectionStrategy, ChangeDetectorRef, Component, OnChanges, OnInit, SimpleChanges } from '@angular/core';

@Component({
  selector: 'app-on-changes',
  template: `
    <button (click)="click()">on changes test</button>
    <p>Time: {{time}}</p>
    <app-on-changes-counter [value]="value"></app-on-changes-counter>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnChangesComponent {

  value: number = 0;
  time: number = 0;

  constructor(private cdr: ChangeDetectorRef) {}

  click(): void {
    this.value = 0;

    const start = performance.now();
    for (let i = 0; i < 100_000; i++) {
      this.value = i;
      this.cdr.detectChanges();
    }

    this.time = performance.now() - start;
    this.cdr.detectChanges();
  }

}
```

To have trustful results, I repeated the test 10 times.
You can try it at home, but if you want, below I presented my results.

| No  | OnChanges | Setter   |
|-----|-----------|----------|
| 1   | 395 ms    | 474 ms   |
| 2   | 464 ms    | 348 ms   |
| 3   | 424 ms    | 371 ms   |
| 4   | 444 ms    | 393 ms   |
| 5   | 387 ms    | 425 ms   |
| 6   | 445 ms    | 346 ms   |
| 7   | 430 ms    | 374 ms   |
| 8   | 413 ms    | 359 ms   |
| 9   | 409 ms    | 411 ms   |
| 10  | 429 ms    | 356 ms   |
| --- | --------- | ------   |
| MAX | 464 ms    | 474 ms   |
| MIN | 387 ms    | 346 ms   |
| AVG | 424 ms    | 385 ms   |
| MED | 426.5 ms  | 372.5 ms |

As you see, the difference between the results is minimal and may be caused by the environmental conditions.
It's slightly better for setters but if we compare minimum and maximum values, we can agree that there is no difference.

## If there is no difference, why not setters

Using setters has a huge con. To demonstrate it, I'm using these two code snippets.
First, we have a Component with two inputs.
The second input is using value from the first input.
Very simple code, right?

``` typescript
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    {{ _normalizedFullName }}
  `
})
export class AppComponent {

  @Input()
  set name(value: string) {
    this._name = value;
  }

  @Input()
  set surname(value: string) {
    this._surname = value;
    this._normalizedFullName = `${this._name.toLocaleUpperCase()} ${this._surname.toLocaleUpperCase()}`
  }

  _name: string;
  _surname: string;
  _normalizedFullName: string;

}
```

The problem occurs when you use the component.
You have to consider the order of executing sets in the class.
It's not going to be executed at the same moment, one set has to be second.

So, if you use this component as in example below, you get an error.

```html
<!-- This one is ok -->
<app-root name="John" surname="Doe"></app-root>

<!-- This one is not ok -->
<app-root surname="Doe" name="John"></app-root>
```

As you may notice, the order of sets is directed by order in a component usage. 
It means that a programmer who is going to use your component has to know its implementation.
This one is very simple, in real-life applications it's very often a lot harder to find where the problem is.
The last thing you want to have is these kinds of problems in your code base.

If you ask me for advice, just don't use setters. Use `ngOnChanges` instead.

## Origin of the myth

Once I asked person who told me again about that performance tip with setters.
I had that luck, to get the source of the myth.
It was this page: [nofluffjobs.com/blog/angular-developer-pytania](https://nofluffjobs.com/blog/angular-developer-pytania/).
It's in Polish, but I translated it for you.

<blockquote>
2. Tips that helps with Angular performance – you should mandatory mention:

- OnPush strategy
- Detaching from Angular Zone
- Using ChangeDetector for manual handling change detection
- Using @Input instead OnChanges
- View lazy loading

Sources:

https://github.com/mgechev/angular-performance-checklist
http://www.angular.love/2017/01/15/angular-2-change-detector-mechanizmy-detekcji-oraz-strategia-onpush/
http://www.angular.love/2018/03/04/angular-i-zone-js/
</blockquote>

It leads us to a very well-known
[mgechev/angular-performance-checklist](https://github.com/mgechev/angular-performance-checklist) repository.

What is interesting, there is a fragment that shows how to use setters to get better performance.
[Detaching the Change Detector](https://github.com/mgechev/angular-performance-checklist#detaching-the-change-detector) shows us a way
how to use native JavaScript to rerender only a part of the page.
This is the clue of that tip. 

If you want to change a css class or a style of an html element, use renderer or native solutions.
It's probably faster thant running the whole rerender of view.
But please don't treat it as a rule.
Sometimes it's better to refresh the whole view.
I'm going to explain it in the future in another post.

## Summary

There is no performance reason to use setters instead of using `ngOnChanges`.
What's more, you can create new, hard to find errors if you create a component with inputs depending on each other.

