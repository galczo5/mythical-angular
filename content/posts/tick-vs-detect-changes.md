---
title: "Myth: There is no difference between ApplicationRef.tick() and ChangeDetectorRef.detectChanges()"
date: 2022-05-21
draft: false
---

"I use ApplicationRef.tick() because it works, and after all, there is no difference"

Come on.
These kinds of sentences almost always are not true.
In this article I will show you how big difference is between `ApplicationRef.tick` and `ChangeDetectorRef.tick`. 

Please remember that all my research is focused on using Angular in "high-performance" mode.
For normal usage, simple webpages or small apps, there is a difference, of course,
but I doubt that we should care about it.

## Application

To test this myth, I wrote a simple application.
The code you will find on my github:
[github.com/galczo5/experiment-angular-tick](https://galczo5.github.io/experiment-angular-tick/).
The application has a few components.
On each component, you can execute two actions:
- tick - which calls `ApplicationRef.tick()`
- detectChanges - call of `ChangeDetectorRef.detectChanges()` method

In addition, I've added timestamp updated in an interval to visualize changes made in a component.
It's a simple test to prove that change detection works.

`noop` zone is provided and every component is using `OnPush` strategy.

{{< rawhtml >}}
<div style="height: 500px; width: 1050px; border: 1px solid lightgray;">
    <iframe style="border: 0; width: 100%; height: 100%;" src="https://galczo5.github.io/experiment-angular-tick/"></iframe>
</div>
{{< /rawhtml >}}

## The test

To understand the first and the most important test in this article, you should check out the code of `ChildComponent`.

```typescript
import {
  ApplicationRef,
  ChangeDetectionStrategy,
  ChangeDetectorRef,
  Component,
  DoCheck,
  Input,
  OnChanges, OnDestroy, OnInit
} from '@angular/core';
import {interval, Subject, takeUntil} from "rxjs";

@Component({
  selector: 'app-child',
  template: `
    <div style="padding: 10px; border: 1px solid;">
      <h1>Child {{id}} [{{timestamp}}]</h1>
      <button (click)="tick()">tick()</button>
      <button (click)="detect()">detectChanges()</button>
      <hr>
      <div style="display: flex; gap: 10px;">
        <app-child style="flex-grow: 1" *ngFor="let childId of childIds" [id]="childId"></app-child>
      </div>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ChildComponent implements OnInit, DoCheck, OnChanges, OnDestroy {

  @Input()
  id: string | undefined;

  @Input()
  childIds: Array<string> = [];

  timestamp: number = 0;

  private readonly destroy$: Subject<void> = new Subject<void>();

  constructor(
    private readonly applicationRef: ApplicationRef,
    private readonly changeDetectorRef: ChangeDetectorRef
  ) {
  }

  tick() {
    console.log('tick');
    this.applicationRef.tick();
  }

  detect() {
    console.log('detectChanges');
    this.changeDetectorRef.detectChanges();
  }

  ngOnInit() {
    console.log('ngOnInit', 'ChildComponent', this.id);

    interval(1000)
      .pipe(
        takeUntil(this.destroy$)
      )
      .subscribe(() => {
        this.timestamp = new Date().getTime();
      });
  }

  ngOnChanges() {
    console.log('ngOnChanges', 'ChildComponent', this.id);
  }

  ngOnDestroy() {
    console.log('ngOnDestroy', 'ChildComponent', this.id);

    this.destroy$.next();
    this.destroy$.complete();
  }

  ngDoCheck() {
    console.log('ngDoCheck', 'ChildComponent', this.id);
  }
}
```

My testing component has some hooks implemented to print out information when hooks are executed,
besides that we have two methods triggered by button click.
One of them calls `ApplicationRef.tick()` and the second one calls `ChangeDetectorRef.detectChanges()`.
This is a very simple test to verify differences between these two methods of triggering change detection.
I want to check which one we should use when we care about performance,
and we want to ensure that we make everything we could to make Angular apps as fast as possible.

Ok, let's test something.

Open the dev tools and click button `detectChanges` in the Child 3 component.
As a result,
you should observe a text `detectChanges` in the console of dev tools and a timestamp
printed next to header in a component.
The timestamp proves that the component was rendered by the change detection.
No unnecessary hook was called.

Now, refresh the page, and click on `tick` button.
Inside the console you should observe:
```text
main.af80531c057bb360.js:1 tick
main.af80531c057bb360.js:1 ngDoCheck AppComponent
main.af80531c057bb360.js:1 ngDoCheck ChildComponent 1
main.af80531c057bb360.js:1 ngDoCheck ChildComponent 2
main.af80531c057bb360.js:1 ngDoCheck ChildComponent 3
main.af80531c057bb360.js:1 ngDoCheck ChildComponent 4
```

What's more important, again next to the header you should be able to observe timestamps again.
The difference is that not only Child 3 component was re-rendered but also Parent and Child 2 was re-rendered too.

![tick](/mythical-angular/images/tick.png)

This simply makes me sad :( I like to avoid any unnecessary functions call as you may notice from my previous posts. 

To sum up, `ChangeDetectorRef.detectChanges()` is a way better option to refresh view in a single component.
You can avoid a lot of work on browser thread, but it means that you have to know what and when you want to refresh!

It's time to dive in code and reveal the differences between these two methods.

## `ChangeDetectorRef.detectChanges()`

My investigation
I started by setting breakpoint in `ChildComponent.detect()` method and stepping into `detectChanges` call.
It leads me to `ViewRef` class, it makes sense because it implements `ChangeDetectorRef`.
Implementation of `detectChanges` method is fairly simple:

```typescript
class ViewRef$1 {
    // ...
    detectChanges() {
        detectChangesInternal(this._lView[TVIEW], this._lView, this.context);
    }
    // ...
}
```

So Angular passes some arguments related to view (component instance) to the `detectChangesInternal`,
which is quite simple too: 
```typescript
function detectChangesInternal(tView, lView, context) {
    const rendererFactory = lView[RENDERER_FACTORY];
    if (rendererFactory.begin)
        rendererFactory.begin();
    try {
        refreshView(tView, lView, tView.template, context);
    }
    catch (error) {
        handleError(lView, error);
        throw error;
    }
    finally {
        if (rendererFactory.end)
            rendererFactory.end();
    }
}
```

Again, we pass some arguments to the next function called `refresView`.
BTW, remember that `tView.template` is not a string containing the HTML markup of the view.
It's a function that's processing components state, and at the end of the process it will produce the HTML elements. 

In my case, it looks like: 
```typescript
function ChildComponent_Template(rf, ctx) {
    if (rf & 1) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](0, "div", 0)(1, "h1");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](2);
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](3, "button", 1);
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵlistener"]("click", function ChildComponent_Template_button_click_3_listener() {
            return ctx.tick();
        });
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](4, "tick()");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](5, "button", 1);
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵlistener"]("click", function ChildComponent_Template_button_click_5_listener() {
            return ctx.detect();
        });
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](6, "detectChanges()");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelement"](7, "hr");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](8, "div", 2);
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtemplate"](9, ChildComponent_app_child_9_Template, 1, 1, "app-child", 3);
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]()();
    }
    if (rf & 2) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵadvance"](2);
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtextInterpolate2"]("Child ", ctx.id, " [", ctx.timestamp, "]");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵadvance"](7);
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵproperty"]("ngForOf", ctx.childIds);
    }
}
```

I wanted to mention that to remind you that there is no magic in the Angular source code,
and it's not that hard to understand it.

Ok, back to our `refreshView`, because fun begins in this function.

```typescript
/**
 * Processes a view in update mode. This includes a number of steps in a specific order:
 * - executing a template function in update mode;
 * - executing hooks;
 * - refreshing queries;
 * - setting host bindings;
 * - refreshing child (embedded and component) views.
 */
function refreshView(tView, lView, templateFn, context) {
    ngDevMode && assertEqual(isCreationMode(lView), false, 'Should be run in update mode');
    const flags = lView[FLAGS];
    if ((flags & 256 /* Destroyed */) === 256 /* Destroyed */)
        return;
    enterView(lView);
    // Check no changes mode is a dev only mode used to verify that bindings have not changed
    // since they were assigned. We do not want to execute lifecycle hooks in that mode.
    const isInCheckNoChangesPass = isInCheckNoChangesMode();
    try {
        resetPreOrderHookFlags(lView);
        setBindingIndex(tView.bindingStartIndex);
        if (templateFn !== null) {
            executeTemplate(tView, lView, templateFn, 2 /* Update */, context);
        }
        const hooksInitPhaseCompleted = (flags & 3 /* InitPhaseStateMask */) === 3 /* InitPhaseCompleted */;
        // execute pre-order hooks (OnInit, OnChanges, DoCheck)
        // PERF WARNING: do NOT extract this to a separate function without running benchmarks
        if (!isInCheckNoChangesPass) {
            if (hooksInitPhaseCompleted) {
                const preOrderCheckHooks = tView.preOrderCheckHooks;
                if (preOrderCheckHooks !== null) {
                    executeCheckHooks(lView, preOrderCheckHooks, null);
                }
            }
            else {
                const preOrderHooks = tView.preOrderHooks;
                if (preOrderHooks !== null) {
                    executeInitAndCheckHooks(lView, preOrderHooks, 0 /* OnInitHooksToBeRun */, null);
                }
                incrementInitPhaseFlags(lView, 0 /* OnInitHooksToBeRun */);
            }
        }
        // First mark transplanted views that are declared in this lView as needing a refresh at their
        // insertion points. This is needed to avoid the situation where the template is defined in this
        // `LView` but its declaration appears after the insertion component.
        markTransplantedViewsForRefresh(lView);
        refreshEmbeddedViews(lView);
        // Content query results must be refreshed before content hooks are called.
        if (tView.contentQueries !== null) {
            refreshContentQueries(tView, lView);
        }
        // execute content hooks (AfterContentInit, AfterContentChecked)
        // PERF WARNING: do NOT extract this to a separate function without running benchmarks
        if (!isInCheckNoChangesPass) {
            if (hooksInitPhaseCompleted) {
                const contentCheckHooks = tView.contentCheckHooks;
                if (contentCheckHooks !== null) {
                    executeCheckHooks(lView, contentCheckHooks);
                }
            }
            else {
                const contentHooks = tView.contentHooks;
                if (contentHooks !== null) {
                    executeInitAndCheckHooks(lView, contentHooks, 1 /* AfterContentInitHooksToBeRun */);
                }
                incrementInitPhaseFlags(lView, 1 /* AfterContentInitHooksToBeRun */);
            }
        }
        processHostBindingOpCodes(tView, lView);
        // Refresh child component views.
        const components = tView.components;
        if (components !== null) {
            refreshChildComponents(lView, components);
        }
        // View queries must execute after refreshing child components because a template in this view
        // could be inserted in a child component. If the view query executes before child component
        // refresh, the template might not yet be inserted.
        const viewQuery = tView.viewQuery;
        if (viewQuery !== null) {
            executeViewQueryFn(2 /* Update */, viewQuery, context);
        }
        // execute view hooks (AfterViewInit, AfterViewChecked)
        // PERF WARNING: do NOT extract this to a separate function without running benchmarks
        if (!isInCheckNoChangesPass) {
            if (hooksInitPhaseCompleted) {
                const viewCheckHooks = tView.viewCheckHooks;
                if (viewCheckHooks !== null) {
                    executeCheckHooks(lView, viewCheckHooks);
                }
            }
            else {
                const viewHooks = tView.viewHooks;
                if (viewHooks !== null) {
                    executeInitAndCheckHooks(lView, viewHooks, 2 /* AfterViewInitHooksToBeRun */);
                }
                incrementInitPhaseFlags(lView, 2 /* AfterViewInitHooksToBeRun */);
            }
        }
        if (tView.firstUpdatePass === true) {
            // We need to make sure that we only flip the flag on successful `refreshView` only
            // Don't do this in `finally` block.
            // If we did this in `finally` block then an exception could block the execution of styling
            // instructions which in turn would be unable to insert themselves into the styling linked
            // list. The result of this would be that if the exception would not be throw on subsequent CD
            // the styling would be unable to process it data and reflect to the DOM.
            tView.firstUpdatePass = false;
        }
        // Do not reset the dirty state when running in check no changes mode. We don't want components
        // to behave differently depending on whether check no changes is enabled or not. For example:
        // Marking an OnPush component as dirty from within the `ngAfterViewInit` hook in order to
        // refresh a `NgClass` binding should work. If we would reset the dirty state in the check
        // no changes cycle, the component would be not be dirty for the next update pass. This would
        // be different in production mode where the component dirty state is not reset.
        if (!isInCheckNoChangesPass) {
            lView[FLAGS] &= ~(64 /* Dirty */ | 8 /* FirstLViewPass */);
        }
        if (lView[FLAGS] & 1024 /* RefreshTransplantedView */) {
            lView[FLAGS] &= ~1024 /* RefreshTransplantedView */;
            updateTransplantedViewCount(lView[PARENT], -1);
        }
    }
    finally {
        leaveView();
    }
}
```

I'm not going to describe every line of this function, it's quite complicated, but please try to analyze it.
From point of this article, two things in this function are the most important.
The first, the actual template refresh:
```typescript
function refreshView(tView, lView, templateFn, context) {
    // ...
    if (templateFn !== null) {
        executeTemplate(tView, lView, templateFn, 2 /* Update */, context);
    }
    // ...
}

// ...

function executeTemplate(tView, lView, templateFn, rf, context) {
    const prevSelectedIndex = getSelectedIndex();
    const isUpdatePhase = rf & 2 /* Update */;
    try {
        setSelectedIndex(-1);
        if (isUpdatePhase && lView.length > HEADER_OFFSET) {
            // When we're updating, inherently select 0 so we don't
            // have to generate that instruction for most update blocks.
            selectIndexInternal(tView, lView, HEADER_OFFSET, isInCheckNoChangesMode());
        }
        const preHookType = isUpdatePhase ? 2 /* TemplateUpdateStart */ : 0 /* TemplateCreateStart */;
        profiler(preHookType, context);
        templateFn(rf, context);
    }
    finally {
        setSelectedIndex(prevSelectedIndex);
        const postHookType = isUpdatePhase ? 3 /* TemplateUpdateEnd */ : 1 /* TemplateCreateEnd */;
        profiler(postHookType, context);
    }
}
```

And, the second one, is refreshing the child views.
```typescript
function refreshView(tView, lView, templateFn, context) {
    // ...
    refreshEmbeddedViews(lView);

    // ...

    if (components !== null) {
        refreshChildComponents(lView, components);
    }
    // ...
}
```

## ApplicationRef.tick()

As you may expect, the tick method is a bit different. It has to be, because the result of the test was different, right?

```typescript
class ApplicationRef {
    // ...
    tick() {
        if (this._runningTick) {
            const errorMessage = (typeof ngDevMode === 'undefined' || ngDevMode) ?
                'ApplicationRef.tick is called recursively' :
                '';
            throw new RuntimeError(101 /* RECURSIVE_APPLICATION_REF_TICK */, errorMessage);
        }
        try {
            this._runningTick = true;
            for (let view of this._views) {
                view.detectChanges();
            }
            if (typeof ngDevMode === 'undefined' || ngDevMode) {
                for (let view of this._views) {
                    view.checkNoChanges();
                }
            }
        }
        catch (e) {
            // Attention: Don't rethrow as it could cancel subscriptions to Observables!
            this._zone.runOutsideAngular(() => this._exceptionHandler.handleError(e));
        }
        finally {
            this._runningTick = false;
        }
    }
    // ...
}
```

Stop... Wait. Yes, it calls the same `detectChanges()` method on root views!
What's important too, these views are implementing `RootViewRef`, so we have another difference.

![tick](/mythical-angular/images/root-view-ref.png)

```typescript
class RootViewRef extends ViewRef$1 {
    constructor(_view) {
        super(_view);
        this._view = _view;
    }
    detectChanges() {
        detectChangesInRootView(this._view);
    }
    checkNoChanges() {
        checkNoChangesInRootView(this._view);
    }
    get context() {
        return null;
    }
}

// ...

function detectChangesInRootView(lView) {
    tickRootContext(lView[CONTEXT]);
}

// ...

function tickRootContext(rootContext) {
    for (let i = 0; i < rootContext.components.length; i++) {
        const rootComponent = rootContext.components[i];
        const lView = readPatchedLView(rootComponent);
        const tView = lView[TVIEW];
        renderComponentOrTemplate(tView, lView, tView.template, rootComponent);
    }
}
```

So it calls `detectChangesInRootView`, which calls `tickRootContext`,
which leads us to the `renderComponentOrTemplate` call.

```typescript
function renderComponentOrTemplate(tView, lView, templateFn, context) {
    const rendererFactory = lView[RENDERER_FACTORY];
    const normalExecutionPath = !isInCheckNoChangesMode();
    const creationModeIsActive = isCreationMode(lView);
    try {
        if (normalExecutionPath && !creationModeIsActive && rendererFactory.begin) {
            rendererFactory.begin();
        }
        if (creationModeIsActive) {
            renderView(tView, lView, context);
        }
        refreshView(tView, lView, templateFn, context);
    }
    finally {
        if (normalExecutionPath && !creationModeIsActive && rendererFactory.end) {
            rendererFactory.end();
        }
    }
}
```

At the end, Angular calls the same `renderView` function, which will refresh the root component and its children.

## Summary

From this article you should learn about the differences between the `ChangeDetectorRef.detectChanges()` and `ApplicationRef.tick()` methods.
Personally,
I like the behaviour
of `detectChanges` and when I precisely know which component I want to re-render I use it.
Please remember that I very rarely use the default settings
and `NgZones` to avoid execution of javascript code that is unnecessary to get the same result. 

If you wonder, yes, I know that again my article contains tons of source code from the Angular repository.
For me, it's important to show you that there is no magic, that code is not that hard to understand.

![tick](/mythical-angular/images/trust-the-code.png)
