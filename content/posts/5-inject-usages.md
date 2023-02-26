---
title: "I'm preparing myself for the lightning talk, so
here you have 5 cool usages of inject function in Angular"
date: 2023-02-25
draft: false
---

Spring conferences are coming.
It's the perfect time to prepare a lighting talk subject.
In this article I will describe the code from one of my subjects: 5 cool usages of `inject()` function from Angular.

Code, unit tests, and a simple app for manual testing you can find in the repository: [github.com/galczo5/ng-inject-lightning-talk](https://github.com/galczo5/ng-inject-lightning-talk).

![tick](/images/inject-examples.png)

## Usage 1: Preventing memory leaks with `useOnDestroy`

There are a lot of different ways of preventing observable memory leaks in components.
The first and the best one is to avoid subscriptions in component typescript code
and move it to the template using `async` or `push` pipe.

If it's not possible for any reason you can use `useOnDestroy` function.

```typescript
import {ChangeDetectorRef, inject, ViewRef} from "@angular/core";
import {Subject} from "rxjs";

export function useOnDestroy() {
  const onDestroy$ = new Subject<void>();
  const viewRef = inject(ChangeDetectorRef) as ViewRef;

  viewRef.onDestroy(() => {
    onDestroy$.next(void 0);
    onDestroy$.complete();
  })

  return onDestroy$;
}

```

It's quite simple. I'm injecting `ChangeDetectorRef` which in components is implemented by `ViewRef`.
This cast is not the perfect solution,
but there is no other (as far as I know, let me know if there is) way of injecting `ViewRef` into the component.

Having `ViewRef` allows me to provide a callback that will be executed on component destroy.

How to use it? Quite simple:

```typescript
import {Component} from '@angular/core';
import {interval, takeUntil} from "rxjs";
import {useOnDestroy} from "../util/useOnDestroy";

@Component({
  selector: 'app-root',
  template: `
  `
})
export class AppComponent {

  intervalSub = interval(1000)
    .pipe(
      takeUntil(useOnDestroy())
    )
    .subscribe(i => {
      // some stuff
    });
}

// OR

@Component({
  selector: 'app-root',
  template: `
  `
})
export class AppComponent {

  // You can reuse it
  onDestroy$ = useOnDestroy();
  
  intervalSub = interval(1000)
    .pipe(
      takeUntil(this.onDestroy$)
    )
    .subscribe(i => {
      // some stuff
    });
}
```

I've tested it manually and with unit tests, which you can find below.

```typescript
import {TestBed} from "@angular/core/testing";
import {Component} from "@angular/core";
import {Subject, takeUntil} from "rxjs";
import {useOnDestroy} from "./useOnDestroy";
import {CommonModule} from "@angular/common";

@Component({
  selector: 'app-test',
  template: ''
})
class TestComponent {
  private readonly onDestroy$ = useOnDestroy();

  stream$ = new Subject()
    .pipe(
      takeUntil(this.onDestroy$)
    );
}

@Component({
  selector: 'app-test',
  template: ''
})
class ShouldNotWorkTestComponent {
  stream$ = new Subject();
}

describe('useOnDestroy', () => {

  it('should close the stream when component is destroyed', done => {

    TestBed.configureTestingModule({
        imports: [CommonModule],
        declarations: [TestComponent]
      }
    ).compileComponents();

    const fixture = TestBed.createComponent<TestComponent>(TestComponent);

    fixture.componentInstance.stream$
      .subscribe({
        complete: () => {
          expect(true).toBeTrue();
          done();
        }
      })

    fixture.destroy();
  });

  it('should not close the stream when component is destroyed and there is useOnDestroy', done => {

    TestBed.configureTestingModule({
      imports: [CommonModule],
      declarations: [ShouldNotWorkTestComponent]
    }).compileComponents();

    const fixture = TestBed.createComponent<ShouldNotWorkTestComponent>(ShouldNotWorkTestComponent);

    fixture.componentInstance.stream$
      .subscribe({
        complete: () => expect(true).toBeFalse()
      });

    fixture.destroy();
    done();
  });

});
```

## Usage 2: Improve performance with `useHostBinding`

The standard way of adding a class to the component is to use the `@HostBinding` decoration.
I have only one problem with this method. 
It requires running a change detection cycle to apply changes to the component.

If you work with disabled NgZones, it's a problem.
What's more, I don't like the idea of running change detection to add a CSS class.

Well-known technique to handle this problem is to use the `Renderer2` instance.
This method can be simply improved with the `inject()` function.

```typescript
import {ElementRef, inject, Renderer2} from "@angular/core";

export function useHostBinding(className: string, enabledByDefault: boolean) {
  const renderer2 = inject(Renderer2);
  const nativeElement = inject(ElementRef).nativeElement;

  let value = enabledByDefault;
  
  if (value) {
    renderer2.addClass(nativeElement, className);
  }

  return {
    set(newValue: boolean) {
      value = newValue;

      if (value) {
        renderer2.addClass(nativeElement, className);
      } else {
        renderer2.removeClass(nativeElement, className);
      }

    },
    get() {
      return value;
    }
  }
}
```

The function injects `Renderer2`, and `ElementRef` and uses them to add or remove CSS class to the component host element.
The result of the function contains an object
that can return the current state and a setter function that can be used to set a new state.

Does it look like a React hook...? YES. I found it extremely useful.
Usage is simple, the DX of using it is fine and it improves the performance of my app in many cases.

```typescript
import {Component, DoCheck, NgZone, OnInit, ViewEncapsulation} from '@angular/core';
import {useHostBinding} from "../../../util/useHostBinding";
import {interval, takeUntil} from "rxjs";
import {useOnDestroy} from "../../../util/useOnDestroy";

@Component({
  selector: 'app-use-host-binding',
  template: `
    <div style="display: flex; height: 500px; width: 100%; align-items: center; justify-content: center;">
      This block is changing color and no change detection is triggered
    </div>
  `,
  styles: [`
    app-use-host-binding {
      display: flex;
      border: 1px dashed gray;
    }

    .red {
      background: red;
    }
  `],
  encapsulation: ViewEncapsulation.None
})
export class UseHostBindingComponent implements OnInit, DoCheck {

  background = useHostBinding('red', false);

  private readonly onDestroy$ = useOnDestroy();

  constructor(private readonly ngZone: NgZone) {
  }

  ngDoCheck(): void {
    console.log('DoCheck called!');
  }

  ngOnInit(): void {
    this.ngZone.runOutsideAngular(() => {
      interval(1000)
        .pipe(takeUntil(this.onDestroy$))
        .subscribe(() => {
          const oldValue = this.background.get();
          this.background.set(!oldValue);
        });
    });
  }

}
```

If you're looking for unit tests for that function, you'll find it in my [repository](https://github.com/galczo5/ng-inject-lightning-talk).

## Usage 3: Stream-based `HostListener`

```typescript
import {ElementRef, inject} from "@angular/core";
import {fromEvent} from "rxjs";

export function useHostListen<T extends Event>(eventName: string) {
  const nativeElement = inject(ElementRef).nativeElement;
  return fromEvent<T>(nativeElement, eventName);
}
```

This one is extremely simple.
I'm injecting `ElementRef` and I'm using `fromEvent` to create a stream from an event listener.

Why do I like to use it instead of `@HostListener`?
First of all, I don't need to create a method which is not used directly anywhere.

```typescript
@Directive({selector: 'button[counting]'})
class CountClicks {
  numberOfClicks = 0;

  @HostListener('click', ['$event.target'])
  onClick(btn) {
    console.log('button', btn, 'number of clicks:', this.numberOfClicks++);
  }
}
```

For example, in this case, [Angular docs](https://angular.io/api/core/HostListener),
method `onClick` is not called anywhere.
The same code using `useHostListen` should look like this:

```typescript
@Directive({selector: 'button[counting]'})
class CountClicks {
  numberOfClicks = 0;
  
  constructor() {
    useHostListen('click')
      .pipe(takeUntil(useOnDestroy()))
      .subscribe((event: MouseEvent) => {
        console.log('button', event.target, 'number of clicks:', this.numberOfClicks++);
      });
  }
}
```

In addition, because it's stream-based, you can take advantage of it and use operators to manipulate your data flow.

```typescript
@Directive({selector: 'complex'})
class ComplexExample {
  constructor(private readonly httpClient: HttpClient) {
    combineLatest(
      useHostListen('click'),
      useHostListen('mouseenter')
    )
      .pipe(
        switchMap(event => this.httpClient.get('url-example')),
        takeUntil(useOnDestroy())
      )
      .subscribe((event: MouseEvent) => {
        console.log('It is so easy to use');
      });
  }
}
```

## Usage 4: Stream-based `HostListener`, but runs outside Angular zone

It's very similar to the previous one.
The only difference is that it runs outside the Angular zone,
so events handled with this method won't trigger unnecessary change detection cycles.

```typescript
import {ElementRef, inject, NgZone} from "@angular/core";
import {fromEvent, Subject, takeUntil} from "rxjs";
import {useOnDestroy} from "./useOnDestroy";

export function useZonelessHostListen<T extends Event>(eventName: string) {
  const nativeElement = inject(ElementRef).nativeElement;
  const ngZone = inject(NgZone);

  const events$ = new Subject<T>();

  ngZone.runOutsideAngular(() => {
    fromEvent<T>(nativeElement, eventName)
      .pipe(
        takeUntil(useOnDestroy())
      )
      .subscribe(value =>
        events$.next(value)
      );
  })

  return events$.asObservable();
}
```

It might be useful when you need to handle `mousemove` or `mouseover` events.

## Usage 5: Observable `OnChanges` method

The final one is the most complicated today.
Technically it's not using inject,
but it looks similar to the previous `useOnDestroy`,
`useHostBinding`, `useHostListen`, and `useZonelessHostListen` functions.
Its main strength is in synergy with other functions.

I named that function `useOnChanges`. The main goal is to create streams from `ngOnChanges` method.

```typescript
import {OnChanges, SimpleChanges} from "@angular/core";
import {Subject, takeUntil} from "rxjs";
import {useOnDestroy} from "./useOnDestroy";

export function useOnChanges<T extends OnChanges>(component: T, ...input: Array<keyof T>) {
  let originalNgOnChanges = component.ngOnChanges;

  const stream$ = new Subject<void>()
  const wrapper = (changes: SimpleChanges) => {
    const anyChanged = (input as Array<string>)
      .some(i => !!changes[i]);

    if (anyChanged) {
      stream$.next(void 0);
    }

    originalNgOnChanges(changes);
  }

  component.ngOnChanges = wrapper;

  return stream$
    .pipe(
      takeUntil(useOnDestroy())
    );
}
```

The function copies the reference to the original `ngOnChanges` method, then wraps it with additional behavior.
That allows us to emit a value on the result stream.
The next function call will wrap it again, and so on.

I think that it's interesting because of its usage.
For example, let's assume that we have to create a component that will get the user id on input.
After each id change component have to get user details from the HTTP client.
If the user is active, the background color should be green, if not, red.
In the end, user details should be passed to view.

```typescript
@Component({
  selector: 'app-root',
  template: `
    {{ userDetails$ | async }}
  `
})
export class AppComponent implements OnChanges {
  @Input()
  userId!: string;

  private readonly background = useHostBinding('valid', false);

  userDetails$ = combineLatest(of(inject(HttpClient)), useOnChanges(this, 'userId'))
    .pipe(
      switchMap(([httpClient]: [HttpClient]) =>
        httpClient.get('https://catfact.ninja/fact?id=' + this.userId)
      ),
      tap(response => this.background.get(response.isValid))
    );

  ngOnChanges(): void {
  }
}

// How it looks like without useOnChanges

@Component({
  selector: 'app-root',
  template: `
    {{ userDetails }}
  `
})
export class AppComponent implements OnChanges, OnDestroy {
  @Input()
  userId!: string;

  @HostBinding('class.valid')
  background = false;

  userDetails = {};

  private readonly onDestroy$ = new Subject<void>();

  constructor(private readonly httpClient: HttpClient) {
  }

  ngOnDestroy(): void {
    this.onDestroy$.next(void 0);
    this.onDestroy$.complete();
  }

  ngOnChanges(changes: SimpleChanges): void {
    // This one does not suggest names of the input. useOnChanges does
    if (changes.userId) {
      this.fetchUserDetails(this.userId);
    }
  }

  private fetchUserDetails(id: string): void {
    this.httpClient.get('https://catfact.ninja/fact?id=' + id)
      .pipe(
        takeUntil(this.onDestroy$)
      )
      .subscribe(data => {
        this.userDetails = data;
        this.background = data.isValid;
      })
  }
}
```

The readability of the code that uses `useOnChanges` is of course strongly connected to the experience in working with streams.
Some developers may be confused about it, so I can't fully recommend this approach to anyone.

Still, it's interesting.

## Summary

I think that the first four functions (`useOnDestroy`,
`useHostBinding`, `useHostListen`, and `useZonelessHostListen`)
are straightforward and you may find a lot of usage for them.

The last one is quite complicated.
It allows you to change your mindset from a "lifecycle approach" to a more "streams/reactive approach".
I like to think that input is a stream, and at some point, you need to convert a value to observable.
You may pass these observables to component children, but it's a subject for the next article.

![tick](/images/sad.png)

I have a homework for you.
The idea is to leave you with a challenge that you can solve after reading this article.
The goal to achieve is to let you write some code and better experience the magic of `inject()` based functions.

I will post the usage of my imagined fuctions and I will let you to think about them.
After few weeks I will post my solutions. 

PS. Don't be afraid to change my usages if you think that you want to create a better solution.

```typescript
// Example 1
// Template: Table of cat facts
@Component({ /* ... */ })
export class Component {
    catFacts$ = useHttpResource('GET', 'https://catfact.ninja/facts');
}

// Example 2
// Template: Table of cat facts
@Component({ /* ... */ })
export class Component {
    
    @Input()
    id!: string;
    
    catFacts$ = useOnInit(this, "id")
        .pipe(
            useCatFacts(), // Pass id to useCatFacts: https://catfact.ninja/facts/${id}
            useMessageErrorHandler() // Create dynamically component that will display readable error
        );
}

// Example 3
@Component({ /* ... */ })
export class Component {

    private readonly navigateToCatFacts = useNavigation("/cat/facts");
    
    onClick() {
        this.navigateToCatFacts();
    }
}

```
