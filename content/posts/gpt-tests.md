---
title: "Chat GPT can write tests without TestBed, you should too"
date: 2023-03-18
draft: false
---

![gpt](/images/gpt.png)

**Me:**
Hi GPT! Are you ready for some frontend dev coding?

**Chat GPT:**
Hello! While I am not capable of writing code directly, I can certainly provide guidance and suggestions on frontend development. What do you need help with?

---

**Me:**
Do you know the Angular framework?

**Chat GPT:**
Yes, I am familiar with the Angular framework. Angular is a popular JavaScript framework for building web applications, and it is often used for building complex single-page applications. It was developed by Google and is known for its robustness, modularity, and scalability. What do you need help with in Angular?

---

**Me:**
Can you generate form an empty component an call it ExampleComponent. The template should contain header with text "I was generated by GPT"

**Chat GPT:**
Sure, here is an example of how you can generate a new component named ExampleComponent with a header containing the text "I was generated by GPT":

1. Open up a terminal and navigate to your Angular project's root directory.
2. Run the following command to generate a new component:  
`ng generate component Example`.\
This will create a new folder named example under the src/app folder, and add the necessary files for the new component.
3. Open the newly created example.component.html file and replace its contents with the following code: \
`<h1>I was generated by GPT</h1>` \
This will add a header with the desired text to the component's template.
4. Save the file, and you're done!
   You can now use the ExampleComponent in your Angular application by adding its selector
   (app-example) to any other component's template.
   \
For example, in the app.component.html file, you can add the following line to display the ExampleComponent:\
`<app-example></app-example>`\
When you run your Angular application, you should see the header "I was generated by GPT" displayed on the page.

---

**Me:**
Can you put the template in the component typescript file and print that code for me?

**Chat GPT:**
Sure, here is an example of how to define the component's template directly in the TypeScript file:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1>I was generated by GPT</h1>
  `
})
export class ExampleComponent {}
```

This code defines a new component named ExampleComponent and sets its template to the HTML code contained within the backticks (`) in the template property.

To display this component in another component's template, you can use its selector (app-example) as follows:

```html
<app-example></app-example>
```

When the parent component is rendered, it will include the ExampleComponent with the "I was generated by GPT" header.

---

**Me:**
Can you add one text input to the component and call it backgroudColor?

**Chat GPT:**
Sure, here's an example of how to add a text input to the ExampleComponent and give it the name backgroundColor:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1 [style.background-color]="backgroundColor">I was generated by GPT</h1>
    <input [(ngModel)]="backgroundColor" placeholder="Enter a background color">
  `
})
export class ExampleComponent {
  backgroundColor: string = '';
}
```
This code adds an input element to the component's template, which is bound to the backgroundColor property of the component using [(ngModel)]. The placeholder attribute sets the default text for the input.

The backgroundColor property is initialized to an empty string in the component's class, and is used to set the background color of the h1 element using the [style.background-color] directive.

With this code, the user can enter a background color value into the text input, and the header's background color will change accordingly.

---

**Me:**
I mean, I wanted an @Input property, not an input html element. Can you fix it?

**Chat GPT:**
My apologies for the confusion. Here's an example of how to add an @Input property named backgroundColor to the ExampleComponent:

```typescript
import { Component, Input } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1 [style.background-color]="backgroundColor">I was generated by GPT</h1>
  `
})
export class ExampleComponent {
  @Input() backgroundColor: string = '';
}
```

This code adds an @Input property named backgroundColor to the ExampleComponent class, which can be set from the parent component using property binding.

In the template of the ExampleComponent, the backgroundColor property is used to set the background color of the h1 element using the [style.background-color] directive.

With this code, you can use the ExampleComponent in another component's template like this:

```html
<app-example [backgroundColor]="'red'"></app-example>
```

This will set the background color of the ExampleComponent's header to red. The backgroundColor value can be dynamically set by binding to a property on the parent component.

---

**Me:**
Thx! Now, every time that backgroundColor is changing I want to add css class to the h1 element using Renderer2 service


**Chat GPT:**
Sure! Here's an example of how to use the Renderer2 service to add a CSS class to the h1 element in the ExampleComponent whenever the backgroundColor input property changes:

```typescript
import { Component, Input, Renderer2, ElementRef } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1 [style.background-color]="backgroundColor">I was generated by GPT</h1>
  `
})
export class ExampleComponent {
  @Input() backgroundColor: string = '';

  constructor(private renderer: Renderer2, private el: ElementRef) {}

  ngOnChanges() {
    const h1Element = this.el.nativeElement.querySelector('h1');
    if (h1Element) {
      this.renderer.addClass(h1Element, 'my-class');
    }
  }
}
```

In this code, the ngOnChanges() method is called whenever the value of the backgroundColor input property changes.

Inside ngOnChanges(), we use the ElementRef service to get a reference to the h1 element, and then use the Renderer2 service to add a CSS class named my-class to the element using the addClass() method.

With this code, whenever the backgroundColor input property changes, the my-class CSS class will be added to the h1 element, allowing you to apply custom styles to the element using CSS.

---

**Me:**
Nice! We don't need that background-color in the template

**Chat GPT:**
You're right, we can remove the [style.background-color] directive from the template, since we're now setting the background color using the Renderer2 service. Here's the updated ExampleComponent code:

```typescript
import { Component, Input, Renderer2, ElementRef } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1>I was generated by GPT</h1>
  `
})
export class ExampleComponent {
  @Input() backgroundColor: string = '';

  constructor(private renderer: Renderer2, private el: ElementRef) {}

  ngOnChanges() {
    const h1Element = this.el.nativeElement.querySelector('h1');
    if (h1Element) {
      this.renderer.addClass(h1Element, 'my-class');
      this.renderer.setStyle(h1Element, 'background-color', this.backgroundColor);
    }
  }
}
```

In this code, we've removed the [style.background-color] directive from the template.

Inside ngOnChanges(), we use the Renderer2 service to add the my-class CSS class to the h1 element, and also set the background color of the element using the setStyle() method.

With this code, the ExampleComponent will no longer have the [style.background-color] directive in its template, and the background color will be set dynamically using the Renderer2 service.

---

**Me:**
Wow! I need to remove that line with my-class. Can you do it for me?

**Chat GPT:**
Sure thing! Here's the updated ExampleComponent code without the my-class CSS class:

```typescript
import { Component, Input, Renderer2, ElementRef } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1>I was generated by GPT</h1>
  `
})
export class ExampleComponent {
  @Input() backgroundColor: string = '';

  constructor(private renderer: Renderer2, private el: ElementRef) {}

  ngOnChanges() {
    const h1Element = this.el.nativeElement.querySelector('h1');
    if (h1Element) {
      this.renderer.setStyle(h1Element, 'background-color', this.backgroundColor);
    }
  }
}
```

In this code, we've removed the my-class CSS class from the ngOnChanges() method. Now, whenever the backgroundColor input property changes, the ngOnChanges() method will use the Renderer2 service to set the background color of the h1 element directly using the setStyle() method.

With this code, the ExampleComponent will no longer have the my-class CSS class, and the background color will be set dynamically using the Renderer2 service.

---

**Me:**
Is it a good idea to make Renderer2 and ElementRef readonly?

**Chat GPT:**
Yes, it's generally a good idea to make the Renderer2 and ElementRef properties readonly in an Angular component. By making these properties readonly, we ensure that they can only be set once, during the component's initialization. This helps to prevent unintended changes to these properties later on in the component's lifecycle.

Here's an example of how we can make the Renderer2 and ElementRef properties readonly in the ExampleComponent:

```typescript
import { Component, Input, Renderer2, ElementRef } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1>I was generated by GPT</h1>
  `
})
export class ExampleComponent {
  @Input() backgroundColor: string = '';

  constructor(private readonly renderer: Renderer2, private readonly el: ElementRef) {}

  ngOnChanges() {
    const h1Element = this.el.nativeElement.querySelector('h1');
    if (h1Element) {
      this.renderer.setStyle(h1Element, 'background-color', this.backgroundColor);
    }
  }
}
```

In this code, we've added the readonly keyword to the Renderer2 and ElementRef properties in the ExampleComponent constructor. This makes these properties readonly, preventing them from being changed after the component is initialized.

Making the Renderer2 and ElementRef properties readonly is a good practice in Angular, as it helps to prevent unintended changes to these properties later on in the component's lifecycle.

---

**Me:**
Ok. I heard that we can replace querySelector with a @ViewChild, do you know how to do it?

**Chat GPT:**
Yes, you're right! We can use the @ViewChild decorator to get a reference to the h1 element instead of using querySelector in the ngOnChanges() method.

Here's how we can update the ExampleComponent to use @ViewChild:

```typescript
import { Component, Input, Renderer2, ElementRef, ViewChild, OnChanges } from '@angular/core';

@Component({
  selector: 'app-example',
  template: `
    <h1 #titleElement>I was generated by GPT</h1>
  `
})
export class ExampleComponent implements OnChanges {
  @Input() backgroundColor: string = '';
  @ViewChild('titleElement') titleElement!: ElementRef;

  constructor(private readonly renderer: Renderer2) {}

  ngOnChanges() {
    if (this.titleElement) {
      this.renderer.setStyle(this.titleElement.nativeElement, 'background-color', this.backgroundColor);
    }
  }
}
```

In this updated code, we've added a ViewChild decorator for the h1 element with the template reference variable #titleElement. We've also removed the el property from the constructor since we no longer need it.

Inside ngOnChanges(), we check if this.titleElement is defined (since it will only be defined after the view has been initialized), and if it is, we use the Renderer2 service to set the background color of the h1 element using this.titleElement.nativeElement instead of querySelector.

Using @ViewChild is a more efficient way to get a reference to an element in the component's template, since it doesn't require searching the entire DOM tree for the element like querySelector does.

---

**Me:**
Looks good! We will need a unit test for that component.

**Chat GPT:**
Great! Here's an example of a unit test for the ExampleComponent:

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ExampleComponent } from './example.component';

describe('ExampleComponent', () => {
  let component: ExampleComponent;
  let fixture: ComponentFixture<ExampleComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [ ExampleComponent ]
    })
    .compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(ExampleComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create the component', () => {
    expect(component).toBeTruthy();
  });

  it('should set the background color of the h1 element when the input changes', () => {
    const h1Element = fixture.nativeElement.querySelector('h1');
    component.backgroundColor = 'red';
    fixture.detectChanges();
    expect(h1Element.style.backgroundColor).toBe('red');
  });
});
```

This test suite uses the TestBed framework to create and configure the ExampleComponent for testing. The beforeEach function creates a ComponentFixture instance for the component, and the it blocks contain individual test cases.

In the first test case, we use expect(component).toBeTruthy() to check that the component was created successfully.

In the second test case, we set the backgroundColor input property to 'red', trigger change detection with fixture.detectChanges(), and then use expect(h1Element.style.backgroundColor).toBe('red') to check that the background color of the h1 element was set correctly.

This test suite provides a basic set of tests for the ExampleComponent, covering the creation of the component and the correct behavior of the ngOnChanges() method. You can add additional test cases as needed to further test the component's functionality.

---

**Me:**
The next one will be tricky. Are you ready?

**Chat GPT:**
Sure! I'm always ready to help. What's the challenge?

---

**Me:**
I need the same tests but because I plan to use esbuild, we cannot use TestBed util. So it must be working and testing the same, but you cannot use the TestBed.

**Chat GPT:**
Ah, I see. In that case, we can create an instance of ExampleComponent directly in our test file and manually set its properties and call its methods.

Here's an example of how to write the same tests without using TestBed:

```typescript
import { ExampleComponent } from './example.component';

describe('ExampleComponent', () => {
  let component: ExampleComponent;

  beforeEach(() => {
    component = new ExampleComponent();
  });

  it('should create the component', () => {
    expect(component).toBeTruthy();
  });

  it('should set the background color of the h1 element when the input changes', () => {
    const mockElementRef = {
      nativeElement: {
        style: {
          backgroundColor: ''
        }
      }
    };
    const mockRenderer2 = {
      setStyle: jest.fn()
    };

    component.titleElement = mockElementRef as any;
    component.renderer = mockRenderer2 as any;
    component.backgroundColor = 'red';
    component.ngOnChanges();

    expect(mockRenderer2.setStyle).toHaveBeenCalledWith(mockElementRef.nativeElement, 'background-color', 'red');
  });
});
```

In this updated code, we create a new instance of ExampleComponent in the beforeEach function. We then create a mock ElementRef object and a mock Renderer2 object that we can use to test the component.

In the second test case, we set the titleElement, renderer, and backgroundColor properties of the component to our mock objects. We then call ngOnChanges() to trigger the method that sets the background color of the h1 element. Finally, we use jest.fn() to create a spy on the setStyle method of the mock Renderer2 object, and we use expect(mockRenderer2.setStyle).toHaveBeenCalledWith(mockElementRef.nativeElement, 'background-color', 'red') to check that the background color of the h1 element was set correctly.

This test suite provides the same set of tests as the previous example, but without using TestBed. It uses manual instantiation of the component and mocks of the ElementRef and Renderer2 objects to simulate the behavior of the component in a test environment.

---

**Me:**
Good job!

**Chat GPT:**
Thank you! Let me know if you have any other questions or if there's anything else I can help you with.

---

**Me:**
Is there any advantage of the tests without TestBed? Or should I stay with the standard method?

**Chat GPT:**
There are pros and cons to both approaches, and ultimately the choice between using TestBed and manually instantiating components for testing will depend on the specific needs and constraints of your project.

Here are some factors to consider:

TestBed:

Pros:

Provides a higher-level API for testing components that more closely resembles the way components are used in a real Angular application.
Automates many common setup tasks, such as creating a test module, compiling components, and injecting dependencies.
Provides a more comprehensive testing environment that includes support for things like service mocking and asynchronous operations.
Cons:

Can be slower and more resource-intensive, especially for large or complex components.
Can be more difficult to customize the test environment or inject complex dependencies.
Requires more knowledge of Angular's testing API and conventions to use effectively.
Manual Instantiation:

Pros:

Can be faster and more lightweight than using TestBed.
Provides greater control over the test environment and the ability to inject any dependencies needed for testing.
Can be simpler to set up for simpler components.
Cons:

Can be more verbose and time-consuming to write the test setup code manually.
May require more knowledge of the underlying framework and its behavior to use effectively.
May not provide the same level of testing coverage or be able to test certain aspects of component behavior that depend on the Angular framework.
In general, if you're working on a large or complex Angular application with many interdependent components, using TestBed may be a more practical choice due to its higher-level API and built-in support for common testing tasks. However, for smaller or simpler components or for certain kinds of testing scenarios, manually instantiating components may be more efficient and flexible.

Ultimately, the best approach will depend on the specific needs and constraints of your project, and may require some experimentation and iteration to find the most effective testing strategy.

---

**Me:**
I'm wondering, do you know how the TestBed affects the run time of the tests?


**Chat GPT:**
Yes, using TestBed can have an impact on the runtime of your tests. Because TestBed creates a full Angular testing environment, it can be more resource-intensive than manually instantiating components for testing.

Here are some factors that can affect the runtime of tests using TestBed:

Test complexity: Tests that require more setup or involve more complex components or dependencies will generally take longer to run.
Module size: If your application has a large number of modules, or if the modules themselves are large, it can take longer for TestBed to create a test module.
Compilation time: When using TestBed, the Angular compiler must compile your components before they can be tested. This can add to the overall runtime of your tests.
Test configuration: If your test environment includes additional configuration steps, such as setting up mocks or configuring services, these steps can also add to the overall runtime.
In general, it's a good idea to keep your tests as lightweight as possible to reduce their impact on overall test runtime. This may involve things like breaking up large tests into smaller, more focused tests, minimizing setup and teardown steps, and using techniques like mocking and dependency injection to simplify test setup. Additionally, you can use tools like Jasmine's describe and it functions to organize your tests and make it easier to identify and diagnose slow tests.

---

**Me:**
Can you recommend me any article about esbuild configuration for Angular unit testing in Jest?


**Chat GPT:**
Yes, there are several articles available that discuss setting up esbuild with Jest for Angular unit testing. Here are a few resources you might find helpful:

1. [Angular unit tests with Jest and esbuild](https://dev.to/camptocamp/angular-unit-tests-with-jest-and-esbuild-3mg0) - This article provides a step-by-step guide to setting up esbuild with Jest for Angular unit testing, and includes sample code and configuration files.
2. [Angular unit testing with Jest and esbuild](https://medium.com/angular-in-depth/angular-unit-testing-with-jest-and-esbuild-7bce0d83ca1e) - This article provides an overview of using Jest and esbuild for Angular unit testing, including some tips for optimizing test performance.
3. [Faster Angular testing with Jest and esbuild](https://jesperhoy.com/faster-angular-testing-with-jest-and-esbuild/) - This article provides a detailed walkthrough of setting up Jest and esbuild for Angular unit testing, and includes sample code and configuration files.

Keep in mind that the specifics of esbuild configuration may vary depending on your specific project setup and testing requirements, so you may need to adjust the examples provided in these articles to fit your needs.

---

**Me:**
Thx!

**Chat GPT:**
You're welcome! Let me know if you have any other questions or if there's anything else I can help you with.

---

**Me:**
Can you export me the content of this chat as a markdown file?

**Chat GPT:**
Sure, here is the content of our chat as a Markdown file...

---

Soooo...
As you see, Chat GPT is able to create unit tests without TestBed.
It's not like it's going to replace us, but It may help you a lot.

If you're afraid that AI will replace you in the short time (because a lot of journalists say that),
I have great news for you! It's not going to happen :D  

How can I know...? I asked.

![gpt](/images/programmers.png)
