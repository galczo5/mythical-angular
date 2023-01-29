---
title: "What you can do with host directives"
date: 2023-01-28
draft: false
---

Host directives were a highly requested feature for a very long time.
I think that this feature is great.
First of all, I don't have to create unnecessary div or other elements to add a directive to my components.
I don't have to create additional inputs, and outputs and pass them to the template.

The next idea about host directives,
which is quite intuitive and obvious to me,
is that finally we can encapsulate some behavior or logic in directives, and compose our components with it easily.
Using this method allows us to drastically increase the re-usability of our code.

Today I want to show you the idea of how you can use host directives to improve your unit testing approach.
This method is not only healthy for TTD, but it helps a lot in the matter of the separation of businesses and rendering logic.

## Code

First of all, let's start with the code.
Working app with simple unit tests you can find in my repo:
[github.com/galczo5/experiment-host-directives](https://github.com/galczo5/experiment-host-directives). Today there is no need
to deploy it because I'm focusing on the code, developer experience, and a little on architecture.

The GitHub repo contains a simple app that has three main components:
- User list - presents a list of users, each row can be selected
- User details - presents detailed information about the selected user
- Filter input - used to filter users on the list

![host-directives-app](/images/host-directives-app.png)

It's extremely simple,
and I bet that almost every front-end developer had to create a feature like this at some point.

## Idea

The main idea is to move all non-UI logic to the directive.
The directive has to be a host directive and should expose inputs and outputs to be visible as we do it for components.

At this stage, we have UI logic in the component and business logic in the directive.
This solution has a few advantages, for example, it's a lot easier to replace/rewrite the UI layer of your app.
The second advantage is that you can create multiple components for the same logic.
You can have different ways to select users but the logic for that is the same.
What's more, with this approach it's quite easy to compose your components with directives,
which I mentioned in the intro.

All of this is super cool, but there is one advantage that convinces me completely.
If you split your code as I described, it should be easy to create unit tests.
From my previous articles, you might know, that I prefer my tests to be extremely fast.

To achieve this, I cannot use `TestBed` and I would use esbuild as a transformer in jest.
When the most important code is in a directive, there is no dilemma.
There is no UI so I don't need to use `TestBed`!

In this approach, some may say architecture, when I discussed it with other devs I found one more problem to solve.
We need to create a good way of communication between our directive and a component.

## Communication and example

Let's go back to our example.
The user list has to fetch data from some kind of API.
The call should contain information about the filter applied in the filter input.

I created a directive that gets filter value on input and gets filtered user in the `ngOnChanges` hook.

```typescript
import {Directive, Input, OnChanges, OnDestroy, Self, SimpleChanges} from '@angular/core';
import {UserListService} from "./user-list.service";
import {Subject, takeUntil} from "rxjs";
import {UsersService} from "./users.service";

@Directive({
  selector: '[appUserList]',
  standalone: true,
  providers: [
    UserListService
  ]
})
export class UserListDirective implements OnChanges, OnDestroy {

  @Input()
  filterValue = '';

  private readonly onDestroy$ = new Subject<void>();

  constructor(
    private readonly usersService: UsersService,
    @Self() private readonly userListService: UserListService
  ) {
  }

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['filterValue']) {
      this.getFilteredUsers();
    }
  }

  ngOnDestroy(): void {
    this.onDestroy$.next(void 0);
    this.onDestroy$.complete();
  }

  private getFilteredUsers(): void {
    this.usersService.getFilteredUsers(this.filterValue)
      .pipe(
        takeUntil(this.onDestroy$)
      )
      .subscribe(users => this.userListService.setUsers(users));
  }
}
```

When the data is fetched I pass it through the `UserListService`,
which is provided with `@Self()` and is declared in the directive using `providers` field.

```typescript
import {Injectable} from '@angular/core';
import {Observable, ReplaySubject, Subject} from "rxjs";

@Injectable()
export class UserListService {

  private readonly users$ = new ReplaySubject<Array<string>>(1);

  setUsers(users: Array<string>): void {
    this.users$.next(users);
  }

  getUsers(): Observable<Array<string>> {
    return this.users$.asObservable();
  }

}
```

Now my component can consume this service and treat it as a contract or API.
If I would like to create another component that acts like the previous one,
simply, just add a host directive and react on changes from `UserListService`.

```typescript
import {Component, EventEmitter, Output, Self} from '@angular/core';
import {CommonModule} from '@angular/common';
import {UserListService} from "./user-list.service";
import {ListComponent} from "../list/list.component";
import {PushModule} from "@rx-angular/template/push";
import {UserListDirective} from "./user-list.directive";

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [CommonModule, ListComponent, PushModule],
  template: `
    <app-list [items]="userListService.getUsers() | push"
              (itemSelected)="userSelected.emit($event)"></app-list>
  `,
  hostDirectives: [
    {
      directive: UserListDirective,
      inputs: ['filterValue']
    }
  ]
})
export class UserListComponent {

  @Output()
  userSelected = new EventEmitter<string>();

  constructor(
    @Self() readonly userListService: UserListService
  ) {
  }

}
```

In this case, unit tests for `UserListService` would be straightforward.
This service will not contain any complicated logic,
because its only responsibility is to be a proxy between UI and business logic.

There is no need to create unit tests for `UserListComponent` if `ListComponent` is tested.
is easier to test, because it doesn't contain any UI logic.

```typescript
import {UserListDirective} from "./user-list.directive";
import {UsersService} from "./users.service";
import {Observable, of, take} from "rxjs";
import {FakeHttpService} from "../fake-http.service";
import {UserListService} from "./user-list.service";

class MockUserService extends UsersService {
  constructor() {
    super(null as unknown as FakeHttpService);
  }

  override getFilteredUsers(filter: string): Observable<Array<string>> {
    return of(
      ['user1', 'user2', 'user3'].filter(x => x.includes(filter))
    );
  }
}

describe('user-list-directive', function () {

  it('should pass filtered users on filter change', function (done) {
    const userListService = new UserListService();
    const userListDirective = new UserListDirective(
      new MockUserService(),
      userListService
    );

    userListService.getUsers()
      .pipe(take(1))
      .subscribe(users => {
        expect(users).toEqual(['user1']);
        userListDirective.ngOnDestroy();
        done();
      });

    userListDirective.filterValue = '1';
    userListDirective.ngOnChanges({
      filterValue: {
        currentValue: '1',
        previousValue: undefined,
        firstChange: true,
        isFirstChange: () => true
      }
    });

  });

});
```

## Conclusion
The introduction of the host directives feature might change the way we use Angular.
It's not only a simple helper that will help us to reduce one `div` in our code.

It encourages us to change our approach to managing our code.
It amazes me how I can change my code and boosts the re-usability of my code (if well designed).

## Bonus: Alternative scenarios and ideas that I considered

I think that I should publish not only the final product of my thoughts but also the "alternative"
paths that I considered.
This way, you'll know more about the whole process and the pluses and minuses of other approaches.

### Why host directive and not a service? The service does not contain a template and it was available to use a long time ago.

So, that's right.
But services cannot contain inputs and outputs, so they are not exposing API that can be used in templates.
You can still achieve a similar result
but your component will contain unnecessary logic only to pass data to the service.
So, it's not only responsible for UI, but it's a proxy for business logic too.

### Why do I prefer to have an additional service as a proxy instead of injecting an instance of the directive directly?

This one is tricky.
First of all, if you inject the instance, you have all access to everything that is inside.
Your directive will not deal only with the complex logic but with being an API for a component also.

The second reason is that it's easier to mock service in unit tests.
In most cases, this proxy service will contain a very simple logic, and probably will be stateless.
In my case, there is no need to mock it, because I can use `setUsers()` method. 
