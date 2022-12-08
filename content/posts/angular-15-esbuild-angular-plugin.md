---
title: "Angular 15 - esbuild `angular-compiler` plugin"
date: 2022-12-07
draft: false
---

In the two previous articles I tried to test and diagnose the `esbuild` builder for Angular.
It has been introduced recently in Angular 15, and it should fix my main problems with building time using ng cli.

Unfortunately after a few first tests, `@angular-devkit/build-angular:browser-esbuild`
was faster than `@angular-devkit/build-angular:browser`, but not as fast as advertised on the `esbuild` webpage.

In the second article, I found out that the configuration contains custom plugins added by Angular repository:
- `angular-compiler`
- `angular-global-styles`
- `angular-sass`
- `angular-css-resource`

The most impactful plugin is the `angular-compiler`. Please check [the previous article for details.](https://mythical-angular.dev/posts/angular-15-esbuild-plugins/)

![deeper](/images/deeper.jpeg)


Today I want to analyse for you what this plugin looks like and what is the slowest part. Enjoy! 

## Prepare

To fully understand the code below, it is good to read about `esbuild` plugins. How it's working, how to write one, etc.

[Here](https://esbuild.github.io/plugins/#using-plugins) you can find the docs for it. Focus on `setup` function and `build.onLoad` hook.

## Plugin code

```javascript
function createCompilerPlugin(pluginOptions, styleOptions) {
    return {
        name: 'angular-compiler',
        // eslint-disable-next-line max-lines-per-function
        async setup(build) {
            var _a, _b;
            var _c;
            let setupWarnings;
            // This uses a wrapped dynamic import to load `@angular/compiler-cli` which is ESM.
            // Once TypeScript provides support for retaining dynamic imports this workaround can be dropped.
            const { GLOBAL_DEFS_FOR_TERSER_WITH_AOT, NgtscProgram, OptimizeFor, readConfiguration } = await (0, load_esm_1.loadEsmModule)('@angular/compiler-cli');
            // Temporary deep import for transformer support
            const { mergeTransformers, replaceBootstrap, } = require('@ngtools/webpack/src/ivy/transformation');
            // Setup defines based on the values provided by the Angular compiler-cli
            (_a = (_c = build.initialOptions).define) !== null && _a !== void 0 ? _a : (_c.define = {});
            for (const [key, value] of Object.entries(GLOBAL_DEFS_FOR_TERSER_WITH_AOT)) {
                if (key in build.initialOptions.define) {
                    // Skip keys that have been manually provided
                    continue;
                }
                // esbuild requires values to be a string (actual strings need to be quoted).
                // In this case, all provided values are booleans.
                build.initialOptions.define[key] = value.toString();
            }

            // The tsconfig is loaded in setup instead of in start to allow the esbuild target build option to be modified.
            // esbuild build options can only be modified in setup prior to starting the build.
            const { options: compilerOptions, rootNames, errors: configurationDiagnostics, } = (0, profiling_1.profileSync)('NG_READ_CONFIG', () => readConfiguration(pluginOptions.tsconfig, {
                noEmitOnError: false,
                suppressOutputPathCheck: true,
                outDir: undefined,
                inlineSources: pluginOptions.sourcemap,
                inlineSourceMap: pluginOptions.sourcemap,
                sourceMap: false,
                mapRoot: undefined,
                sourceRoot: undefined,
                declaration: false,
                declarationMap: false,
                allowEmptyCodegenFiles: false,
                annotationsAs: 'decorators',
                enableResourceInlining: false,
            }));
            if (compilerOptions.target === undefined || compilerOptions.target < typescript_1.default.ScriptTarget.ES2022) {
                // If 'useDefineForClassFields' is already defined in the users project leave the value as is.
                // Otherwise fallback to false due to https://github.com/microsoft/TypeScript/issues/45995
                // which breaks the deprecated `@Effects` NGRX decorator and potentially other existing code as well.
                compilerOptions.target = typescript_1.default.ScriptTarget.ES2022;
                (_b = compilerOptions.useDefineForClassFields) !== null && _b !== void 0 ? _b : (compilerOptions.useDefineForClassFields = false);
                (setupWarnings !== null && setupWarnings !== void 0 ? setupWarnings : (setupWarnings = [])).push({
                    text: 'TypeScript compiler options "target" and "useDefineForClassFields" are set to "ES2022" and ' +
                        '"false" respectively by the Angular CLI.',
                    location: { file: pluginOptions.tsconfig },
                    notes: [
                        {
                            text: 'To control ECMA version and features use the Browerslist configuration. ' +
                                'For more information, see https://angular.io/guide/build#configuring-browser-compatibility',
                        },
                    ],
                });
            }
            // The file emitter created during `onStart` that will be used during the build in `onLoad` callbacks for TS files
            let fileEmitter;
            // The stylesheet resources from component stylesheets that will be added to the build results output files
            let stylesheetResourceFiles;
            let previousBuilder;
            let previousAngularProgram;
            const babelDataCache = new Map();
            const diagnosticCache = new WeakMap();
            build.onStart(async () => {
                const result = {
                    warnings: setupWarnings,
                };
                // Reset the setup warnings so that they are only shown during the first build.
                setupWarnings = undefined;
                // Reset debug performance tracking
                (0, profiling_1.resetCumulativeDurations)();
                // Reset stylesheet resource output files
                stylesheetResourceFiles = [];
                // Create TypeScript compiler host
                const host = typescript_1.default.createIncrementalCompilerHost(compilerOptions);
                // Temporarily process external resources via readResource.
                // The AOT compiler currently requires this hook to allow for a transformResource hook.
                // Once the AOT compiler allows only a transformResource hook, this can be reevaluated.
                host.readResource = async function (fileName) {
                    var _a, _b, _c;
                    // Template resources (.html/.svg) files are not bundled or transformed
                    if (fileName.endsWith('.html') || fileName.endsWith('.svg')) {
                        return (_a = this.readFile(fileName)) !== null && _a !== void 0 ? _a : '';
                    }
                    const { contents, resourceFiles, errors, warnings } = await (0, stylesheets_1.bundleStylesheetFile)(fileName, styleOptions);
                    ((_b = result.errors) !== null && _b !== void 0 ? _b : (result.errors = [])).push(...errors);
                    ((_c = result.warnings) !== null && _c !== void 0 ? _c : (result.warnings = [])).push(...warnings);
                    stylesheetResourceFiles.push(...resourceFiles);
                    return contents;
                };
                // Add an AOT compiler resource transform hook
                host.transformResource = async function (data, context) {
                    var _a, _b, _c;
                    // Only inline style resources are transformed separately currently
                    if (context.resourceFile || context.type !== 'style') {
                        return null;
                    }
                    // The file with the resource content will either be an actual file (resourceFile)
                    // or the file containing the inline component style text (containingFile).
                    const file = (_a = context.resourceFile) !== null && _a !== void 0 ? _a : context.containingFile;
                    const { contents, resourceFiles, errors, warnings } = await (0, stylesheets_1.bundleStylesheetText)(data, {
                        resolvePath: path.dirname(file),
                        virtualName: file,
                    }, styleOptions);
                    ((_b = result.errors) !== null && _b !== void 0 ? _b : (result.errors = [])).push(...errors);
                    ((_c = result.warnings) !== null && _c !== void 0 ? _c : (result.warnings = [])).push(...warnings);
                    stylesheetResourceFiles.push(...resourceFiles);
                    return { content: contents };
                };
                // Temporary deep import for host augmentation support
                const { augmentHostWithCaching, augmentHostWithReplacements, augmentProgramWithVersioning, } = require('@ngtools/webpack/src/ivy/host');
                // Augment TypeScript Host for file replacements option
                if (pluginOptions.fileReplacements) {
                    augmentHostWithReplacements(host, pluginOptions.fileReplacements);
                }
                // Augment TypeScript Host with source file caching if provided
                if (pluginOptions.sourceFileCache) {
                    augmentHostWithCaching(host, pluginOptions.sourceFileCache);
                    // Allow the AOT compiler to request the set of changed templates and styles
                    host.getModifiedResourceFiles = function () {
                        var _a;
                        return (_a = pluginOptions.sourceFileCache) === null || _a === void 0 ? void 0 : _a.modifiedFiles;
                    };
                }
                // Create the Angular specific program that contains the Angular compiler
                const angularProgram = (0, profiling_1.profileSync)('NG_CREATE_PROGRAM', () => new NgtscProgram(rootNames, compilerOptions, host, previousAngularProgram));
                previousAngularProgram = angularProgram;
                const angularCompiler = angularProgram.compiler;
                const typeScriptProgram = angularProgram.getTsProgram();
                augmentProgramWithVersioning(typeScriptProgram);
                const builder = typescript_1.default.createEmitAndSemanticDiagnosticsBuilderProgram(typeScriptProgram, host, previousBuilder, configurationDiagnostics);
                previousBuilder = builder;
                await (0, profiling_1.profileAsync)('NG_ANALYZE_PROGRAM', () => angularCompiler.analyzeAsync());
                const affectedFiles = (0, profiling_1.profileSync)('NG_FIND_AFFECTED', () => findAffectedFiles(builder, angularCompiler));
                if (pluginOptions.sourceFileCache) {
                    for (const affected of affectedFiles) {
                        pluginOptions.sourceFileCache.typeScriptFileCache.delete((0, node_url_1.pathToFileURL)(affected.fileName).href);
                    }
                }
                function* collectDiagnostics() {
                    // Collect program level diagnostics
                    yield* builder.getConfigFileParsingDiagnostics();
                    yield* angularCompiler.getOptionDiagnostics();
                    yield* builder.getOptionsDiagnostics();
                    yield* builder.getGlobalDiagnostics();
                    // Collect source file specific diagnostics
                    const optimizeFor = affectedFiles.size > 1 ? OptimizeFor.WholeProgram : OptimizeFor.SingleFile;
                    for (const sourceFile of builder.getSourceFiles()) {
                        if (angularCompiler.ignoreForDiagnostics.has(sourceFile)) {
                            continue;
                        }
                        // TypeScript will use cached diagnostics for files that have not been
                        // changed or affected for this build when using incremental building.
                        yield* (0, profiling_1.profileSync)('NG_DIAGNOSTICS_SYNTACTIC', () => builder.getSyntacticDiagnostics(sourceFile), true);
                        yield* (0, profiling_1.profileSync)('NG_DIAGNOSTICS_SEMANTIC', () => builder.getSemanticDiagnostics(sourceFile), true);
                        // Declaration files cannot have template diagnostics
                        if (sourceFile.isDeclarationFile) {
                            continue;
                        }
                        // Only request Angular template diagnostics for affected files to avoid
                        // overhead of template diagnostics for unchanged files.
                        if (affectedFiles.has(sourceFile)) {
                            const angularDiagnostics = (0, profiling_1.profileSync)('NG_DIAGNOSTICS_TEMPLATE', () => angularCompiler.getDiagnosticsForFile(sourceFile, optimizeFor), true);
                            diagnosticCache.set(sourceFile, angularDiagnostics);
                            yield* angularDiagnostics;
                        }
                        else {
                            const angularDiagnostics = diagnosticCache.get(sourceFile);
                            if (angularDiagnostics) {
                                yield* angularDiagnostics;
                            }
                        }
                    }
                }
                (0, profiling_1.profileSync)('NG_DIAGNOSTICS_TOTAL', () => {
                    var _a, _b;
                    for (const diagnostic of collectDiagnostics()) {
                        const message = convertTypeScriptDiagnostic(diagnostic, host);
                        if (diagnostic.category === typescript_1.default.DiagnosticCategory.Error) {
                            ((_a = result.errors) !== null && _a !== void 0 ? _a : (result.errors = [])).push(message);
                        }
                        else {
                            ((_b = result.warnings) !== null && _b !== void 0 ? _b : (result.warnings = [])).push(message);
                        }
                    }
                });
                fileEmitter = createFileEmitter(builder, mergeTransformers(angularCompiler.prepareEmit().transformers, {
                    before: [replaceBootstrap(() => builder.getProgram().getTypeChecker())],
                }), (sourceFile) => angularCompiler.incrementalCompilation.recordSuccessfulEmit(sourceFile));
                return result;
            });
            build.onLoad({ filter: compilerOptions.allowJs ? /\.[cm]?[jt]sx?$/ : /\.[cm]?tsx?$/ }, (args) => (0, profiling_1.profileAsync)('NG_EMIT_TS*', async () => {
                var _a, _b, _c, _d, _e, _f;
                assert.ok(fileEmitter, 'Invalid plugin execution order');
                const request = (_b = (_a = pluginOptions.fileReplacements) === null || _a === void 0 ? void 0 : _a[args.path]) !== null && _b !== void 0 ? _b : args.path;
                // The filename is currently used as a cache key. Since the cache is memory only,
                // the options cannot change and do not need to be represented in the key. If the
                // cache is later stored to disk, then the options that affect transform output
                // would need to be added to the key as well as a check for any change of content.
                let contents = (_c = pluginOptions.sourceFileCache) === null || _c === void 0 ? void 0 : _c.typeScriptFileCache.get((0, node_url_1.pathToFileURL)(request).href);
                if (contents === undefined) {
                    const typescriptResult = await fileEmitter(request);
                    if (!typescriptResult) {
                        // No TS result indicates the file is not part of the TypeScript program.
                        // If allowJs is enabled and the file is JS then defer to the next load hook.
                        if (compilerOptions.allowJs && /\.[cm]?js$/.test(request)) {
                            return undefined;
                        }
                        // Otherwise return an error
                        return {
                            errors: [
                                createMissingFileError(request, args.path, (_d = build.initialOptions.absWorkingDir) !== null && _d !== void 0 ? _d : ''),
                            ],
                        };
                    }
                    const data = (_e = typescriptResult.content) !== null && _e !== void 0 ? _e : '';
                    // The pre-transformed data is used as a cache key. Since the cache is memory only,
                    // the options cannot change and do not need to be represented in the key. If the
                    // cache is later stored to disk, then the options that affect transform output
                    // would need to be added to the key as well.
                    contents = babelDataCache.get(data);
                    if (contents === undefined) {
                        const transformedData = await transformWithBabel(request, data, pluginOptions);
                        contents = Buffer.from(transformedData, 'utf-8');
                        babelDataCache.set(data, contents);
                    }
                    (_f = pluginOptions.sourceFileCache) === null || _f === void 0 ? void 0 : _f.typeScriptFileCache.set((0, node_url_1.pathToFileURL)(request).href, contents);
                }
                return {
                    contents,
                    loader: 'js',
                };
            }, true));
            build.onLoad({ filter: /\.[cm]?js$/ }, (args) => (0, profiling_1.profileAsync)('NG_EMIT_JS*', async () => {
                var _a, _b;
                // The filename is currently used as a cache key. Since the cache is memory only,
                // the options cannot change and do not need to be represented in the key. If the
                // cache is later stored to disk, then the options that affect transform output
                // would need to be added to the key as well as a check for any change of content.
                let contents = (_a = pluginOptions.sourceFileCache) === null || _a === void 0 ? void 0 : _a.babelFileCache.get(args.path);
                if (contents === undefined) {
                    const data = await fs.readFile(args.path, 'utf-8');
                    const transformedData = await transformWithBabel(args.path, data, pluginOptions);
                    contents = Buffer.from(transformedData, 'utf-8');
                    (_b = pluginOptions.sourceFileCache) === null || _b === void 0 ? void 0 : _b.babelFileCache.set(args.path, contents);
                }
                return {
                    contents,
                    loader: 'js',
                };
            }, true));
            build.onEnd((result) => {
                var _a;
                if (stylesheetResourceFiles.length) {
                    (_a = result.outputFiles) === null || _a === void 0 ? void 0 : _a.push(...stylesheetResourceFiles);
                }
                (0, profiling_1.logCumulativeDurations)();
            });
        },
    };
}
```

OH MY... IT'S SO LONG AND COMPLICATED!

It is, but there are a lot of source code lines in four main "sections":
- `build.onStart()` - a hook that is called **only once** on the build start
- `build.onLoad()` - with filter set to `compilerOptions.allowJs ? /\.[cm]?[jt]sx?$/ : /\.[cm]?tsx?$/`, so this is the one which compiles all the Typescript files
- `build.onLoad()` - with filter set to `/\.[cm]?js$/`, for Javascript files
- `build.onEnd()` - a hook called **only once** on the end of the build

From documentation and experiments with `esbuild` I know that `build.onLoad()` hooks are called multiple times.
It's harder to measure time for them, so I've started with the rest of the hooks first. 

The results that I got are pretty interesting:
```text
> ~/D/experiment-angular-15-esbuild main yarn build
yarn run v1.22.17
$ ng build
The esbuild browser application builder ('browser-esbuild') is currently experimental.
The 'extractLicenses' option is currently unsupported by this experimental builder and will be ignored.
The 'progress' option is currently unsupported by this experimental builder and will be ignored.
build.onStart() 116283 ms
build.onEnd() 0 ms
Complete. [129.851 seconds]
✨  Done in 131.65s.
```

For some reason, the first called hook is the slowest one.
Knowing that, the rest of the research is a lot easier to do.
I decided to do fast bisection on that function, and after few tries I found the slowest line of code.
I modified the code to measure it, and everything you can find in a code box below.
```typescript
// build.onStart(build.onStart(async () => { ...
const builder = typescript_1.default.createEmitAndSemanticDiagnosticsBuilderProgram(typeScriptProgram, host, previousBuilder, configurationDiagnostics);
previousBuilder = builder;


const profilingStart = new Date().getTime();


await (0, profiling_1.profileAsync)('NG_ANALYZE_PROGRAM', () => angularCompiler.analyzeAsync());


const profilingEnd = new Date().getTime();
console.log('NG_ANALYZE_PROGRAM', profilingEnd - profilingStart, 'ms'); // <- Interesting part


const affectedFiles = (0, profiling_1.profileSync)('NG_FIND_AFFECTED', () => findAffectedFiles(builder, angularCompiler));
if (pluginOptions.sourceFileCache) {
    for (const affected of affectedFiles) {
        pluginOptions.sourceFileCache.typeScriptFileCache.delete((0, node_url_1.pathToFileURL)(affected.fileName).href);
    }
}
// ...
```
```text
> ~/D/experiment-angular-15-esbuild main yarn build
yarn run v1.22.17
$ ng build
The esbuild browser application builder ('browser-esbuild') is currently experimental.
The 'extractLicenses' option is currently unsupported by this experimental builder and will be ignored.
The 'progress' option is currently unsupported by this experimental builder and will be ignored.
NG_ANALYZE_PROGRAM 127212 ms
build.onStart() 141464 ms
build.onEnd() 0 ms
Complete. [159.394 seconds]
✨  Done in 160.80s.
```

It looks like `angularCompiler.analyzeAsync()` took 127,2s (about 80%) of the whole 159.4s building time!

## `NgCompiler.analyzeAsync()`

I don't want to do the complete analysis of that method (because I'm not prepared to do this... yet),
but I think that it's worth posting at least the comment that is above this method:
```typescript
/**
 * Perform Angular's analysis step (as a precursor to `getDiagnostics` or `prepareEmit`)
 * asynchronously.
 *
 * Normally, this operation happens lazily whenever `getDiagnostics` or `prepareEmit` are called.
 * However, certain consumers may wish to allow for an asynchronous phase of analysis, where
 * resources such as `styleUrls` are resolved asynchronously. In these cases `analyzeAsync` must
 * be called first, and its `Promise` awaited prior to calling any other APIs of `NgCompiler`.
 */
analyzeAsync(): Promise<void>;
```

## Summary

I found
that `esbuild` builder for Angular is using the same `NgCompiler` that we know from standard Webpack-based builder.
This might be the problem.

When I first time heard about `esbuild` I learned that it is so fast, because it has been written in go.
Go is an extremely fast language, and it is able to process code using threads.
Javascript can use only one thread, so it must be slower. 

For now `NgCompiler` is written in Javascript, and it's slow.
I have a hope that it will eventually change in next versions,
because `esbuild`, `swc`
and `vite` proved that it's worth going outside the Javascript comfort zone when you want
to have a faster building processes. 
