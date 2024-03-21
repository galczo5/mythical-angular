---
title: "What do we know about signal inputs limitations"
date: 2024-03-20
draft: false
---

Not so long ago, we got a new feature for Angular. We can now create inputs using a simple function that returns a signal. It looks nice when the code does not contain the `@Input()` decorator, but what's more important, it's using a new reactivity approach powered by signals.

```typescript
text = input('default value');

//...vs

@Input()
text2 = 'default value';
```

Let's take a closer look how this new feature works and what are the limitations of it.

## How it works

The primary usage is quite simple. To create a new input in your component, all you need to do is add a new field and initialize it with the result of the `input(devault_value)` function.

```typescript
import {Component, input} from '@angular/core';

@Component({
  selector: 'app-child',
  standalone: true,
  template: `
    <strong>Child</strong>
    {{ text() }}
  `
})
export class ChildComponent {
  text = input('default value');
}

// ---

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [ChildComponent],
  template: `
    <strong>Parent</strong>
    <app-child [text]="text"/>
  `
})
export class AppComponent {
  text = 'value from parent';
}
```

The first advantage is that by calling `input()` function you need to specify the default value.

It's important because if you design your component well and take care of these default values, you will achieve a better code. I mean, it's way easier with inputs of type `string` than inputs of type `string | null` or `string | undefined`. In the second case, you'll need to check for null or undefined values every time when you access the value. It adds a lot of unnecessary if statements in your code.

### But how does it work under the hood...?

So, first of all let's check the code of `input()` function in Angular repository.

```typescript
// packages/core/src/authoring/input/input.ts

export function inputFunction<ReadT, WriteT>(
    initialValue?: ReadT,
    opts?: InputOptions<ReadT, WriteT>): InputSignalWithTransform<ReadT|undefined, WriteT> {
    return createInputSignal(initialValue, opts);
}

// ...

export function createInputSignal<ReadT, WriteT>(
    initialValue: ReadT,
    options?: InputOptions<ReadT, WriteT>): InputSignalWithTransform<ReadT, WriteT> {
    const node: InputSignalNode<ReadT, WriteT> = Object.create(INPUT_SIGNAL_NODE);

    node.value = initialValue;

    // Perf note: Always set `transformFn` here to ensure that `node` always
    // has the same v8 class shape, allowing monomorphic reads on input signals.
    node.transformFn = options?.transform;

    function inputValueFn() {
        // Record that someone looked at this signal.
        producerAccessed(node);

        if (node.value === REQUIRED_UNSET_VALUE) {
            throw new RuntimeError(
                RuntimeErrorCode.REQUIRED_INPUT_NO_VALUE,
                ngDevMode && 'Input is required but no value is available yet.');
        }

        return node.value;
    }

    (inputValueFn as any)[SIGNAL] = node;

    if (ngDevMode) {
        inputValueFn.toString = () => `[Input Signal: ${inputValueFn()}]`;
    }

    return inputValueFn as InputSignalWithTransform<ReadT, WriteT>;
}
```

Before I started to investigate the subject, I was sure that this function was using the `inject()` function to do awesome things with it. I was wrong.

As you can see, `inputFunction` is calling `createInputSignal` function, and the only thing that is happening inside is a simple code that creates a `node`, assign `initialValue` and `transformFn` and then create a signal.

So, how is Angular able to recognize a field created with this function as component input...?

The answer is simpler than I thought. All is done by ahead-of-time compilation.

To check it let's execute `yarn build` and check what is inside the `dist` directory.

```text
> ~/D/experiment-signal-input main yarn build
yarn run v1.22.21
$ ng build
⠙ Building...
Initial chunk files   | Names         |  Raw size | Estimated transfer size
main-PYYQRTEG.js      | main          | 169.39 kB |                45.93 kB
polyfills-RT5I6R6G.js | polyfills     |  33.10 kB |                10.72 kB
styles-5INURTSO.css   | styles        |   0 bytes |                 0 bytes

                      | Initial total | 202.50 kB |                56.64 kB
Output location: experiment-signal-input/dist/experiment-signal-input

Application bundle generation complete. [2.581 seconds]
✨  Done in 3.42s.
```

In my example, the important code should be in the `main-PYYQRTEG.js`.

```javascript
var Py = (() => {
  let t = class t {
    constructor() {
      this.text = Tu("default value")
    }
  };
  t.\u0275fac = function (o) {
    return new (o || t)
  }, t.\u0275cmp = ln({
    type: t,
    selectors: [["app-child"]],
    inputs: {text: [we.SignalBased, "text"]},
    standalone: !0,
    features: [yn],
    decls: 3,
    vars: 1,
    template: function (o, i) {
      o & 1 && (gn(0, "strong"), Zr(1, "Child"), mn(), Zr(2)), o & 2 && (Qi(2), is(" ", i.text(), " "))
    },
    encapsulation: 2
  });
  let e = t;
  return e
})(), Ad = (() => {
  let t = class t {
    constructor() {
      this.text = "value from parent"
    }
  };
  t.\u0275fac = function (o) {
    return new (o || t)
  }, t.\u0275cmp = ln({
    type: t,
    selectors: [["app-root"]],
    standalone: !0,
    features: [yn],
    decls: 3,
    vars: 1,
    consts: [[3, "text"]],
    template: function (o, i) {
      o & 1 && (gn(0, "strong"), Zr(1, "Parent"), mn(), vn(2, "app-child", 0)), o & 2 && (Qi(2), os("text", i.text))
    },
    dependencies: [Py],
    encapsulation: 2
  });
  let e = t;
  return e
})();
```

I know it's not easy to read, but that's what it looks like after optimization.

In this code `Py` is our `ChildComponent`, `Ad` is `AppComponent`. **As you can see, inputs are added during Angular compilation.** It's important, because it means that inputs are not resolved in runtime, and you cannot easily create a custom "clone" of `input` function.

To be honest, I shouldn't be surprised. We have ahead-of-time compilation for a reason.

## What is not working and why it's important

Now that we know that the `input()` function is used by the compiler to add information about inputs during the compilation process, we should talk about limitations. **Compiler is not running your code, so you need to use `input()` in a very specific way.**

Let's check what is not working.

### Inputs in constructors

The first example of code when `input()` will not work is presented below.

```typescript
import {Component, input} from '@angular/core';

@Component({
  selector: 'app-child',
  standalone: true,
  template: `
    <strong>Child</strong>
    {{ text() }}
  `
})
export class ChildComponent {
  text = input('default value');

  constructor() {
    const fullname = input('');
  }
}

// result

var Py = (() => {
    let t = class t {
        constructor() {
            this.text = Si("default value");
            let r = Si("")
        }
    };
    t.\u0275fac = function (o) {
        return new (o || t)
    }, t.\u0275cmp = ln({
        type: t,
        selectors: [["app-child"]],
        inputs: {text: [we.SignalBased, "text"]},
        standalone: !0,
        features: [yn],
        decls: 3,
        vars: 1,
        template: function (o, i) {
            o & 1 && (gn(0, "strong"), Zr(1, "Child"), mn(), Zr(2)), o & 2 && (Ki(2), ss(" ", i.text(), " "))
        },
        encapsulation: 2
    });
    let e = t;
    return e
})()
```

After adding a code to the constructor, we still have only one input. Let's try to change it `fullname` from local variable to a field and test it again.

```typescript
import {Component, input, InputSignal} from '@angular/core';

@Component({
  selector: 'app-child',
  standalone: true,
  template: `
    <strong>Child</strong>
    {{ text() }}
  `
})
export class ChildComponent {
  text = input('default value');
  fullname!: InputSignal<string>;

  constructor() {
    this.fullname = input('');
  }
}

// result

var Py = (() => {
    let t = class t {
        constructor() {
            this.text = Si("default value"), this.fullname = Si("")
        }
    };
    t.\u0275fac = function (o) {
        return new (o || t)
    }, t.\u0275cmp = ln({
        type: t,
        selectors: [["app-child"]],
        inputs: {text: [we.SignalBased, "text"]},
        standalone: !0,
        features: [yn],
        decls: 3,
        vars: 1,
        template: function (o, i) {
            o & 1 && (gn(0, "strong"), Zr(1, "Child"), mn(), Zr(2)), o & 2 && (Ki(2), ss(" ", i.text(), " "))
        },
        encapsulation: 2
    });
    let e = t;
    return e
})()
```

Still no change. Input was not added; even in the code, it looks like something that might work.

### Inputs in functions

The second non-working case is when you try to wrap input in other functions.

```typescript
import {Component, input} from '@angular/core';

function addInputs() {
  return input('');
}

@Component({
  selector: 'app-child',
  standalone: true,
  template: `
    <strong>Child</strong>
    {{ text() }}
  `
})
export class ChildComponent {
  text = input('default value');
  fullname = addInputs();
}

// result

var Fy = (() => {
    let t = class t {
        constructor() {
            this.text = Si("default value"), this.fullname = Py()
        }
    };
    t.\u0275fac = function (o) {
        return new (o || t)
    }, t.\u0275cmp = ln({
        type: t,
        selectors: [["app-child"]],
        inputs: {text: [we.SignalBased, "text"]},
        standalone: !0,
        features: [yn],
        decls: 3,
        vars: 1,
        template: function (o, i) {
            o & 1 && (gn(0, "strong"), Zr(1, "Child"), mn(), Zr(2)), o & 2 && (Ki(2), ss(" ", i.text(), " "))
        },
        encapsulation: 2
    });
    let e = t;
    return e
})()
```

As you can see, there is no additional input. Even if you try to inline the function, compiler will not recognize a new input.

```typescript
@Component({/* ... */})
export class ChildComponent {
    text = input('default value');
    fullname = (() => input(''))();
}
```

This case is quite important for me. When I heard about signal inputs for the first time, I was quite sure that it will allow me to wrap them in a function and create a smart composite signals.

For example:

```typescript
export function fullname() {
    const firstname = input('');
    const lastname = input('');

    return computed(() => firstname() + ' ' + lastname());
}
```

Would be nice of this function to add two new inputs and encapsulate some business logic inside, so I can easily reuse it in my components. I can imagine, that with synergy with `inject()` it would let you build a very useful code.

Unfortunately, as we proved, it's not working. At least in Angular 17.2, that I used for my tests.

## Summary

I believe that `inject()` function is a very nice and clean looking way to add inputs to your components. It has a lot to offer, but you're obligated to use it in "correct" way.

For now, it won't let you build more complex functions that you can reuse in your components. It's worth remembering that it works thanks to the Angular compiler, and it's not adding inputs in runtime. 
