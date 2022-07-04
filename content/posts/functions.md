---
title: "Not a myth: Using fields instead of methods in templates is good for the performance"
date: 2022-07-04
draft: false
---

So, this article is a bit different.
This time, I'm describing a very popular technique of optimization in Angular that is actually not a myth.

The whole idea is to use fields instead of methods in templates,
because to get value from a function you have to execute it.
Using methods makes memoization not possible etc.
In the end, do you really have to care about it?

## The test

To test this hypothesis, I've created an example app. It's very simple. Just three components.

Let's start with the starting conditions.
As always all of my components are using `OnPush` change detection strategy, `NoopNgZone` is provided.
I'm doing it to minimize the influence of the framework.

So my app contains one component which can be used to check the performance when a component is using fields.

```typescript
import {ChangeDetectionStrategy, Component} from '@angular/core';

@Component({
  selector: 'app-fields',
  template: `
    <p>
      {{ test }}
    </p>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class FieldsComponent {

  test: string = 'test';

}
```

And almost the same component, but with method instead of field.

```typescript
import {ChangeDetectionStrategy, Component} from '@angular/core';

@Component({
  selector: 'app-functions',
  template: `
    <p>
      {{ test() }}
    </p>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class FunctionsComponent {

  test(): string {
    return 'test';
  }

}
```

As you may expect, the difference in performance in these two components will be very, very small.
What we do to handle this...? Repetitions! 

My tests will render multiple instances of each component, multiple times.
Executing a test like this will give us more comparable results.
Of course, to get the best results, we have to execute the whole test multiple times.

Here is my test component class.

```typescript
import {ChangeDetectionStrategy, ChangeDetectorRef, Component} from '@angular/core';

@Component({
  selector: 'app-root',
  template: `
    <div>
      <button (click)="selectTestType('fields')">Use Fields</button>
      <button (click)="selectTestType('functions')">Use Functions</button>
      <button (click)="test()">Run test (selected {{ testType }})</button>
      <button (click)="reset()">Reset</button>
      <span *ngIf="result">Result {{ result }}</span>
    </div>
    <div *ngIf="testType === 'fields' && visible">
        <app-fields *ngFor="let x of TEST_ARRAY"></app-fields>
    </div>
    <div *ngIf="testType === 'functions' && visible">
      <app-functions *ngFor="let x of TEST_ARRAY"></app-functions>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AppComponent {

  readonly TEST_ARRAY = new Array(1000).map((value, index) => index);
  readonly ITERATIONS = 1000;

  result = 0;
  testType: 'fields' | 'functions' = 'functions';
  visible = false;


  constructor(private readonly changeDetectorRef: ChangeDetectorRef) {
  }

  reset() {
    this.result = 0;
    this.changeDetectorRef.detectChanges();
  }

  selectTestType(type: 'fields' | 'functions') {
    this.testType = type;
    this.changeDetectorRef.detectChanges();
  }

  test() {
    const start = performance.now();
    for (let i = 0; i < this.ITERATIONS; i++) {
      this.visible = !this.visible;
      this.changeDetectorRef.detectChanges();
    }
    const end = performance.now();

    this.result = end - start;
    this.changeDetectorRef.detectChanges();
  }

}
```

If you want to test it at home, follow [this link to my GitHub repository with the code.](https://github.com/galczo5/experiment-angular-functions-fields)

My test will render 1000 component instances 500 times (each iteration will create or destroy components).

## Results

I repeated the test 15 times, before I was ready to agree that the results are trustful.
Everything you can find in the table below.

| Test no. | Fields (ms) | Functions (ms) |
|----------|-------------|---------------|
| 1        | 1708        | 2051          |
| 2        | 1717        | 1727          | 
| 3        | 1692        | 1706          |
| 4        | 1749        | 1706          |
| 5        | 2054        | 1712          |
| 6        | 1854        | 1961          |
| 7        | 1692        | 1684          |
| 8        | 1673        | 1774          |
| 9        | 1703        | 1712          |
| 10       | 1684        | 1696          |
| 11       | 1700        | 1914          |
| 12       | 1732        | 1716          |
| 13       | 1939        | 1673          |
| 14       | 1694        | 1817          |
| 15       | 1703        | 1780          |
| **AVG**  | 1752,93     | 1775,27       |
| **MED**  | 1703        | 1716          |
| **MIN**  | 1673        | 1673          |
| **MAX**  | 2054        | 2051          |

## Summary

In the end, using fields is slightly faster.
Remember that on average there's only 13ms of difference (rendering 1000 components 1000 times).

I'm not sure if sacrificing code readability is worth refactor all of your templates.
After all, you have to introduce a few more fields in class if you want to avoid using functions,
and it may be harder to read and understand.

My example, of course, is pretty basic.
When you have long calculations, you should avoid using functions in templates.
Like in the real world.
There are no best practices you should always follow.
Everything has its context, and it's very important to know the facts before refactoring your code. 
