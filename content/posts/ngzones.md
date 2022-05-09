---
title: "Code analysis: NgZone vs NoopNgZone"
date: 2022-05-09
draft: false
---

In previous articles I described how to start creating Angular apps using `noop` zone instead `NgZone`.
The text below describes the technical difference in implementation between `NgZone` and `NoopNgZone`.
In my opinion,
it's quite important to know that difference because it helps to understand how change detections work.

## NgZone

I decided to remove some comments from the code, so it should be easier to read.
Check out the `packages/core/src/zone/ng_zone.ts` file in Angular repository for the full version.
It's really well commented there.

```typescript
export class NgZone {
  readonly hasPendingMacrotasks: boolean = false;
  readonly hasPendingMicrotasks: boolean = false;

  readonly isStable: boolean = true;
  readonly onUnstable: EventEmitter<any> = new EventEmitter(false);

  /**
   * Notifies when there is no more microtasks enqueued in the current VM Turn.
   * This is a hint for Angular to do change detection, which may enqueue more microtasks.
   * For this reason this event can fire multiple times per VM Turn.
   */
  readonly onMicrotaskEmpty: EventEmitter<any> = new EventEmitter(false);
  
  readonly onStable: EventEmitter<any> = new EventEmitter(false);
  readonly onError: EventEmitter<any> = new EventEmitter(false);

  constructor({
    enableLongStackTrace = false,
    shouldCoalesceEventChangeDetection = false,
    shouldCoalesceRunChangeDetection = false
  }) {
    if (typeof Zone == 'undefined') {
      throw new Error(`In this configuration Angular requires Zone.js`);
    }

    Zone.assertZonePatched();
    const self = this as any as NgZonePrivate;
    self._nesting = 0;

    self._outer = self._inner = Zone.current;

    if ((Zone as any)['TaskTrackingZoneSpec']) {
      self._inner = self._inner.fork(new ((Zone as any)['TaskTrackingZoneSpec'] as any));
    }

    if (enableLongStackTrace && (Zone as any)['longStackTraceZoneSpec']) {
      self._inner = self._inner.fork((Zone as any)['longStackTraceZoneSpec']);
    }
    // if shouldCoalesceRunChangeDetection is true, all tasks including event tasks will be
    // coalesced, so shouldCoalesceEventChangeDetection option is not necessary and can be skipped.
    self.shouldCoalesceEventChangeDetection =
        !shouldCoalesceRunChangeDetection && shouldCoalesceEventChangeDetection;
    self.shouldCoalesceRunChangeDetection = shouldCoalesceRunChangeDetection;
    self.lastRequestAnimationFrameId = -1;
    self.nativeRequestAnimationFrame = getNativeRequestAnimationFrame().nativeRequestAnimationFrame;
    forkInnerZoneWithAngularBehavior(self);
  }

  static isInAngularZone(): boolean {
    // Zone needs to be checked, because this method might be called even when NoopNgZone is used.
    return typeof Zone !== 'undefined' && Zone.current.get('isAngularZone') === true;
  }

  static assertInAngularZone(): void {
    if (!NgZone.isInAngularZone()) {
      throw new Error('Expected to be in Angular Zone, but it is not!');
    }
  }

  static assertNotInAngularZone(): void {
    if (NgZone.isInAngularZone()) {
      throw new Error('Expected to not be in Angular Zone, but it is!');
    }
  }
  
  run<T>(fn: (...args: any[]) => T, applyThis?: any, applyArgs?: any[]): T {
    return (this as any as NgZonePrivate)._inner.run(fn, applyThis, applyArgs);
  }
  
  runTask<T>(fn: (...args: any[]) => T, applyThis?: any, applyArgs?: any[], name?: string): T {
    const zone = (this as any as NgZonePrivate)._inner;
    const task = zone.scheduleEventTask('NgZoneEvent: ' + name, fn, EMPTY_PAYLOAD, noop, noop);
    try {
      return zone.runTask(task, applyThis, applyArgs);
    } finally {
      zone.cancelTask(task);
    }
  }
  
  runGuarded<T>(fn: (...args: any[]) => T, applyThis?: any, applyArgs?: any[]): T {
    return (this as any as NgZonePrivate)._inner.runGuarded(fn, applyThis, applyArgs);
  }
  
  runOutsideAngular<T>(fn: (...args: any[]) => T): T {
    return (this as any as NgZonePrivate)._outer.run(fn);
  }
}
```

As always, analysis of the class we start with the constructor.
It looks complicated, but it's not.
The first part of the code is just the process of zone forking
and setting necessary properties provided by the configuration from the platform.

The last line of the constructor hides the most important stuff.
The call of the `forkInnerZoneWithAngularBehavior` function,
contains logic of the behaviour that has logic important for Angular.

```typescript
function forkInnerZoneWithAngularBehavior(zone: NgZonePrivate) {
  const delayChangeDetectionForEventsDelegate = () => {
    delayChangeDetectionForEvents(zone);
  };
  zone._inner = zone._inner.fork({
    name: 'angular',
    properties: <any>{'isAngularZone': true},
    onInvokeTask:
        (delegate: ZoneDelegate, current: Zone, target: Zone, task: Task, applyThis: any,
         applyArgs: any): any => {
          try {
            onEnter(zone);
            return delegate.invokeTask(target, task, applyThis, applyArgs);
          } finally {
            if ((zone.shouldCoalesceEventChangeDetection && task.type === 'eventTask') ||
                zone.shouldCoalesceRunChangeDetection) {
              delayChangeDetectionForEventsDelegate();
            }
            onLeave(zone);
          }
        },

    onInvoke:
        (delegate: ZoneDelegate, current: Zone, target: Zone, callback: Function, applyThis: any,
         applyArgs?: any[], source?: string): any => {
          try {
            onEnter(zone);
            return delegate.invoke(target, callback, applyThis, applyArgs, source);
          } finally {
            if (zone.shouldCoalesceRunChangeDetection) {
              delayChangeDetectionForEventsDelegate();
            }
            onLeave(zone);
          }
        },

    onHasTask:
        (delegate: ZoneDelegate, current: Zone, target: Zone, hasTaskState: HasTaskState) => {
          delegate.hasTask(target, hasTaskState);
          if (current === target) {
            // We are only interested in hasTask events which originate from our zone
            // (A child hasTask event is not interesting to us)
            if (hasTaskState.change == 'microTask') {
              zone._hasPendingMicrotasks = hasTaskState.microTask;
              updateMicroTaskStatus(zone);
              checkStable(zone);
            } else if (hasTaskState.change == 'macroTask') {
              zone.hasPendingMacrotasks = hasTaskState.macroTask;
            }
          }
        },

    onHandleError: (delegate: ZoneDelegate, current: Zone, target: Zone, error: any): boolean => {
      delegate.handleError(target, error);
      zone.runOutsideAngular(() => zone.onError.emit(error));
      return false;
    }
  });
}
```

Again, it looks complicated, but it's not.
Zone.js implementation works on callbacks,
so if something happens and zone.js catches an event we
(the programmers) can provide a callback and execute some kind of side effect.

In case of `NgZone` these side effects are described by the functions:
`onEnter(zone)`, `onLeave(zone)`, `updateMicroTaskStatus(zone)` and `checkStable(zone)`. 


`updateMicroTaskStatus` changes only the status of the field `hasPendingMicrotasks`.

`onEnter` nests new zone and changes the NgZone to unstable state, and emits the information on `onUnstable` stream.

`onLeave` reduces nesting of the zones and checks if zone is in stable state.

At last, `checkStable`, checks if zone is not nested anymore and
if there are zero pending micro tasks and zone is not stable.
If all conditions are fulfilled, two events are emitted.
First one on `onMicrotaskEmpty` stream and the second one on `onStable` stream.
Please remember this information, we will use it later.

```typescript
function updateMicroTaskStatus(zone: NgZonePrivate) {
  if (zone._hasPendingMicrotasks ||
      ((zone.shouldCoalesceEventChangeDetection || zone.shouldCoalesceRunChangeDetection) &&
       zone.lastRequestAnimationFrameId !== -1)) {
    zone.hasPendingMicrotasks = true;
  } else {
    zone.hasPendingMicrotasks = false;
  }
}

function onEnter(zone: NgZonePrivate) {
  zone._nesting++;
  if (zone.isStable) {
    zone.isStable = false;
    zone.onUnstable.emit(null);
  }
}

function onLeave(zone: NgZonePrivate) {
  zone._nesting--;
  checkStable(zone);
}

function checkStable(zone: NgZonePrivate) {
    if (zone._nesting == 0 && !zone.hasPendingMicrotasks && !zone.isStable) {
        try {
            zone._nesting++;
            zone.onMicrotaskEmpty.emit(null);
        } finally {
            zone._nesting--;
            if (!zone.hasPendingMicrotasks) {
                try {
                    zone.runOutsideAngular(() => zone.onStable.emit(null));
                } finally {
                    zone.isStable = true;
                }
            }
        }
    }
}
```

Basically, that's all. There is nothing complicated in implementation of `NgZone`.
All magic is happening inside `zone.js` library and Angular is adding few callbacks to make it working. 

BTW. Check out interesting way of using typescript in `NgZone` class.
internally is using `NgZonePrivate` interface that extends `NgZone`.
Check out the [decorator pattern](https://refactoring.guru/design-patterns/decorator).

Using this method allows us to declare fields and use all cool features from Typescript, but only when we cast it.
Looks like a cool way to add "secret" fields to the class.

```typescript
interface NgZonePrivate extends NgZone {
  _outer: Zone;
  _inner: Zone;
  _nesting: number;
  _hasPendingMicrotasks: boolean;
  hasPendingMacrotasks: boolean;
  hasPendingMicrotasks: boolean;
  lastRequestAnimationFrameId: number;
  isCheckStableRunning: boolean;
  isStable: boolean;
  shouldCoalesceEventChangeDetection: boolean;
  shouldCoalesceRunChangeDetection: boolean;
  nativeRequestAnimationFrame: (callback: FrameRequestCallback) => number;
  fakeTopEventTask: Task;
}
```

## NoopNgZone

`NoopNgZone` is way easier than `NgZone`.  

```typescript
export class NoopNgZone implements NgZone {
  readonly hasPendingMicrotasks: boolean = false;
  readonly hasPendingMacrotasks: boolean = false;
  readonly isStable: boolean = true;
  readonly onUnstable: EventEmitter<any> = new EventEmitter();
  readonly onMicrotaskEmpty: EventEmitter<any> = new EventEmitter();
  readonly onStable: EventEmitter<any> = new EventEmitter();
  readonly onError: EventEmitter<any> = new EventEmitter();

  run<T>(fn: (...args: any[]) => T, applyThis?: any, applyArgs?: any): T {
    return fn.apply(applyThis, applyArgs);
  }

  runGuarded<T>(fn: (...args: any[]) => any, applyThis?: any, applyArgs?: any): T {
    return fn.apply(applyThis, applyArgs);
  }

  runOutsideAngular<T>(fn: (...args: any[]) => T): T {
    return fn();
  }

  runTask<T>(fn: (...args: any[]) => T, applyThis?: any, applyArgs?: any, name?: string): T {
    return fn.apply(applyThis, applyArgs);
  }
}
```

Basically, there is nothing there.
Every action is just called and not passed to the zone.
In this case `zone.js` library is needless, you don't have to load it.
It's not a lot, but still it's something.

![zone](/mythical-angular/images/zone-size.png)

## How change detection is triggered by NgZone

We know how `NgZone` and `NoopNgZone` is implemented.
The most important question is how it triggers the change detection process...?

To find it, we have to follow `onMicrotaskEmpty` stream (event emitter if you want).
Little reminder, value is emitted in this stream, when every micro task is completed.
So when there is only one micro task at the same moment, there is no problem; value is emitted.
When Angular is dealing with more than one micro task, it will emit value only **when all micro tasks are completed**.
It's some kind of optimization. 
The idea is to reduce unnecessary change detection processes.

So `onMicrotaskEmpty` has only one subscriber, that is constructor of `ApplicationRef` service.
On every value `tick` method is called, and it's a direct start of change detection.

```typescript
@Injectable({providedIn: 'root'})
export class ApplicationRef {
    /* ... */
    constructor(
        private _zone: NgZone, private _injector: Injector, private _exceptionHandler: ErrorHandler,
        private _componentFactoryResolver: ComponentFactoryResolver,
        private _initStatus: ApplicationInitStatus) {
        this._onMicrotaskEmptySubscription = this._zone.onMicrotaskEmpty.subscribe({
            next: () => {
                this._zone.run(() => {
                    this.tick();
                });
            }
        });
    }
    /* ... */
}
```

## Summary

That's all for today.
Next time, I will compare the performance of standard and zone-less Angular application.

Probably soon I will release the description of all methods of triggering change detection from code,
I think that it might be interesting.
Is there a difference between `ApplicationRef.tick()` and `ChangeDetectorRef.detectChanges()`...?
Of course, there is, but it's a subject for a different article.
