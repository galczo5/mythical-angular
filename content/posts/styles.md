---
title: "Myth: Component styles"
date: 2022-03-10
draft: false
---

This is a very short and interesting subject to me.
In Angular every component can have its own attached styles.
Styles can be in the same file or in separated files, similar to the templates.

In addition, we can encapsulate styles.
There are two methods of encapsulation: `Emulated` and `ShadowDom`.
This article describes both types of encapsulation, and its influence on the performance of the whole app.

## Using `styles` instead of `styleUrls` is good for performance

Just before we start with the "complicated" stuff,
I want to mention one of the myths connected to the place where you can put your style.

A Few times, I head that keeping styles in the file with logic and the template.
It's easy to check and verify this myth.

First, I created two components with code like bellow.

```typescript
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-styles',
  template: `<p>styles works!</p>`,
  styles: [`
    .app-styles {
      display: flex;
    }
  `]
})
export class StylesComponent implements OnInit {

  constructor() { }

  ngOnInit(): void {
  }

}
```

I see no reason to paste here the content of the `./style-urls.component.css` file.
Believe me that it looks almost the same `.app-styles` css class like in the code above.
It's not that important in our investigation.

```typescript
import { Component, OnInit } from '@angular/core';

@Component({
  selector: 'app-style-urls',
  template: `<p>style-urls works!</p>`,
  styleUrls: ['./style-urls.component.css']
})
export class StyleUrlsComponent implements OnInit {

  constructor() { }

  ngOnInit(): void {
  }

}
```

Nothing special, right...?
So, after that I have disabled minification of the artifacts in `angular.json` file and run `ng build`.
This command should produce javascript code in `/dist` directory, and for us the most interesting file is `main.js`.

Using your IDE, you can search for lines with `StylesComponent` and `StyleUrlsComponent` words.

```javascript
StylesComponent.ɵcmp = /*@__PURE__*/ _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵdefineComponent"]({ type: StylesComponent, selectors: [["app-styles"]], decls: 2, vars: 0, template: function StylesComponent_Template(rf, ctx) { if (rf & 1) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](0, "p");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](1, " styles works! ");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
    } }, styles: [".app-styles[_ngcontent-%COMP%] {\n      display: flex;\n    }"] });

// ...

StyleUrlsComponent.ɵcmp = /*@__PURE__*/ _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵdefineComponent"]({ type: StyleUrlsComponent, selectors: [["app-style-urls"]], decls: 2, vars: 0, template: function StyleUrlsComponent_Template(rf, ctx) { if (rf & 1) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](0, "p");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](1, " style-urls works! ");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
    } }, styles: [".app-style-urls[_ngcontent-%COMP%] {\n  display: flex;\n}"] });
```

Yup, the same result for `styles` and `styleUrls`.
Do we need better proof that it's a myth and `styles` is not faster than `styleUrls`...?

Now we can dig deeper into an encapsulation subject.

## Encapsulation

To check the differences between the encapsulation types, I'm using the same method as in the previous chapter of this article.
I've generated three components, one for each possible value from `ViewEncapsulation` enum, I've changed setting in the `angular.json` and I checked the result.

```typescript
export enum ViewEncapsulation {
  Emulated = 0,
  // Historically the 1 value was for `Native` encapsulation which has been removed as of v11.
  None = 2,
  ShadowDom = 3
}
```

After the building, javascript should look like in the code snippet bellow.

```javascript
NoEncapsulationComponent.ɵcmp = /*@__PURE__*/ _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵdefineComponent"]({ type: NoEncapsulationComponent, selectors: [["app-no-encapsulation"]], decls: 2, vars: 0, template: function NoEncapsulationComponent_Template(rf, ctx) { if (rf & 1) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](0, "p");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](1, " no-encapsulation works! ");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
    } }, styles: ["\n    .app-no-encapsulation {\n      display: flex;\n    }\n  "], encapsulation: 2 });

// ...

EmulatedComponent.ɵcmp = /*@__PURE__*/ _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵdefineComponent"]({ type: EmulatedComponent, selectors: [["app-emulated"]], decls: 2, vars: 0, template: function EmulatedComponent_Template(rf, ctx) { if (rf & 1) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](0, "p");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](1, " emulated works! ");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
    } }, styles: [".app-emulated[_ngcontent-%COMP%] {\n      display: flex;\n    }"] });

// ...

ShadowComponent.ɵcmp = /*@__PURE__*/ _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵdefineComponent"]({ type: ShadowComponent, selectors: [["app-shadow"]], decls: 2, vars: 0, template: function ShadowComponent_Template(rf, ctx) { if (rf & 1) {
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementStart"](0, "p");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵtext"](1, " shadow works! ");
        _angular_core__WEBPACK_IMPORTED_MODULE_0__["ɵɵelementEnd"]();
    } }, styles: ["\n    .app-shadow {\n      display: flex;\n    }\n  "], encapsulation: 3 });
```

The most visible difference is of course `[_ngcontent-%COMP%]` part.
It was generated by the compiler, but why...?

To check this, we again have to go back to the `Renderer2` class. If you're new, check my previous post about it.

For each component, I've checked the `Renderer2` implementation. The results were: 
- `ViewEncapsulation.Emulated` = `EmulatedEncapsulationDomRenderer2`
- `ViewEncapsulation.None` = `DefaultDomRenderer2`
- `ViewEncapsulation.ShadowDom` = `ShadowDomRenderer`

### `EmulatedEncapsulationDomRenderer2`

OK.
We know that for every component with encapsulation set to `Emulated` Angular generates something that was not the in source code.
From CSS, we know that added part makes that elements with class `app-emulated` and attribute will use this style.
It looks suspicious, but it's nothing more than HTML attributed with a strange name. 

```css
.app-emulated[_ngcontent-%COMP%] {
    display: flex;
}
```

Knowing that we may expect that Angular adds that attribute in runtime. In fact, it does.
Check out the code of `EmulatedEncapsulationDomRenderer2` which is used by framework for emulated encapsulation components.

Two parts of that code are important.
First, in constructor, class is generating property `contentAttr`.
This field is used later in overridden method `createElement`,
every element created by this implementation of `Renderer2` will add this special attribute by default.

```typescript
class EmulatedEncapsulationDomRenderer2 extends DefaultDomRenderer2 {
  private contentAttr: string;
  private hostAttr: string;

  constructor(
      eventManager: EventManager, sharedStylesHost: DomSharedStylesHost,
      private component: RendererType2, appId: string) {
    super(eventManager);
    const styles = flattenStyles(appId + '-' + component.id, component.styles, []);
    sharedStylesHost.addStyles(styles);

    this.contentAttr = shimContentAttribute(appId + '-' + component.id);
    this.hostAttr = shimHostAttribute(appId + '-' + component.id);
  }

  applyToHost(element: any) {
    super.setAttribute(element, this.hostAttr, '');
  }

  override createElement(parent: any, name: string): Element {
    const el = super.createElement(parent, name);
    super.setAttribute(el, this.contentAttr, '');
    return el;
  }
}
```

I answered where is the code responsible for adding that attribute, but still we have one important question: WHY?

Basically, it's very simple.
Angular is adding this attribute to CSS code and in runtime to the components,
it is a very smart and native way
to make sure that added style will be applied only to elements from Angular app scope.
You simply aren't able to use it somewhere outside app, for example,
directly in `index.html` file, because you cannot know the name of the attribute before build.

As I said, a very smart way of encapsulating styles.

### `ShadowDomRenderer`

A different approach to encapsulation is to use Shadow DOM.
To be honest, maybe I live in a very monotonic environment,
or it's very rarely used, because I saw usages of this strategy only a few times yet.
The whole idea consumes the Shadow DOM API of the browser,
right now; according to [caniuse.com](https://caniuse.com/) almost all modern browsers are supporting it. 

`ShadowDomRenderer` in constructor is attaching shadow to the host, and later it appends styles to created shadow root. There is no need to create weird named attributes or something, just some javascript code.

```typescript
class ShadowDomRenderer extends DefaultDomRenderer2 {
  private shadowRoot: any;

  constructor(
      eventManager: EventManager, private sharedStylesHost: DomSharedStylesHost,
      private hostEl: any, component: RendererType2) {
    super(eventManager);
    this.shadowRoot = (hostEl as any).attachShadow({mode: 'open'});
    this.sharedStylesHost.addHost(this.shadowRoot);
    const styles = flattenStyles(component.id, component.styles, []);
    for (let i = 0; i < styles.length; i++) {
      const styleEl = document.createElement('style');
      styleEl.textContent = styles[i];
      this.shadowRoot.appendChild(styleEl);
    }
  }

  private nodeOrShadowRoot(node: any): any {
    return node === this.hostEl ? this.shadowRoot : node;
  }

  override destroy() {
    this.sharedStylesHost.removeHost(this.shadowRoot);
  }

  override appendChild(parent: any, newChild: any): void {
    return super.appendChild(this.nodeOrShadowRoot(parent), newChild);
  }
  override insertBefore(parent: any, newChild: any, refChild: any): void {
    return super.insertBefore(this.nodeOrShadowRoot(parent), newChild, refChild);
  }
  override removeChild(parent: any, oldChild: any): void {
    return super.removeChild(this.nodeOrShadowRoot(parent), oldChild);
  }
  override parentNode(node: any): any {
    return this.nodeOrShadowRoot(super.parentNode(this.nodeOrShadowRoot(node)));
  }
}
```

I decided not to focus on this strategy a lot, because as I mentioned earlier, I'm not sure if it's very popular.

If you want to invest some time in learning how Shadow DOM is working, here you have a very good article from MDN; [developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_shadow_DOM).

## How styles are added in the runtime

Another interesting part of styling that is worth knowing is part of adding styles to the page by the framework.
Did you ever wonder how Angular is doing that?
Are the styles loaded eagerly using HTML `link` tag?
Maybe some kind of dynamic/lazy loading?

To calm you, of course, component styles are not loading eagerly. It would be potentially easy to break the performance.

Once again, I'm going to paste some code from Angular sources :) Please forgive me that,
but I still think that you should not trust me.
Remember: "Trust the code".

This time we start again with `EmulatedEncapsulationDomRenderer2`.
In the constructor we have very few things injected and,
one of the arguments is an instance of class `DomSharedStylesHost`.
In the third line of the constructor method `addStyles` is called.
This is our investigation entry point.

```typescript
class EmulatedEncapsulationDomRenderer2 extends DefaultDomRenderer2 {
  private contentAttr: string;
  private hostAttr: string;

  constructor(
      eventManager: EventManager, sharedStylesHost: DomSharedStylesHost,
      private component: RendererType2, appId: string) {
    super(eventManager);
    const styles = flattenStyles(appId + '-' + component.id, component.styles, []);
    sharedStylesHost.addStyles(styles);

    this.contentAttr = shimContentAttribute(appId + '-' + component.id);
    this.hostAttr = shimHostAttribute(appId + '-' + component.id);
  }
  
  // ...
}
```

And of course, code for it can be found in the sources.
It's important to know that `DomSharedStylesHost` extends the class `SharedStylesHost`,
there is another class that extends it `ServerStylesHost`.
We can assume that this operation is implemented differently for server side rendering configuration.

As always I want to encourage you to check it,
but for now I'll focus on the browser version, because it's probably the most commonly used implementation.

```typescript
@Injectable()
export class DomSharedStylesHost extends SharedStylesHost implements OnDestroy {
  // Maps all registered host nodes to a list of style nodes that have been added to the host node.
  private _hostNodes = new Map<Node, Node[]>();

  constructor(@Inject(DOCUMENT) private _doc: any) {
    super();
    this._hostNodes.set(_doc.head, []);
  }

  private _addStylesToHost(styles: Set<string>, host: Node, styleNodes: Node[]): void {
    styles.forEach((style: string) => {
      const styleEl = this._doc.createElement('style');
      styleEl.textContent = style;
      styleNodes.push(host.appendChild(styleEl));
    });
  }

  addHost(hostNode: Node): void {
    const styleNodes: Node[] = [];
    this._addStylesToHost(this._stylesSet, hostNode, styleNodes);
    this._hostNodes.set(hostNode, styleNodes);
  }

  removeHost(hostNode: Node): void {
    const styleNodes = this._hostNodes.get(hostNode);
    if (styleNodes) {
      styleNodes.forEach(removeStyle);
    }
    this._hostNodes.delete(hostNode);
  }

  override onStylesAdded(additions: Set<string>): void {
    this._hostNodes.forEach((styleNodes, hostNode) => {
      this._addStylesToHost(additions, hostNode, styleNodes);
    });
  }

  ngOnDestroy(): void {
    this._hostNodes.forEach(styleNodes => styleNodes.forEach(removeStyle));
  }
}
```

Ok, let's start with the constructor. There is only one object injected.
It's the `document` global object.
Why it's typed as any...?
Probably there is a reason, but I don't know it, and I don't want to create a new myth about it 

In `DomSharedStylesHost` class methods `_addStylesToHost` and `onStylesAdded` are used
to add new `<style>` element with the styles contend when it's necessary.

But when it's necessary...? Great question! 

Angular has to add style when `addStyles` method has been called
(look back to the constructor of the `EmulatedEncapsulationDomRenderer2`).
This method is implemented in the `SharedStylesHost` base class.

Check out the code bellow.

```typescript
@Injectable()
export class SharedStylesHost {
  /** @internal */
  protected _stylesSet = new Set<string>();

  addStyles(styles: string[]): void {
    const additions = new Set<string>();
    styles.forEach(style => {
      if (!this._stylesSet.has(style)) {
        this._stylesSet.add(style);
        additions.add(style);
      }
    });
    this.onStylesAdded(additions);
  }

  onStylesAdded(additions: Set<string>): void {}

  getAllStyles(): string[] {
    return Array.from(this._stylesSet);
  }
}
```

From the implementation of the method whe know few things:
- There is a global set object that keeps all the styles added to the application.
- Styles are always filtered using this set, so **Angular will never add the same style twice.**
- At the end of the method `onStylesAdded` is called, and it will call one of the "browser" or "server" implementations.

Unfortunately, you cannot use SharedStylesHost by yourself to add styles "manually".
This class is not a part of Angular public API, and it's not exported. 

At the end, the code is very simple, and you can implement it in your project.
Why...? A good use case it when you want to load the styles dynamically with the code and webpack dynamic imports. 

## Performance

I described all the encapsulation styles provided by the Angular framework.
After reading tons of the code, it's time to compare the performance of each method.

As always, I create a repository with stupid simple application that can be used for testing purposes.
You can find it here [github.com/galczo5/experiment-angular-encapsulation](https://github.com/galczo5/experiment-angular-encapsulation).

{{< rawhtml >}}
<div style="height: 400px; width: 1300px; border: 1px solid lightgray;">
    <iframe style="border: 0; width: 100%; height: 100%;" src="https://galczo5.github.io/experiment-angular-encapsulation/"></iframe>
</div>
{{< /rawhtml >}}

### The test

To understand the test, it's necessary to understand the components.
Below I pasted the code of the `AppComponent` and its template.

To test it well, I decided to run it in the loop with 100 iterations;
each iteration is rendering 10 000 components.
It looks like it's enough to get satisfying results.

```typescript
import {ChangeDetectionStrategy, ChangeDetectorRef, Component} from '@angular/core';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AppComponent {

  type: 'clear' | 'emulated' | 'shadow' | 'none' = 'clear';

  array = new Array(10000).fill(0);

  constructor(
    private readonly changeDetectorRef: ChangeDetectorRef
  ) {
  }

  render(type: 'clear' | 'emulated' | 'shadow' | 'none'): void {
    let testTime = 0;
    for (let i = 0; i < 100; i++) {
      const start = performance.now();
      this.type = type;
      this.changeDetectorRef.detectChanges();
      const end = performance.now();
      testTime += (end - start);

      this.type = 'clear';
      this.changeDetectorRef.detectChanges();
    }

    console.log('TOTAL', testTime);
    console.log('AVG', testTime / 100);
  }

}
```

And of course, the template:

```html
<h1>Styles encapsulation</h1>

<div>
  <h2>Emulated</h2>
  <app-emulated></app-emulated>
</div>

<div>
  <h2>None</h2>
  <app-no-encapsulation></app-no-encapsulation>
</div>

<div>
  <h2>ShadowDom</h2>
  <app-shadow></app-shadow>
</div>

<div>
  <h2>Test - 100 * Create 10000 components</h2>
  <button (click)="render('clear')">Clear</button>
  <button (click)="render('emulated')">Emulated</button>
  <button (click)="render('none')">None</button>
  <button (click)="render('shadow')">ShadowDom</button>
</div>

<div *ngIf="type === 'emulated'">
  <app-emulated *ngFor="let x of array"></app-emulated>
</div>

<div *ngIf="type === 'shadow'">
  <app-shadow *ngFor="let x of array"></app-shadow>
</div>

<div *ngIf="type === 'none'">
  <app-no-encapsulation *ngFor="let x of array"></app-no-encapsulation>
</div>
```

### My results 

If you don't want to do the tests on your computer, here are my results. 

| Encapsulation type | Total | Avg iteration time |
|--------------------| --- |--------------------|
| None               | 12496ms | 124,9 ms           |
| Emulated           | 13027ms | 130,2ms            |
| ShadowDom          | 21352ms | 213,5ms            |

There is almost no difference between the `Emulated` and disabled encapsulation.
The difference between `ShadowDom` and the rest of the encapsulation types is huge.

## Conclusion

It scares me a little that I wrote an article about encapsulation
and its performance where the test is a very small part of the whole text.
I just wanted to describe differences well :D 

To sum up, the default `Emulated` encapsulation is as fast as no encapsulation at all.
It looks that this can be explained by the implementation.
At the end, it's only a smart way of using native css attribute selectors.
The `ShadowDom` way is significantly slower,
maybe it's the reason why it's not very popular in my environment.
