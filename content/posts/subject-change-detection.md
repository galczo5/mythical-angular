---
title: "Angular Myth: Observables triggers change detection"
date: 2023-01-13
draft: false
---

One of the most common myths about Angular is that observables are triggering change detection cycle.
It's so easy to prove that this is going to be probably one of the shortest on this blog. 

## Test

We need some code to show how it works.
Today I'm starting with a new Angular project and only one component.
I leave default settings, event NgZones are enabled.
YOLO.

The app, as always,
is available for you in [galczo5/experiment-subject-change-detection](https://github.com/galczo5/experiment-subject-change-detection) repo,
and it's hosted using GitHub pages [galczo5.github.io/experiment-subject-change-detection/](https://galczo5.github.io/experiment-subject-change-detection/).

```typescript
import {Component, OnDestroy, OnInit} from '@angular/core';
import {Subject, Subscription} from "rxjs";

@Component({
  selector: 'app-root',
  template: `
    <h1>{{ time }}</h1>
  `
})
export class AppComponent implements OnInit, OnDestroy {

  time = new Date().toISOString();

  private subscription!: Subscription;

  ngOnInit(): void {
    const subject = new Subject();

    this.subscription = subject
      .subscribe(() => {
        this.time = new Date().toISOString();
        console.log('New value', this.time);
      })

    // @ts-ignore
    window['observable'] = subject;
  }

  ngOnDestroy(): void {
    this.subscription.unsubscribe();
  }
}
```

In the code I'm creating an Observable using `Subject`.
In the subscription I'm updating `time` field which is rendered.
So, if value in observable triggers the change detection cycle,
every time when I execute `window.observable.next()` using Dev Tools of my browser,
it should update the HTML visible for the user.

![window](/images/windowSubject.png)

## Summary

It doesn't refresh the HTML, so this proves that **observables are not triggering the change detection cycle.**

Why and how this myth was created...?
I believe (because I cannot prove that)
that it was created because we use Observables to handle asynchronous actions in our apps.
We create streams to handle promises, events, setTimeouts and setIntervals,
and all those things will punch NgZones to trigger the change detection.  
