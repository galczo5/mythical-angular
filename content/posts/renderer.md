---
title: "Myth: Styling method doesn't matter"
date: 2022-02-21
draft: false
---

Today I'm going to show you how to test different methods of html styling in Angular.
You're going to learn what is `Renderer`, how and when to it and of course how it's implemented in the framework. I'm going to compare different approaches of adding styles to find the best one. You'll see that it's not that simple.

## Angular build in directives
At the very beginning, when you learn about Angular styling. It's very simple in the framework to add style or class dynamically to an html element.

For example. If I want to add background to my div, and value is not static, I can do it like in the code below.

```typescript
import {ChangeDetectionStrategy, Component, Input} from '@angular/core';

@Component({
  selector: 'app-list-item-directive',
  template: `
    <div [style.background]="color">Lorem ipsum dolor sit amet enim.</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListItemDirectiveComponent {

  @Input()
  color: string;

}
```

This way works with classes, attributes and so on. But how it is working under the hood...?
Let's start with analysis of `main.js` located in dist directory after the `ng build` has ended.
It looks very hard to understand, so i decided to make it easier.
Search for function `ListItemDirectiveComponent_Template`.
Basically, every template you define in Angular is transformed into a function in the javascript code.
Our background styles are added using something called `ɵɵstyleProp`;

```javascript
ListItemDirectiveComponent.ɵcmp = /*@__PURE__*/ _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵdefineComponent"]({ type: ListItemDirectiveComponent, selectors: [["app-list-item-directive"]], inputs: { color: "color" }, decls: 2, vars: 2, template: function ListItemDirectiveComponent_Template(rf, ctx) { if (rf & 1) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](0, "div");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](1, "Lorem ipsum dolor sit amet enim.");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
    } if (rf & 2) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵstyleProp"]("background", ctx.color);
    } }, encapsulation: 2, changeDetection: 0 });
```

To check it further, you have a few options. 
You can try to mess with the Angular sources and follow the usages of `ɵɵstyleProp`. 
Second one, that I used in ths case is to put `debugger` and above line contains `_angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵstyleProp"]("background", ctx.color);`. 
After few round of step over and step into actions in the dev tools I found `applyStyling` function.

```typescript
/**
 * Writes class/style to element.
 *
 * @param renderer Renderer to use.
 * @param isClassBased `true` if it should be written to `class` (`false` to write to `style`)
 * @param rNode The Node to write to.
 * @param prop Property to write to. This would be the class/style name.
 * @param value Value to write. If `null`/`undefined`/`false` this is considered a remove (set/add
 *        otherwise).
 */
export function applyStyling(
    renderer: Renderer3, isClassBased: boolean, rNode: RElement, prop: string, value: any) {
  const isProcedural = isProceduralRenderer(renderer);
  if (isClassBased) {
    // We actually want JS true/false here because any truthy value should add the class
    if (!value) {
      ngDevMode && ngDevMode.rendererRemoveClass++;
      if (isProcedural) {
        (renderer as Renderer2).removeClass(rNode, prop);
      } else {
        (rNode as HTMLElement).classList.remove(prop);
      }
    } else {
      ngDevMode && ngDevMode.rendererAddClass++;
      if (isProcedural) {
        (renderer as Renderer2).addClass(rNode, prop);
      } else {
        ngDevMode && assertDefined((rNode as HTMLElement).classList, 'HTMLElement expected');
        (rNode as HTMLElement).classList.add(prop);
      }
    }
  } else {
    let flags = prop.indexOf('-') === -1 ? undefined : RendererStyleFlags2.DashCase as number;
    if (value == null /** || value === undefined */) {
      ngDevMode && ngDevMode.rendererRemoveStyle++;
      if (isProcedural) {
        (renderer as Renderer2).removeStyle(rNode, prop, flags);
      } else {
        (rNode as HTMLElement).style.removeProperty(prop);
      }
    } else {
      // A value is important if it ends with `!important`. The style
      // parser strips any semicolons at the end of the value.
      const isImportant = typeof value === 'string' ? value.endsWith('!important') : false;

      if (isImportant) {
        // !important has to be stripped from the value for it to be valid.
        value = value.slice(0, -10);
        flags! |= RendererStyleFlags2.Important;
      }

      ngDevMode && ngDevMode.rendererSetStyle++;
      if (isProcedural) {
        (renderer as Renderer2).setStyle(rNode, prop, value, flags);
      } else {
        ngDevMode && assertDefined((rNode as HTMLElement).style, 'HTMLElement expected');
        (rNode as HTMLElement).style.setProperty(prop, value, isImportant ? 'important' : '');
      }
    }
  }
}
```

Looks like `isProcedural` is the key here.
If it's true, `Renderer2.setStyle` method is used, if not style is set on the reference to the element.

Checking if the renderer is procedural is quite simple.
It's procedural if there is a `listen` method.
In our case, renderer is procedural.

```typescript
/** Returns whether the `renderer` is a `ProceduralRenderer3` */
export function isProceduralRenderer(renderer: ProceduralRenderer3|
                                     ObjectOrientedRenderer3): renderer is ProceduralRenderer3 {
  return !!((renderer as any).listen);
}
```

So at the end,
Angular is using `Renderer2.setStyle` method to set style when we are using build in directives and properties.

## Renderer2

Second way of adding styles to the element is using `Renderer2` directly on the html elements.
This object can be injected into a Component or Directive class. You can use it like in the code example below. 

```typescript
import {
  ChangeDetectionStrategy,
  Component,
  ElementRef,
  Input,
  OnDestroy,
  OnInit,
  Renderer2,
  ViewChild
} from '@angular/core';
import {BackgroundService} from '../background.service';
import {Subject} from "rxjs";
import {takeUntil} from "rxjs/operators";

@Component({
  selector: 'app-list-item-renderer',
  template: `
    <div #item>Lorem ipsum dolor sit amet enim.</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListItemRendererComponent implements OnInit, OnDestroy {

  @ViewChild('item', { static: true, read: ElementRef })
  item: ElementRef;

  @Input()
  id: number;

  private readonly onDestroy$: Subject<void> = new Subject<void>();

  constructor(private backgroundService: BackgroundService,
              private renderer: Renderer2) {
  }

  ngOnInit(): void {
    this.backgroundService.get(this.id)
      .pipe(takeUntil(this.onDestroy$))
      .subscribe(color => {
        this.renderer.setStyle(this.item.nativeElement, 'background', color);
      });
  }

  ngOnDestroy() {
    this.onDestroy$.next();
    this.onDestroy$.complete();
  }
}
```

In fact `Renderer2` class is an abstract class implemented by few other classes.
**I decided to remove description comments from the code, because it was so long. If you want to check the whole class, you can do it here: [packages/core/src/render/api.ts](https://github.com/angular/angular/blob/8c71b9fc42e1aa898200b2b3ebb54d6faf0fa645/packages/core/src/render/api.ts#L66).**

**Please, notice that it's a link to GitHub sources, so it may be changed by the time.**
```typescript
export abstract class Renderer2 {
  abstract get data(): {[key: string]: any};
  abstract destroy(): void;
  abstract createElement(name: string, namespace?: string|null): any;
  abstract createComment(value: string): any;
  abstract createText(value: string): any;
  destroyNode!: ((node: any) => void)|null;
  abstract appendChild(parent: any, newChild: any): void;
  abstract insertBefore(parent: any, newChild: any, refChild: any, isMove?: boolean): void;
  abstract removeChild(parent: any, oldChild: any, isHostElement?: boolean): void;
  abstract selectRootElement(selectorOrNode: string|any, preserveContent?: boolean): any;
  abstract parentNode(node: any): any;
  abstract nextSibling(node: any): any;
  abstract setAttribute(el: any, name: string, value: string, namespace?: string|null): void;
  abstract removeAttribute(el: any, name: string, namespace?: string|null): void;
  abstract addClass(el: any, name: string): void;
  abstract removeClass(el: any, name: string): void;
  abstract setStyle(el: any, style: string, value: any, flags?: RendererStyleFlags2): void;
  abstract removeStyle(el: any, style: string, flags?: RendererStyleFlags2): void;
  abstract setProperty(el: any, name: string, value: any): void;
  abstract setValue(node: any, value: string): void;
  abstract listen(
      target: 'window'|'document'|'body'|any, eventName: string,
      callback: (event: any) => boolean | void): () => void;
  static __NG_ELEMENT_ID__: () => Renderer2 = () => injectRenderer2();
}
```

And here you have a list of implementations that I found in the sources.
```text
AnimationRenderer
BaseAnimationRenderer
DefaultDomRenderer2
DefaultServerRenderer2
EmulatedEncapsulationDomRenderer2
EmulatedEncapsulationServerRenderer2
ShadowDomRenderer
```

I encourage you to check how every one of these works, but in my case `DefaultDomRenderer2` was used, I'm going to focus on that one. 

Implementation is quite easy and I decided to place it in the code snippet below.

```typescript
class DefaultDomRenderer2 implements Renderer2 {
  data: {[key: string]: any} = Object.create(null);

  constructor(private eventManager: EventManager) {}

  destroy(): void {}

  destroyNode = null;

  createElement(name: string, namespace?: string): any {
    if (namespace) {
      // TODO: `|| namespace` was added in
      // https://github.com/angular/angular/commit/2b9cc8503d48173492c29f5a271b61126104fbdb to
      // support how Ivy passed around the namespace URI rather than short name at the time. It did
      // not, however extend the support to other parts of the system (setAttribute, setAttribute,
      // and the ServerRenderer). We should decide what exactly the semantics for dealing with
      // namespaces should be and make it consistent.
      // Related issues:
      // https://github.com/angular/angular/issues/44028
      // https://github.com/angular/angular/issues/44883
      return document.createElementNS(NAMESPACE_URIS[namespace] || namespace, name);
    }

    return document.createElement(name);
  }

  createComment(value: string): any {
    return document.createComment(value);
  }

  createText(value: string): any {
    return document.createTextNode(value);
  }

  appendChild(parent: any, newChild: any): void {
    parent.appendChild(newChild);
  }

  insertBefore(parent: any, newChild: any, refChild: any): void {
    if (parent) {
      parent.insertBefore(newChild, refChild);
    }
  }

  removeChild(parent: any, oldChild: any): void {
    if (parent) {
      parent.removeChild(oldChild);
    }
  }

  selectRootElement(selectorOrNode: string|any, preserveContent?: boolean): any {
    let el: any = typeof selectorOrNode === 'string' ? document.querySelector(selectorOrNode) :
                                                       selectorOrNode;
    if (!el) {
      throw new Error(`The selector "${selectorOrNode}" did not match any elements`);
    }
    if (!preserveContent) {
      el.textContent = '';
    }
    return el;
  }

  parentNode(node: any): any {
    return node.parentNode;
  }

  nextSibling(node: any): any {
    return node.nextSibling;
  }

  setAttribute(el: any, name: string, value: string, namespace?: string): void {
    if (namespace) {
      name = namespace + ':' + name;
      const namespaceUri = NAMESPACE_URIS[namespace];
      if (namespaceUri) {
        el.setAttributeNS(namespaceUri, name, value);
      } else {
        el.setAttribute(name, value);
      }
    } else {
      el.setAttribute(name, value);
    }
  }

  removeAttribute(el: any, name: string, namespace?: string): void {
    if (namespace) {
      const namespaceUri = NAMESPACE_URIS[namespace];
      if (namespaceUri) {
        el.removeAttributeNS(namespaceUri, name);
      } else {
        el.removeAttribute(`${namespace}:${name}`);
      }
    } else {
      el.removeAttribute(name);
    }
  }

  addClass(el: any, name: string): void {
    el.classList.add(name);
  }

  removeClass(el: any, name: string): void {
    el.classList.remove(name);
  }

  setStyle(el: any, style: string, value: any, flags: RendererStyleFlags2): void {
    if (flags & (RendererStyleFlags2.DashCase | RendererStyleFlags2.Important)) {
      el.style.setProperty(style, value, flags & RendererStyleFlags2.Important ? 'important' : '');
    } else {
      el.style[style] = value;
    }
  }

  removeStyle(el: any, style: string, flags: RendererStyleFlags2): void {
    if (flags & RendererStyleFlags2.DashCase) {
      el.style.removeProperty(style);
    } else {
      // IE requires '' instead of null
      // see https://github.com/angular/angular/issues/7916
      el.style[style] = '';
    }
  }

  setProperty(el: any, name: string, value: any): void {
    NG_DEV_MODE && checkNoSyntheticProp(name, 'property');
    el[name] = value;
  }

  setValue(node: any, value: string): void {
    node.nodeValue = value;
  }

  listen(target: 'window'|'document'|'body'|any, event: string, callback: (event: any) => boolean):
      () => void {
    NG_DEV_MODE && checkNoSyntheticProp(event, 'listener');
    if (typeof target === 'string') {
      return <() => void>this.eventManager.addGlobalEventListener(
          target, event, decoratePreventDefault(callback));
    }
    return <() => void>this.eventManager.addEventListener(
               target, event, decoratePreventDefault(callback)) as () => void;
  }
}
```

As you see, it's using a native way to add styles to the elements.

## Native way

I see no reason to describe it using a very long and complicated description.
Just check the code bellow, it's easy to understand.

```typescript
import {
  ChangeDetectionStrategy,
  Component,
  ElementRef,
  Input,
  OnDestroy,
  OnInit,
  ViewChild
} from '@angular/core';
import {BackgroundService} from '../background.service';
import {Subject} from "rxjs";
import {takeUntil} from "rxjs/operators";

@Component({
  selector: 'app-list-item-native',
  template: `
    <div #item>Lorem ipsum dolor sit amet enim.</div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class ListItemNativeComponent implements OnInit, OnDestroy {

  @ViewChild('item', { static: true, read: ElementRef })
  item: ElementRef;

  @Input()
  id: number;

  private readonly onDestroy$: Subject<void> = new Subject<void>();

  constructor(private backgroundService: BackgroundService) { }

  ngOnInit(): void {
    this.backgroundService.get(this.id)
      .pipe(takeUntil(this.onDestroy$))
      .subscribe(color => {
        this.item.nativeElement.style.background = color;
      });
  }

  ngOnDestroy() {
    this.onDestroy$.next();
    this.onDestroy$.complete();
  }
}
```

## Test app

I described three ways of styling in the Angular framework. So which one is the best/most efficient? Unfortunately, it's not that simple.

To demonstrate that I created an app, you can download the sources from my GitHub [github.com/galczo5/experiment-styles-performance](https://github.com/galczo5/experiment-styles-performance).

For your convenience, I hosted it and placed an instance in the iframe bellow.

In the app I created three columns with the thousands of the elements. Each column is styled with a different method. You can click on three buttons.

First one is toggling the background style in all of the elements in that column.

Second one is toggling the background style in half of the elements.

Last button is toggling the background style in only one element from the list.

Do the tests with the dev tools open. There are logs with a precise time measurement for each action.

{{< rawhtml >}}
<div style="height: 400px; width: 1300px; border: 1px solid lightgray;">
    <iframe style="border: 0; width: 100%; height: 100%;" src="https://galczo5.github.io/experiment-styles-performance/"></iframe>
</div>
{{< /rawhtml >}}

With my setup, I had results like in the table below.

|                 | Directive | Renderer | Native |
|-----------------|-----------|--------|--------|
| **Change all**  | 277ms     | 5155ms | 5485ms |
| **Change half** | 164ms     | 2597ms | 2835ms | 
| **Change one**  | 54ms      | 11ms   | 11ms   |

## Conclusion

There is no one the best way to add styles to the html elements. 
It really depends on what you want to achieve.

Why changing the styles in one element is more efficient with the `Renderer2`...? It's easy. There is no overhead of framework checks etc. 

Why changing the styles in multiple elements is faster with directives...? Well, in that approach we avoid multiple repaints and possible layout trashing problems.

We can generalize the rule to:
- When you want to change a lot of styles at the same moment, use build-in Angular directive/properties.
- When you want to change only one style, it's better to use `Renderer2` service.

