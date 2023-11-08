---
title: "You all need to know about this! You can massively reduce Angular build time, and it's easy!"
date: 2023-11-08T15:07:20+01:00
draft: false
---

I've been recently on NgPoland 2023 in Warsaw.
It's the biggest Angular conference in Poland, and if you have a chance and love Angular, it's worth considering visiting Warsaw.

As you may expect, the biggest thing presented during talks was signals.
Really, it looks like that community is crazy about them.
Below, I'm posting the list of all talks with bolded sessions that at least touched the signals.

- **Alex Rickabaugh / Pawel Kozlowski - Keynote Session - Exploring Control Flow and Reactivity**
- **Manfred Steyer - Angular-Architectures with Signals**
- **Michael Hladky - 3 Reactive Primitives – Angular Architecture With RxJS and Signals**
- Rainer Hahnekamp - There's New Sheriff in Town
- Younes Jaaidi - Fake it till you Mock it
- **Alex Okrushko - Angular Signals are here. Is NgRx now obsolete?**
- **Tomasz Ducin - Thinking in Signals**
- Matt Lewis - Making Development Times Fast With Esbuild
- **Miško Hevery - Comparative Study of Reactivity Across Frameworks and Implications for Resumability**
- Brygida Fiejdasz / Daniel Glejzner / Rafał Brzoska & Jarred Utt / Vojtech Mašek - Lightning Talks
- Julian Jandl - Virtual Scrolling Unveiled: High Performance Lists in Angular
- Mateusz Łędzewicz - To Module or Not to Module, That Is the Question!
- Alisa Duncan - Introducing the Identity Guardians
- Tobiasz Ciesielski - Angular Best Practices: Reactive Data Flow with RxJS and Bulletproof Component Design
- **Michael Egger-Zikes - The Internal Magic of Angular Signals / Hosting the Q&A Session with the Angular Core Team**
- Christopher Holder - Managing Design Systems With Storybook and Angular
- Minko Gechev / Mark Thompson - Keynote Session

Unfortunately, signals aren't working without NgZone, so I will not focus on them today.
If you want, there are a lot of incredible articles about them.

In this article, I want to show you what I learned from the "Making Development Times Fast With Esbuild" presentation by Matt Lewis.

## Faster builds with `@clickup/ngx-esbuild`

First, as you may remember from my previous posts, `@angular-devkit/build-angular:browser-esbuild` has some problems.
It's slower than we imagined because of a time-consuming project analysis step.
It's faster, of course, but slower than you may expect from esbuild.

![esbuild-compilation-problems](/images/esbuild-compilation-problems.jpg)

Since Angular 15, I've seen attempts to solve the problem from the different GitHub repositories,
blog posts, etc. I even tried to solve it by myself, and I failed.
A very long Angular build time still needs to be solved.

**I'm happy to report that there's probably the first working solution
that can build an Angular application using esbuild in a short time!**
Moreover, it's not a concept because ClickUp, the team behind the solution code, is using it.
They have a huge and mature application, so I wouldn't consider this as a PoC.

**The code and installation instructions can be found here: [github.com/clickup/ngx-esbuild](https://github.com/clickup/ngx-esbuild).**

Please remember that it's still fresh and new, and there might be some issues. I personally found two of them:

- You need to install `@babel/plugin-syntax-typescript` and `@babel/plugin-syntax-decorators` to make it work
- [`@Attribute(attrName)`](https://angular.io/api/core/Attribute) is not supported at the time when I'm writing this post

If you find something else, remember to open an issue.

**PS. It works only with Nx monorepositories. You need to wait for Angular CLI support.**

## How fast it is

I tested it on an empty repository, and it was faster than a standard `ng serve` command.
But we aren't working with empty repositories, right?

Especially for this post, I took a photo of the results presented during the talk. Improvement is massive!

![clickup-stats](/images/clickup-stats.jpg)

![clickup-esbuild-results](/images/clickup-esbuild-results.jpg)

## Summary

Thanks to NgPoland, the talk from Matt Lewis, and ClickUp for making it open source, we all can stop suffering from slow build processes.

Attending programming conferences is always worth the hassle; you can learn a lot in just a day.
What's more important, you can meet interesting people and learn from them.

In the end, I have to ask you a favor.
Find a moment to test [ngx-esbuild](https://github.com/clickup/ngx-esbuild) and spread the information about it.
The more we help with testing, the sooner [ngx-esbuild](https://github.com/clickup/ngx-esbuild) will be able
to cover most of the Angular community needs.  
