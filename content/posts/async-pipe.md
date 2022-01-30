title: "Myth: Async pipes are good for performance" date: 2022-01-29 draft: false

Introduction
Async pipes are considered a very good practice as they help with web performance problems and btw are easy to use. Sounds very cool, right?

Unfortunately, it's not true. It would be so good to use async pipes everywhere and don't worry about any observable related performance problems.

Before I'll start to explain where the problem with async pipe is, let's check how it's implemented.

AsyncPipe source code
/**
* @license
* Copyright Google LLC All Rights Reserved.
*
* Use of this source code is governed by an MIT-style license that can be
* found in the LICENSE file at https://angular.io/license
  */

import {ChangeDetectorRef, EventEmitter, OnDestroy, Pipe, PipeTransform, ɵisPromise, ɵisSubscribable} from '@angular/core';
import {Observable, Subscribable, Unsubscribable} from 'rxjs';

import {invalidPipeArgumentError} from './invalid_pipe_argument_error';

interface SubscriptionStrategy {
createSubscription(async: Subscribable<any>|Promise<any>, updateLatestValue: any): Unsubscribable
|Promise<any>;
dispose(subscription: Unsubscribable|Promise<any>): void;
onDestroy(subscription: Unsubscribable|Promise<any>): void;
}

class SubscribableStrategy implements SubscriptionStrategy {
createSubscription(async: Subscribable<any>, updateLatestValue: any): Unsubscribable {
return async.subscribe({
next: updateLatestValue,
error: (e: any) => {
throw e;
}
});
}

dispose(subscription: Unsubscribable): void {
subscription.unsubscribe();
}

onDestroy(subscription: Unsubscribable): void {
subscription.unsubscribe();
}
}

class PromiseStrategy implements SubscriptionStrategy {
createSubscription(async: Promise<any>, updateLatestValue: (v: any) => any): Promise<any> {
return async.then(updateLatestValue, e => {
throw e;
});
}

dispose(subscription: Promise<any>): void {}

onDestroy(subscription: Promise<any>): void {}
}

const _promiseStrategy = new PromiseStrategy();
const _subscribableStrategy = new SubscribableStrategy();

/**
* @ngModule CommonModule
* @description
*
* Unwraps a value from an asynchronous primitive.
*
* The `async` pipe subscribes to an `Observable` or `Promise` and returns the latest value it has
* emitted. When a new value is emitted, the `async` pipe marks the component to be checked for
* changes. When the component gets destroyed, the `async` pipe unsubscribes automatically to avoid
* potential memory leaks. When the reference of the expression changes, the `async` pipe
* automatically unsubscribes from the old `Observable` or `Promise` and subscribes to the new one.
*
* @usageNotes
*
* ### Examples
*
* This example binds a `Promise` to the view. Clicking the `Resolve` button resolves the
* promise.
*
* {@example common/pipes/ts/async_pipe.ts region='AsyncPipePromise'}
*
* It's also possible to use `async` with Observables. The example below binds the `time` Observable
* to the view. The Observable continuously updates the view with the current time.
*
* {@example common/pipes/ts/async_pipe.ts region='AsyncPipeObservable'}
*
* @publicApi
  */
  @Pipe({name: 'async', pure: false})
  export class AsyncPipe implements OnDestroy, PipeTransform {
  private _latestValue: any = null;

private _subscription: Unsubscribable|Promise<any>|null = null;
private _obj: Subscribable<any>|Promise<any>|EventEmitter<any>|null = null;
private _strategy: SubscriptionStrategy = null!;

constructor(private _ref: ChangeDetectorRef) {}

ngOnDestroy(): void {
if (this._subscription) {
this._dispose();
}
}

// NOTE(@benlesh): Because Observable has deprecated a few call patterns for `subscribe`,
// TypeScript has a hard time matching Observable to Subscribable, for more information
// see https://github.com/microsoft/TypeScript/issues/43643

transform<T>(obj: Observable<T>|Subscribable<T>|Promise<T>): T|null;
transform<T>(obj: null|undefined): null;
transform<T>(obj: Observable<T>|Subscribable<T>|Promise<T>|null|undefined): T|null;
transform<T>(obj: Observable<T>|Subscribable<T>|Promise<T>|null|undefined): T|null {
if (!this._obj) {
if (obj) {
this._subscribe(obj);
}
return this._latestValue;
}

    if (obj !== this._obj) {
      this._dispose();
      return this.transform(obj);
    }

    return this._latestValue;
}

private _subscribe(obj: Subscribable<any>|Promise<any>|EventEmitter<any>): void {
this._obj = obj;
this._strategy = this._selectStrategy(obj);
this._subscription = this._strategy.createSubscription(
obj, (value: Object) => this._updateLatestValue(obj, value));
}

private _selectStrategy(obj: Subscribable<any>|Promise<any>|EventEmitter<any>): any {
if (ɵisPromise(obj)) {
return _promiseStrategy;
}

    if (ɵisSubscribable(obj)) {
      return _subscribableStrategy;
    }

    throw invalidPipeArgumentError(AsyncPipe, obj);
}

private _dispose(): void {
this._strategy.dispose(this._subscription!);
this._latestValue = null;
this._subscription = null;
this._obj = null;
}

private _updateLatestValue(async: any, value: Object): void {
if (async === this._obj) {
this._latestValue = value;
this._ref.markForCheck();
}
}
}
The important part starts in line 55. As you can see, AsyncPipe code is not that complicated. It's just a standard not-pure pipe.

There is the implementation of the transform method that gets Observable or Promise object, subscribes to it in _subscribe, and calls _updateLatestValue for every value emitted in the provided stream.

_updateLatestValue is the most important part in the whole class.

private _updateLatestValue(async: any, value: Object): void {
if (async === this._obj) {
this._latestValue = value;
this._ref.markForCheck();
}
}
Again, it's a very simple code. It sets the new value to the _latestValue and calls ChangeDetectorRef.markForCheck().

Little summary. So, AsyncPipe subscribes to a stream, saves the last emitted value to the local state, and at the end marks component for a check. Easy right?

Why we use AsyncPipe
AsyncPipe is considered to be good because it solves another very important problem. Nature of observable streams is very simple. If you subscribe to a stream, you have to remember to unsubscribe.

We can use a lot of very interesting patterns to make it easier to remember about that. This build-in pipe is using another build-in mechanism called Angular Lifecycle Hooks. It's very simple, AsyncPipe will unsubscribe on the OnDestroy hook.

With a little framework magic, we get the automatic unsubscribe mechanism.

Where is the problem?
As you may notice, the whole idea behind this pipe is very smart. But there is no light without the dark. In method _updateLatestValue we call ChangeDetectorRef.markForCheck().

From the sources, we know that after marking the view dirty it's going to be refreshed. Nothing surprising, it looks OK at first sight. We updated the value so view has to render once again.

export abstract class ChangeDetectorRef {
/**
* When a view uses the {@link ChangeDetectionStrategy#OnPush OnPush} (checkOnce)
* change detection strategy, explicitly marks the view as changed so that
* it can be checked again.
*
* Components are normally marked as dirty (in need of rerendering) when inputs
* have changed or events have fired in the view. Call this method to ensure that
* a component is checked even if these triggers have not occured.
*
* <!-- TODO: Add a link to a chapter on OnPush components -->
*
*/
abstract markForCheck(): void;

/* ... */
}
ChangeDetectorRef is an abstract class. After a few jumps in the code, you'll find the right implementation. So at the very end of there is code presented bellow.

/**
* Marks current view and all ancestors dirty.
*
* Returns the root view because it is found as a byproduct of marking the view tree
* dirty, and can be used by methods that consume markViewDirty() to easily schedule
* change detection. Otherwise, such methods would need to traverse up the view tree
* an additional time to get the root view and schedule a tick on it.
*
* @param lView The starting LView to mark dirty
* @returns the root LView
  */
  export function markViewDirty(lView: LView): LView|null {
  while (lView) {
  lView[FLAGS] |= LViewFlags.Dirty;
  const parent = getLViewParent(lView);
  // Stop traversing up as soon as you find a root view that wasn't attached to any container
  if (isRootView(lView) && !parent) {
  return lView;
  }
  // continue otherwise
  lView = parent!;
  }
  return null;
  }
  I'm going to cite the most important part: "Marks current view and all ancestors dirty.". It means that if you use async pipe, and there is a new value on the stream, it's going to refresh your component and all its parents.

trustthecode

Is it a necessary thing to do...? No.

Template changed only in one component there is no reason to render more than one view. Why Angular is checking everything on the path to the top...? There is a simple explanation. To refresh the view after marking it dirty, we need NgZones to run the whole check. NgZones are using ApplicationRef.tick() to process the checking, and it's processed from the top to bottom.

How to solve the problem
Manual solution
Forget about AsyncPipes. Subscribe to your streams in a component. Assign the value from the stream to class field and call ChangeDetectorRef.detectChanges().

It's that simple. ChangeDetectorRef.detectChanges() will refresh only one view. You still need to unsubscribe the streams, but your runtime performance will be a lot better.

PushPipe or rxLet directive from rx-angular
Package rx-angular offers a smart solution for the problem. It's called PushPipe. I'm not going to publish code for it, you can find it in the repository.

Summary
Right now, I'm sure that I have convinced you that AsyncPipe should be considered harmful as it's not good for your performance.

There are other solutions for the problem. It's not worth sacrificing rendering performance for automatic unsubscription. You can handle it manually or use other solutions.

BTW. If you're planning to go for zone-less app, you have to remove usages of the async anyway. It's not working without zone.js.