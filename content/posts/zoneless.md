---
title: "Tutorial: How to start zone-less Angular app"
date: 2022-04-11
draft: false
---

Well, hello again!

This time I decided to do something else that usual.
Today I will not try to describe and demystify any Angular-related myth,
because today I will show you a simple tutorial how to start a new Angular app without NgZones.

## Generating new app

There is no reason to describe it very deeply.
It's just a standard `ng new` command.
In my example, I'm using routing.
First, you probably use router too,
and I want to make this example as simple as possible, but still pretty close to real-life apps.
Second of all, there is one trick necessary to use router in zone-less mode, and I want to show and describe it.

## App demo



## Disabling NgZones

First thing you probably want to do is disabling `zone.js`.
It's very easy to do, find the file where you obtain `PlatformRef` object, and you bootstrap your first module.
By default, it's the `main.ts` file.

The Code should look similar to the snippet below.

```typescript
import {enableProdMode} from '@angular/core';
import {platformBrowserDynamic} from '@angular/platform-browser-dynamic';

import {AppModule} from './app/app.module';
import {environment} from './environments/environment';

if (environment.production) {
  enableProdMode();
}

const compilerOptions = {
  ngZone: 'noop' as 'noop'
};

platformBrowserDynamic().bootstrapModule(AppModule, compilerOptions)
  .catch(err => console.error(err));
```

And that is it. It's so simple, isn't it?

Sooooo...
because it was so easy,
I want to describe a little implementation of the first things executed by Angular when we start any Angular app. 

### Production mode

First, on production we call function `enableProdMode`. It would be good to check it's implementation.

```typescript
/**
 * @license
 * Copyright Google LLC All Rights Reserved.
 *
 * Use of this source code is governed by an MIT-style license that can be
 * found in the LICENSE file at https://angular.io/license
 */

import {global} from './global';

/**
 * This file is used to control if the default rendering pipeline should be `ViewEngine` or `Ivy`.
 *
 * For more information on how to run and debug tests with either Ivy or View Engine (legacy),
 * please see [BAZEL.md](./docs/BAZEL.md).
 */

let _devMode: boolean = true;
let _runModeLocked: boolean = false;


/**
 * Returns whether Angular is in development mode. After called once,
 * the value is locked and won't change any more.
 *
 * By default, this is true, unless a user calls `enableProdMode` before calling this.
 *
 * @publicApi
 */
export function isDevMode(): boolean {
    _runModeLocked = true;
    return _devMode;
}

/**
 * Disable Angular's development mode, which turns off assertions and other
 * checks within the framework.
 *
 * One important assertion this disables verifies that a change detection pass
 * does not result in additional changes to any bindings (also known as
 * unidirectional data flow).
 *
 * @publicApi
 */
export function enableProdMode(): void {
    if (_runModeLocked) {
        throw new Error('Cannot enable prod mode after platform setup.');
    }

    // The below check is there so when ngDevMode is set via terser
    // `global['ngDevMode'] = false;` is also dropped.
    if (typeof ngDevMode === undefined || !!ngDevMode) {
        global['ngDevMode'] = false;
    }

    _devMode = false;
}
```

It does two things:
- Changes the state of `_devMode` variable to false. Which means that every assertion made with `isDevMode` function will be resolved as false.
- Sets `global['ngDevMode']` to false, and global is just a globally accessible variable where you can set anything you want. In fact sometimes it's set to `window` value.

```typescript
/**
 * @license
 * Copyright Google LLC All Rights Reserved.
 *
 * Use of this source code is governed by an MIT-style license that can be
 * found in the LICENSE file at https://angular.io/license
 */

// TODO(jteplitz602): Load WorkerGlobalScope from lib.webworker.d.ts file #3492
declare var WorkerGlobalScope: any /** TODO #9100 */;
// CommonJS / Node have global context exposed as "global" variable.
// We don't want to include the whole node.d.ts this this compilation unit so we'll just fake
// the global "global" var for now.
declare var global: any /** TODO #9100 */;

const __globalThis = typeof globalThis !== 'undefined' && globalThis;
const __window = typeof window !== 'undefined' && window;
const __self = typeof self !== 'undefined' && typeof WorkerGlobalScope !== 'undefined' &&
    self instanceof WorkerGlobalScope && self;
const __global = typeof global !== 'undefined' && global;

// Always use __globalThis if available, which is the spec-defined global variable across all
// environments, then fallback to __global first, because in Node tests both __global and
// __window may be defined and _global should be __global in that case.
const _global = __globalThis || __global || __window || __self;

/**
 * Attention: whenever providing a new value, be sure to add an
 * entry into the corresponding `....externs.js` file,
 * so that closure won't use that global for its purposes.
 */
export {_global as global};
```

### PlatformRef

After setting the production or the development mode `PlatformRef` object is created.
Function `platformBrowserDynamic` has a type of `(extraProviders?: StaticProvider[]) => PlatformRef`,
so you may expect it to create some very high level providers.
You can treat it as something **above** root.
Services provided in the platform are available for every module bootstrapped with this `PlatformRef`.

I wanted to hide some lines in code below, but at the end, all of these lines are important.

```typescript
/**
 * The Angular platform is the entry point for Angular on a web page.
 * Each page has exactly one platform. Services (such as reflection) which are common
 * to every Angular application running on the page are bound in its scope.
 * A page's platform is initialized implicitly when a platform is created using a platform
 * factory such as `PlatformBrowser`, or explicitly by calling the `createPlatform()` function.
 *
 * @publicApi
 */
@Injectable()
export class PlatformRef {
  private _modules: NgModuleRef<any>[] = [];
  private _destroyListeners: Function[] = [];
  private _destroyed: boolean = false;

  /** @internal */
  constructor(private _injector: Injector) {}

  /**
   * Creates an instance of an `@NgModule` for the given platform.
   *
   * @deprecated Passing NgModule factories as the `PlatformRef.bootstrapModuleFactory` function
   *     argument is deprecated. Use the `PlatformRef.bootstrapModule` API instead.
   */
  bootstrapModuleFactory<M>(moduleFactory: NgModuleFactory<M>, options?: BootstrapOptions):
      Promise<NgModuleRef<M>> {
    // Note: We need to create the NgZone _before_ we instantiate the module,
    // as instantiating the module creates some providers eagerly.
    // So we create a mini parent injector that just contains the new NgZone and
    // pass that as parent to the NgModuleFactory.
    const ngZoneOption = options ? options.ngZone : undefined;
    const ngZoneEventCoalescing = (options && options.ngZoneEventCoalescing) || false;
    const ngZoneRunCoalescing = (options && options.ngZoneRunCoalescing) || false;
    const ngZone = getNgZone(ngZoneOption, {ngZoneEventCoalescing, ngZoneRunCoalescing});
    const providers: StaticProvider[] = [{provide: NgZone, useValue: ngZone}];
    // Note: Create ngZoneInjector within ngZone.run so that all of the instantiated services are
    // created within the Angular zone
    // Do not try to replace ngZone.run with ApplicationRef#run because ApplicationRef would then be
    // created outside of the Angular zone.
    return ngZone.run(() => {
      const ngZoneInjector = Injector.create(
          {providers: providers, parent: this.injector, name: moduleFactory.moduleType.name});
      const moduleRef = <InternalNgModuleRef<M>>moduleFactory.create(ngZoneInjector);
      const exceptionHandler: ErrorHandler|null = moduleRef.injector.get(ErrorHandler, null);
      if (!exceptionHandler) {
        const errorMessage = (typeof ngDevMode === 'undefined' || ngDevMode) ?
            'No ErrorHandler. Is platform module (BrowserModule) included?' :
            '';
        throw new RuntimeError(RuntimeErrorCode.ERROR_HANDLER_NOT_FOUND, errorMessage);
      }
      ngZone!.runOutsideAngular(() => {
        const subscription = ngZone!.onError.subscribe({
          next: (error: any) => {
            exceptionHandler.handleError(error);
          }
        });
        moduleRef.onDestroy(() => {
          remove(this._modules, moduleRef);
          subscription.unsubscribe();
        });
      });
      return _callAndReportToErrorHandler(exceptionHandler, ngZone!, () => {
        const initStatus: ApplicationInitStatus = moduleRef.injector.get(ApplicationInitStatus);
        initStatus.runInitializers();
        return initStatus.donePromise.then(() => {
          // If the `LOCALE_ID` provider is defined at bootstrap then we set the value for ivy
          const localeId = moduleRef.injector.get(LOCALE_ID, DEFAULT_LOCALE_ID);
          setLocaleId(localeId || DEFAULT_LOCALE_ID);
          this._moduleDoBootstrap(moduleRef);
          return moduleRef;
        });
      });
    });
  }

  /**
   * Creates an instance of an `@NgModule` for a given platform.
   *
   * @usageNotes
   * ### Simple Example
   *
   * typescript
   * @NgModule({
   *   imports: [BrowserModule]
   * })
   * class MyModule {}
   *
   * let moduleRef = platformBrowser().bootstrapModule(MyModule);
   * 
   *
   */
  bootstrapModule<M>(
      moduleType: Type<M>,
      compilerOptions: (CompilerOptions&BootstrapOptions)|
      Array<CompilerOptions&BootstrapOptions> = []): Promise<NgModuleRef<M>> {
    const options = optionsReducer({}, compilerOptions);
    return compileNgModuleFactory(this.injector, options, moduleType)
        .then(moduleFactory => this.bootstrapModuleFactory(moduleFactory, options));
  }

  private _moduleDoBootstrap(moduleRef: InternalNgModuleRef<any>): void {
    const appRef = moduleRef.injector.get(ApplicationRef) as ApplicationRef;
    if (moduleRef._bootstrapComponents.length > 0) {
      moduleRef._bootstrapComponents.forEach(f => appRef.bootstrap(f));
    } else if (moduleRef.instance.ngDoBootstrap) {
      moduleRef.instance.ngDoBootstrap(appRef);
    } else {
      const errorMessage = (typeof ngDevMode === 'undefined' || ngDevMode) ?
          `The module ${stringify(moduleRef.instance.constructor)} was bootstrapped, ` +
              `but it does not declare "@NgModule.bootstrap" components nor a "ngDoBootstrap" method. ` +
              `Please define one of these.` :
          '';
      throw new RuntimeError(RuntimeErrorCode.BOOTSTRAP_COMPONENTS_NOT_FOUND, errorMessage);
    }
    this._modules.push(moduleRef);
  }

  /**
   * Registers a listener to be called when the platform is destroyed.
   */
  onDestroy(callback: () => void): void {
    this._destroyListeners.push(callback);
  }

  /**
   * Retrieves the platform {@link Injector}, which is the parent injector for
   * every Angular application on the page and provides singleton providers.
   */
  get injector(): Injector {
    return this._injector;
  }

  /**
   * Destroys the current Angular platform and all Angular applications on the page.
   * Destroys all modules and listeners registered with the platform.
   */
  destroy() {
    if (this._destroyed) {
      const errorMessage = (typeof ngDevMode === 'undefined' || ngDevMode) ?
          'The platform has already been destroyed!' :
          '';
      throw new RuntimeError(RuntimeErrorCode.ALREADY_DESTROYED_PLATFORM, errorMessage);
    }
    this._modules.slice().forEach(module => module.destroy());
    this._destroyListeners.forEach(listener => listener());
    this._destroyed = true;
  }

  get destroyed() {
    return this._destroyed;
  }
}
```

`PlatformRef` has two public methods, `bootstrapModule` and `bootstrapModuleFactory`. I'll start with the first one, because we all call it in our `main.ts` files. It gets two arguments, the first one is the module to bootstrap, and the second one is some kinds of options. One of them in our `ngZone` which allows us to provide `noop` zone.

You should ask me here "So there are other options that I can set globally for my app?".
Yup, as you see the type `CompilerOptions & BootstrapOptions` describes what you can set.
In code below I prepared a short description of what could be set there.

```typescript
{
    /**
     * @deprecated not used at all in Ivy, providing this config option has no effect.
     */
    useJit?: boolean;
    defaultEncapsulation?: ViewEncapsulation;
    providers?: StaticProvider[];
   
    /**
     * @deprecated not used at all in Ivy, providing this config option has no effect.
     */
    missingTranslation?: MissingTranslationStrategy;
    preserveWhitespaces?: boolean;
   
    /**
     * Optionally specify which `NgZone` should be used.
     *
     * - Provide your own `NgZone` instance.
     * - `zone.js` - Use default `NgZone` which requires `Zone.js`.
     * - `noop` - Use `NoopNgZone` which does nothing.
     */
    ngZone?: NgZone|'zone.js'|'noop';

    /**
     * Optionally specify coalescing event change detections or not.
     * Consider the following case.
     *
     * <div (click)="doSomething()">
     *   <button (click)="doSomethingElse()"></button>
     * </div>
     *
     * When button is clicked, because of the event bubbling, both
     * event handlers will be called and 2 change detections will be
     * triggered. We can colesce such kind of events to only trigger
     * change detection only once.
     *
     * By default, this option will be false. So the events will not be
     * coalesced and the change detection will be triggered multiple times.
     * And if this option be set to true, the change detection will be
     * triggered async by scheduling a animation frame. So in the case above,
     * the change detection will only be triggered once.
     */
    ngZoneEventCoalescing?: boolean;

    /**
     * Optionally specify if `NgZone#run()` method invocations should be coalesced
     * into a single change detection.
     *
     * Consider the following case.
     *
     * for (let i = 0; i < 10; i ++) {
     *   ngZone.run(() => {
     *     // do something
     *   });
     * }
     *
     * This case triggers the change detection multiple times.
     * With ngZoneRunCoalescing options, all change detections in an event loop trigger only once.
     * In addition, the change detection executes in requestAnimation.
     *
     */
    ngZoneRunCoalescing?: boolean;
}
```

Method `bootstrapModule` checks provided options and then calls the second method `bootstrapModuleFactory`. Here the fun begins.

The call of function `getNgZone` determines the object that should be used as NgZones implementation.
It's a simple code that will create objects of class `NgZone` or `NoopNgZone`.

I think that it's a subject for the whole new article,
and I will cover differences and implementation of these two types in the future.

```typescript
function getNgZone(
    ngZoneOption: NgZone|'zone.js'|'noop'|undefined,
    extra?: {ngZoneEventCoalescing: boolean, ngZoneRunCoalescing: boolean}): NgZone {
    let ngZone: NgZone;

    if (ngZoneOption === 'noop') {
        ngZone = new NoopNgZone();
    } else {
        ngZone = (ngZoneOption === 'zone.js' ? undefined : ngZoneOption) || new NgZone({
            enableLongStackTrace: typeof ngDevMode === 'undefined' ? false : !!ngDevMode,
            shouldCoalesceEventChangeDetection: !!extra?.ngZoneEventCoalescing,
            shouldCoalesceRunChangeDetection: !!extra?.ngZoneRunCoalescing
        });
    }
    return ngZone;
}
```

Back to the bootstrapping methods.
After creation of the providers for NgZones there are some additional actions on it and moduleRef
is created using a new injector which is chained to platform injector.

```typescript
const moduleRef = <InternalNgModuleRef<M>>moduleFactory.create(ngZoneInjector);
const ngZoneInjector = Injector.create(
    {providers: providers, parent: this.injector, name: moduleFactory.moduleType.name});
```

Few lines down we've got a lot of important lines, but the most important is the call of `_moduleDoBootstrap` method.

```typescript
// ...
  private _moduleDoBootstrap(moduleRef: InternalNgModuleRef<any>): void {
    const appRef = moduleRef.injector.get(ApplicationRef) as ApplicationRef;
    if (moduleRef._bootstrapComponents.length > 0) {
      moduleRef._bootstrapComponents.forEach(f => appRef.bootstrap(f));
    } else if (moduleRef.instance.ngDoBootstrap) {
      moduleRef.instance.ngDoBootstrap(appRef);
    } else {
      const errorMessage = (typeof ngDevMode === 'undefined' || ngDevMode) ?
          `The module ${stringify(moduleRef.instance.constructor)} was bootstrapped, ` +
              `but it does not declare "@NgModule.bootstrap" components nor a "ngDoBootstrap" method. ` +
              `Please define one of these.` :
          '';
      throw new RuntimeError(RuntimeErrorCode.BOOTSTRAP_COMPONENTS_NOT_FOUND, errorMessage);
    }
    this._modules.push(moduleRef);
  }
// ...
```

This method does one of two scenarios:
- If module has `bootstrap` file in `NgModule` decorator, it starts the components using `ApplicationRef.bootstrap` method. This is the default and most common approach that we use in Angular.
- If module implements the `DoBootstrap` interface ([angular.io/api/core/DoBootstrap](https://angular.io/api/core/DoBootstrap)), it calls the `ngDoBootstrap` method. This one method is not very well known. Personally, I think that it's very interesting because it allows you to add some logic from the very beginning of the lifecycle. You can execute some additional business logic, you can call your backend and conditionally bootstrap one of the components. A lot of cool stuff can be created using the `DoBootstrap` interface.

I know that this code is not very easy to understand, but as you see, it's not magic. 

## Router case

I promise you, from this point, we will not see any source code from Angular :P We are focusing on the tutorial again.

Zone-less applications have one very well-known issue that has to be handled.
The router has some troubles with starting.
The fix is very easy, after `NavigationEnd` event it's necessary to trigger the change detection cycle.
I've done it in my `AppComponent`.

It's not complicated, but it's one thing that has to be done, and it's worth remembering.

```typescript
export class AppComponent implements OnDestroy {

  private readonly onDestroy$: Subject<void> = new Subject<void>();

  constructor(
    router: Router,
    applicationRef: ApplicationRef
  ) {
    this.zonelessRouterStarter(router, applicationRef);
  }

  ngOnDestroy() {
    this.onDestroy$.next();
    this.onDestroy$.complete();
  }

  private zonelessRouterStarter(router: Router, applicationRef: ApplicationRef): void {
    router.events
      .pipe(
        filter(event => event instanceof NavigationEnd),
        takeUntil(this.onDestroy$)
      )
      .subscribe(() => {
        applicationRef.tick();
      });
  }

}
```

## What is not working...?

When you have a zone-less application, some build-in things will not work.
We handled the problems with the router.

I work without zones almost all the time in the last few years and to be honest,
I found only one thing that is problematic.
**Async pipe is not working without zone.js**.

Why...? Because of the implementation.
If you want to read more about it,
please check out my first post [Myth: Async pipes are good for performance](https://galczo5.github.io/mythical-angular/posts/async-pipe/).
After all, I consider this pipe as harmful to the performance, so for me, it's not a problem.

## Summary 

Switching from the default cli app to zone-less one is fairly easy.

Basing on my personal experience,
I think that it's better to start without zones than converting when app is almost ready,
and I recommend doing it even for small apps.

I will publish the comparison of the performance very soon, for today that's all.

Thanks for reading!

PS. I know that posted Angular code may be complicated, but I still think that it's worth exploring it.
