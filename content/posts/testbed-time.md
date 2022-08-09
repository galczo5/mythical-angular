---
title: "Short: I had some time, so I checked how much time-consuming is TesBed"
date: 2022-08-09
draft: false
---

This is going to be short.
I cannot stop
thinking about my last results from the [previous article](https://mythical-angular.dev/posts/ngc-esbuild-jest/).
I was able to use esbuild instead of ts-jest, and it was still slow.

It looked like using TestBed is really time-consuming. So I decided to check it, and I'm ready to show you the results.

## The test

For testing, I used the same repository. 
You can find it on my GitHub: [ng-esbuild-unit-tests](https://github.com/galczo5/ng-esbuild-unit-tests).

The test is simple. Each `it` contains a loop and a test.
Both tests are testing the same case,
tests are checking if `Output` of the component emitted a value after calling `createItem` method.

```typescript
import {TestBed} from '@angular/core/testing';

import {CreateFormComponent} from './create-form.component';
import {take} from "rxjs";

const numberOfIterations = 10000

describe('CreateFormComponent', () => {

    it('TestBed - should emit value if value is not empty', async () => {
      for (let i = 0; i < numberOfIterations; i++) {
        await TestBed.configureTestingModule({ declarations: [CreateFormComponent] }).compileComponents();
        const fixture = TestBed.createComponent(CreateFormComponent);
        const component = fixture.componentInstance;
        fixture.detectChanges();

        const value = 'Test';

        const promise = component.itemCreated
          .pipe(take(1))
          .toPromise();

        component.createItem(value);

        const result = await promise;
        expect(result).toEqual(value);
        await TestBed.resetTestingModule();
      }
    });

    it('No TestBed - should emit value if value is not empty', async () => {
      for (let i = 0; i < numberOfIterations; i++) {
        const component = new CreateFormComponent();
        const value = 'Test';
        const promise = component.itemCreated
          .pipe(take(1))
          .toPromise();

        component.createItem(value);

        const result = await promise;
        expect(result).toEqual(value);
      }
    });

});
```

## Results

I decided to run tests a few times. Each time, I increased the number of iterations in the loop inside the tests. 

{{< rawhtml >}}
<div style="border: 1px solid lightgray;">
{{< /rawhtml >}}
![trust-the-code](/images/testbed-chart.png)
{{< rawhtml >}}
</div>
{{< /rawhtml >}}


| Number of iterations | TestBed \[ms\] | NoTestBed \[ms\] | TestBed time per iteration \[ms\] | No TestBed time per iteration \[ms\] | How many times tests without TestBed were faster that tests with TestBed |
|----------------------|----------------|------------------|-----------------------------------|--------------------------------------|--------------------------------------------------------------------------|
| 10                   | 47             | 1                | 4,7000                            | 0,1000                               | 47                                                                       |
| 100                  | 147            | 6                | 1,4700                            | 0,0600                               | 25                                                                       |
| 250                  | 304            | 13               | 1,2160                            | 0,0520                               | 23                                                                       |
| 500                  | 523            | 19               | 1,0460                            | 0,0380                               | 28                                                                       |
| 1000                 | 915            | 32               | 0,9150                            | 0,0320                               | 29                                                                       |
| 2000                 | 1586           | 62               | 0,7930                            | 0,0310                               | 26                                                                       |
| 5000                 | 3308           | 145              | 0,6616                            | 0,0290                               | 23                                                                       |
| 10000                | 6028           | 261              | 0,6028                            | 0,0261                               | 23                                                                       |

## Summary

I've expected before doing the test that the time of the test will be linear.
It's a simple loop, so it's natural.
What is fascinating is that when I had a small number of iterations,
the difference between my two tests was significantly greater.
When I had a lot of iterations, the difference was still visible, but time per iteration was shrinking.

So why not to verify it...?

ONE LAST TEST: 100000 ITERATIONS!

| Number of iterations | TestBed \[ms\] | NoTestBed \[ms\] | TestBed time per iteration \[ms\] | No TestBed time per iteration \[ms\] | How many times tests without TestBed were faster that tests with TestBed |
|----------------------|----------------|------------------|-----------------------------------|--------------------------------------|--------------------------------------------------------------------------|
| 100000               | 56573          | 3714             | 0,5657                            | 0,0371                               | 15                                                                       |
