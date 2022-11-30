---
title: "What exactly esbuild is doing in Angular...?"
date: 2022-11-23
draft: false
---

Angular 15 was recently released.
From [blog post](https://blog.angular.io/angular-v15-is-now-available-df7be7f2f4c8) we know that the Angular team is working
on bringing faster builds to the Angular ecosystem.

```text
In v14 we announced the experimental support for esbuild 
in ng build to enable faster build times and simplify our pipeline.

In v15 we now have experimental Sass, SVG template, file replacement, 
and ng build --watch support!
```

[Source](https://blog.angular.io/angular-v15-is-now-available-df7be7f2f4c8)

It looks like my dream is coming true.
For a very long time, I'm waiting for that revolution.
Isn't it going to be great to have building processes as fast I have in React or Vue?!

![esbuild](/images/esbuild.png)


You know my articles.
We need to test it, and we'll try to investigate what exactly esbuild is doing in the Angular 15 building processes.

## App generator

To test build time is good to have a huge app.
I don't have a huge enough app on my GitHub profile.
For a moment, let's assume that I have that app.
It's not the best testing subject.
Yes, it's a real life example and so on, but I cannot scale it.
It's hard to find out how the time of the compilation will change when your number of files are constant.

Today I will use a simple generator, that is able to generate as many components as I want.
A very simple components,
but to simulate a scale, I just need to generate a few times more components, as usually companies have in their apps.

```javascript
const {readFileSync, writeFileSync} = require('fs');

const sample = readFileSync('./src/app/generated/sample.component.ts').toString();

const iterations = 5_000;
let template = '';
let imports = '';
let declarations = '';

for (let i = 0; i < iterations; i++) {
    writeFileSync(`./src/app/generated/generated-${i}.component.ts`, sample.replaceAll('${1}', i));
    template += `    <sp-generated-${i}></sp-generated-${i}>` + '\n';
    imports += `import {GeneratedComponent${i}} from "./generated/generated-${i}.component";` + '\n';
    declarations += `    GeneratedComponent${i},` + '\n';
}

const appComponent = readFileSync('./src/app/app.component.ts').toString();
writeFileSync(
    './src/app/app.component.ts',
    appComponent.replaceAll('<!-- REPLACE ME -->', template)
);

const appModule = readFileSync('./src/app/app.module.ts').toString()
writeFileSync(
    './src/app/app.module.ts',
    appModule
        .replaceAll('// REPLACE IMPORT', imports)
        .replaceAll('// REPLACE DECLARATION', declarations)
);
```

Generation is quite simple.
My code is reading file `./src/app/generated/sample.component.ts`, which is a template of component.
Then some code is replaced with the index of the component,
and after that I'm setting everything in `AppModule` and `AppComponent`.

If you want to check it our,
or just double my tests on your machine,
here is the [galczo5/experiment-angular-15-esbuild](https://github.com/galczo5/experiment-angular-15-esbuild) repo.

## Test results

| Number of components | @angular-devkit/build-angular:browser | @angular-devkit/build-angular:browser-esbuild |
|----------------------|---------------------------------------|-----------------------------------------------|
| 100 (cold)           | 11.63s                                | 5.63s                                         |
| 100                  | 4.94s                                 | 4.44s                                         |
| 100                  | 5.55s                                 | 5.11s                                         |
| -------------------- |                                       |                                               |
| 1_000 (cold)         | 16.34s                                | 8.25s                                         |
| 1_000                | 7.83s                                 | 7.38s                                         |
| 1_000                | 7.19s                                 | 7.23s                                         |
| -------------------- |                                       |                                               |
| 5_000 (cold)         | 60.60s                                | 33.62s                                        |
| 5_000                | 38.52s                                | 36.52s                                        |
| 5_000                | 43.84s                                | 34.19s                                        |
| -------------------- |                                       |                                               |
| 10_000 (cold)        | 187.01s                               | 125.51s                                       |
| 10_000               | 119.18s                               | 121.00s                                       |
| 10_000               | 127.30s                               | 118.76s                                       |
| -------------------- |                                       |                                               |
| 20_000 (cold)        | 575.86s                               | 467.61s                                       |

According to my experience with the esbuild, it's not a full potential of its ability to compile and bundle typescript.
Just look at the results that I had almost two years ago when I was preparing my presentation about it. 

![esbuild-react](/images/esbuild-react.png)

When I'm using esbuild as a bundler, I rather expect **10 to 50 times** faster builds,
not time comparable to the webpack build with a cache.
So, what esbuild is doing in an Angular building process? It's time to dig in and check it.

## esbuild in Angular sources

First of all,
I moved in Angular repository to tag for version [15.0.1](https://github.com/angular/angular/tree/15.0.1).
I decided to use simple text search to look for all occurrences of `esbuild` in repo. 

```text
> ~/D/angular grep -r 'esbuild' packages
packages/compiler-cli/esbuild.config.js:  // Note: `@bazel/esbuild` has a bug and does not pass-through the format from Starlark.
packages/compiler-cli/esbuild.config.js:    // Workaround for: https://github.com/evanw/esbuild/issues/946
packages/compiler-cli/BUILD.bazel:load("@npm//@bazel/esbuild:index.bzl", "esbuild", "esbuild_config")
packages/compiler-cli/BUILD.bazel:esbuild_config(
packages/compiler-cli/BUILD.bazel:    name = "esbuild_config",
packages/compiler-cli/BUILD.bazel:    config_file = "esbuild.config.js",
packages/compiler-cli/BUILD.bazel:esbuild(
packages/compiler-cli/BUILD.bazel:    config = ":esbuild_config",
packages/core/test/bundling/hello_world_i18n/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/todo_i18n/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/hello_world/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/standalone_bootstrap/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/animation_world/index.html:          the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/forms_reactive/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/todo/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/image-directive/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/cyclic_import/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/forms_template_driven/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/animations/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/core/test/bundling/router/index.html:        the application. It has also gone through full pipeline of esbuild, babel optimization,
packages/localize/tools/esbuild.config.js:  // Note: `@bazel/esbuild` has a bug and does not pass-through the format from Starlark.
packages/localize/tools/esbuild.config.js:    // Workaround for: https://github.com/evanw/esbuild/issues/946
packages/localize/tools/BUILD.bazel:load("@npm//@bazel/esbuild:index.bzl", "esbuild", "esbuild_config")
packages/localize/tools/BUILD.bazel:esbuild_config(
packages/localize/tools/BUILD.bazel:    name = "esbuild_config",
packages/localize/tools/BUILD.bazel:    config_file = "esbuild.config.js",
packages/localize/tools/BUILD.bazel:esbuild(
packages/localize/tools/BUILD.bazel:    config = ":esbuild_config",
packages/service-worker/cli/esbuild.config.js:  // Note: `@bazel/esbuild` has a bug and does not pass-through the format from Starlark.
packages/service-worker/cli/esbuild.config.js:    // Workaround for: https://github.com/evanw/esbuild/issues/946
packages/service-worker/cli/BUILD.bazel:load("@npm//@bazel/esbuild:index.bzl", "esbuild", "esbuild_config")
packages/service-worker/cli/BUILD.bazel:esbuild_config(
packages/service-worker/cli/BUILD.bazel:    name = "esbuild_config",
packages/service-worker/cli/BUILD.bazel:    config_file = "esbuild.config.js",
packages/service-worker/cli/BUILD.bazel:esbuild(
packages/service-worker/cli/BUILD.bazel:    config = ":esbuild_config",
packages/service-worker/worker/BUILD.bazel:load("@npm//@bazel/esbuild:index.bzl", "esbuild")
packages/service-worker/worker/BUILD.bazel:esbuild(
```

It's not the code we are looking for.
This code proves that Angular repo is using esbuild internally as a bundler (check out usages in `BUILD.bazel` files).
The code that we need is not in the main Angular repository.
You'll find it in [angular/angular-cli.git](https://github.com/angular/angular-cli.git).

```text
> ~/D/angular-cli main grep -r 'from \'esbuild\'' .
./packages/angular_devkit/build_angular/src/builders/browser-esbuild/sass-plugin.ts:import type { PartialMessage, Plugin, PluginBuild } from 'esbuild';
./packages/angular_devkit/build_angular/src/builders/browser-esbuild/compiler-plugin.ts:} from 'esbuild';
./packages/angular_devkit/build_angular/src/builders/browser-esbuild/css-resource-plugin.ts:import type { Plugin, PluginBuild } from 'esbuild';
./packages/angular_devkit/build_angular/src/builders/browser-esbuild/stylesheets.ts:import type { BuildOptions, OutputFile } from 'esbuild';
./packages/angular_devkit/build_angular/src/builders/browser-esbuild/index.ts:import type { BuildInvalidate, BuildOptions, OutputFile } from 'esbuild';
./packages/angular_devkit/build_angular/src/builders/browser-esbuild/esbuild.ts:} from 'esbuild';
./packages/angular_devkit/build_angular/src/webpack/plugins/esbuild-executor.ts:} from 'esbuild';
./packages/angular_devkit/build_angular/src/webpack/plugins/css-optimizer-plugin.ts:import type { Message, TransformResult } from 'esbuild';
./packages/angular_devkit/build_angular/src/webpack/plugins/javascript-optimizer-worker.ts:import type { TransformResult } from 'esbuild';
```

For sure, `esbuild` is used for css optimization and style related stuff. What about typescript? 

File `./packages/angular_devkit/build_angular/src/builders/browser-esbuild/esbuild.ts` looks promising,
I've started with it during my analysis.

```typescript
export async function bundle(
  workspaceRoot: string,
  optionsOrInvalidate: BuildOptions | BuildInvalidate,
): Promise<
  | (BuildResult & { outputFiles: OutputFile[]; initialFiles: FileInfo[] })
  | (BuildFailure & { outputFiles?: never })
> {
  let result;
  try {
    if (typeof optionsOrInvalidate === 'function') {
      result = (await optionsOrInvalidate()) as BuildResult & { outputFiles: OutputFile[] };
    } else {
      result = await build({
        ...optionsOrInvalidate,
        metafile: true,
        write: false,
      });
    }
  } catch (failure) {
    // Build failures will throw an exception which contains errors/warnings
    if (isEsBuildFailure(failure)) {
      return failure;
    } else {
      throw failure;
    }
  }

  const initialFiles: FileInfo[] = [];
  for (const outputFile of result.outputFiles) {
    // Entries in the metafile are relative to the `absWorkingDir` option which is set to the workspaceRoot
    const relativeFilePath = relative(workspaceRoot, outputFile.path);
    const entryPoint = result.metafile?.outputs[relativeFilePath]?.entryPoint;

    outputFile.path = relativeFilePath;

    if (entryPoint) {
      // An entryPoint value indicates an initial file
      initialFiles.push({
        file: outputFile.path,
        // The first part of the filename is the name of file (e.g., "polyfills" for "polyfills.7S5G3MDY.js")
        name: basename(outputFile.path).split('.')[0],
        extension: extname(outputFile.path),
      });
    }
  }

  return { ...result, initialFiles };
}
```

`build` function from `esbuild` is used in `bundle` method, in this case I would be good to create a simple repository, add `console.log` of options passed to the `build` and check what is configured there. 

```text
{
  absWorkingDir: '/Users/kamil/Dev/experiment-angular-15-esbuild',
  bundle: true,
  incremental: false,
  format: 'esm',
  entryPoints: {
    main: '/Users/kamil/Dev/experiment-angular-15-esbuild/src/main.ts',
    polyfills: 'zone.js'
  },
  entryNames: '[name].[hash]',
  assetNames: '[name].[hash]',
  target: [
    'chrome107',  'edge107',
    'edge106',    'firefox107',
    'firefox102', 'ios16.1',
    'ios16.0',    'ios15.6',
    'ios15.5',    'ios15.4',
    'ios15.2',    'ios15.0',
    'safari16.1', 'safari16.0',
    'safari15.6', 'safari15.5',
    'safari15.4', 'safari15.2',
    'safari15.1', 'safari15'
  ],
  supported: { 'async-await': false, 'object-rest-spread': false },
  mainFields: [ 'es2020', 'browser', 'module', 'main' ],
  conditions: [ 'es2020', 'es2015', 'module' ],
  resolveExtensions: [ '.ts', '.tsx', '.mjs', '.js' ],
  logLevel: 'silent',
  minify: true,
  pure: [ 'forwardRef' ],
  outdir: '/Users/kamil/Dev/experiment-angular-15-esbuild',
  sourcemap: false,
  splitting: true,
  tsconfig: '/Users/kamil/Dev/experiment-angular-15-esbuild/tsconfig.app.json',
  external: [],
  write: false,
  platform: 'browser',
  preserveSymlinks: undefined,
  plugins: [ { name: 'angular-compiler', setup: [AsyncFunction: setup] } ],
  define: { ngDevMode: 'false', ngJitMode: 'false' },
  metafile: true
}

{
  absWorkingDir: '/Users/kamil/Dev/experiment-angular-15-esbuild',
  bundle: true,
  entryNames: '[name].[hash]',
  assetNames: '[name].[hash]',
  logLevel: 'silent',
  minify: true,
  sourcemap: false,
  outdir: '/Users/kamil/Dev/experiment-angular-15-esbuild',
  write: false,
  platform: 'browser',
  target: [
    'chrome107',  'edge107',
    'edge106',    'firefox107',
    'firefox102', 'ios16.1',
    'ios16.0',    'ios15.6',
    'ios15.5',    'ios15.4',
    'ios15.2',    'ios15.0',
    'safari16.1', 'safari16.0',
    'safari15.6', 'safari15.5',
    'safari15.4', 'safari15.2',
    'safari15.1', 'safari15'
  ],
  preserveSymlinks: undefined,
  external: [],
  conditions: [ 'style', 'sass' ],
  mainFields: [ 'style', 'sass' ],
  plugins: [
    { name: 'angular-global-styles', setup: [Function: setup] },
    { name: 'angular-sass', setup: [Function: setup] },
    { name: 'angular-css-resource', setup: [Function: setup] }
  ],
  incremental: false,
  entryPoints: { styles: 'angular:styles/global;styles' },
  metafile: true
}
```

There is no magic, bot typescript, and styles are compiled with the `esbuild`.
Both of them are using plugins made by Angular team.
I have that strange feeling that, maybe these plugins create a bottleneck in the whole process.

It's easy to verify.
I changed the code in
`node_modules/@angular-devkit/build-angular/src/builders/browser-esbuild/esbuild.js` to remove all plugins.
I had
to pass through the styles
because entry point of value `{ styles: 'angular:styles/global;styles' }` is not possible
to process without these plugins.

Wow! The results are stunning!

| Number of components | @angular-devkit/build-angular:browser | @angular-devkit/build-angular:browser-esbuild | esbuild - no plugins |
|----------------------|---------------------------------------|-----------------------------------------------|----------------------|
| 100 (cold)           | 11.63s                                | 5.63s                                         | 1.32s                |
| 1_000 (cold)         | 16.34s                                | 8.25s                                         | 1.26s                |
| 5_000 (cold)         | 60.60s                                | 33.62s                                        | 1.47s                |
| 10_000 (cold)        | 187.01s                               | 125.51s                                       | 2.40s                |
| 20_000 (cold)        | 575.86s                               | 467.61s                                       | 1.31s                |

The bundle of course is not working,
because I removed a very important part of a compilation process,
but still, I would like to have that fast build processes.

Now we know that the problem is in the plugins.

## Summary

This post is quite long, so I decided to stop here.
We know that the problem is in esbuild plugins.

We have a few of them:
- angular-compiler
- angular-global-styles
- angular-sass
- angular-css-resource

In next articles I will try
to investigate where might be the problem and why it's not that easy to compile Angular-based apps with `esbuild`.
