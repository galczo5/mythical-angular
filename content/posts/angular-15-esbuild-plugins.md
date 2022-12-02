---
title: "Angular 15 - esbuild plugins"
date: 2022-12-02
draft: false
---

Sooooo... 

In the [previous post](https://mythical-angular.dev/posts/angular-15-esbuild/) I found out that new Angular 15 esbuild builder is usable, but not as fast as I imagined. I showed you a place in the code where esbuild is configured, and we found custom plugins to compile Angular code. 

Today I'll try to find which plugin is problematic.

## Code modifications

The easies way I know to decide which plugin is the most important one, is to modify esbuild configuration to contain only one plugin at the time.

To do it access file `node_modules/@angular-devkit/build-angular/src/builders/browser-esbuild/esbuild.js` and modify function `bundle`.

```javascript
async function bundle(workspaceRoot, optionsOrInvalidate) {
    var _a, _b;
    let result;
    try {
        if (typeof optionsOrInvalidate === 'function') {
            result = (await optionsOrInvalidate());
        }
        else {
          const enabledPlugins = [
            // 'angular-compiler',
            // 'angular-global-styles',
            // 'angular-sass',
            // 'angular-css-resource',
          ];

          const config = {
            ...optionsOrInvalidate,
            metafile: true,
            write: false,
            plugins: optionsOrInvalidate.plugins.filter(p => enabledPlugins.includes(p.name))
          };

          if (config.entryPoints.styles) {
            config.entryPoints = {};
          }

          console.log(config);
          result = await (0, esbuild_1.build)(config);
        }
    }
    catch (failure) {
        // Build failures will throw an exception which contains errors/warnings
        if (isEsBuildFailure(failure)) {
            return failure;
        }
        else {
            throw failure;
        }
    }
    const initialFiles = [];
    for (const outputFile of result.outputFiles) {
        // Entries in the metafile are relative to the `absWorkingDir` option which is set to the workspaceRoot
        const relativeFilePath = (0, node_path_1.relative)(workspaceRoot, outputFile.path);
        const entryPoint = (_b = (_a = result.metafile) === null || _a === void 0 ? void 0 : _a.outputs[relativeFilePath]) === null || _b === void 0 ? void 0 : _b.entryPoint;
        outputFile.path = relativeFilePath;
        if (entryPoint) {
            // An entryPoint value indicates an initial file
            initialFiles.push({
                file: outputFile.path,
                // The first part of the filename is the name of file (e.g., "polyfills" for "polyfills.7S5G3MDY.js")
                name: (0, node_path_1.basename)(outputFile.path).split('.')[0],
                extension: (0, node_path_1.extname)(outputFile.path),
            });
        }
    }
    return { ...result, initialFiles };
}
```

The only thing now I need to do is to uncomment the plugin I want to test.

## Test results

Even without tests it's easy to predict that the most time-consuming plugin should be `angular-compiler`. 

| Plugin Enabled        | Time    |
|-----------------------|---------|
| No Plugins            | 1.82s   |
| angular-css-resource  | 1.93s   |
| angular-sass          | 1.98s   |
| angular-global-styles | 1.86s   |
| angular-compiler      | 107.70s |

## Summary

Now I know on which file I should focus first, it's `node_modules/@angular-devkit/build-angular/src/builders/browser-esbuild/compiler-plugin.js`, function starts at line 140.
