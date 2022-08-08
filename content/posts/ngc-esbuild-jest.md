---
title: "It looks that TestBed works with esbuild"
date: 2022-08-08
draft: false
---

In previous post ["esbuild against TestBed"](https://mythical-angular.dev/posts/jest/) I said
that `TestBed` is not working when we use esbuild to bundle tests.
It seems like it's possible to work around the problems.

I said also that I'm not sure what to think about the `TestBed`, is it worth using it or not.
Every day, I'm closer to getting my own opinion.
But this is a subject for the next post. 

Today we will focus on how to compile our tests with esbuild and still be able to use `TestBed`.

## Repository

As always, code you can find in my repository [ng-esbuild-tests](https://github.com/galczo5/ng-esbuild-tests). 

I had to decompose and change the pipeline.
This time, esbuild is not used as a Jest transformer, but the compilation is executed before running the tests.
In this case, Jest doesn't need to do any transformation.
I had to do this because it was impossible to use esbuild plugins and provide a sync transformation for Jest.
For now, esbuild is supporting plugins only in the async build.

The only problem that I decided not to resolve now, is that there is no caching available.

In addition to the code, I've created a repository with example unit tests using the `TestBed`. Here it is: [ng-esbuild-unit-tests](https://github.com/galczo5/ng-esbuild-unit-tests).

## Test results

Today I will put the results of the tests before the code.
There is only one initial condition.
Because I don't have a caching mechanism in my code, I decided to disable cache in Jest with ts-jest.
Thanks to that, tests are comparable. 

I have only 16 tests, all of them are using `TestBed` to create a component.

| No. | ts-jest | esbuild | no compilation (working on result from esbuild) |
|-----|---------|---------|-------------------------------------------------|
| 1   | 15.86s  | 13.90s  | 10.45s                                          |
| 2   | 16.07s  | 11.09s  | 10.48s                                          |
| 3   | 15.98s  | 11.79s  | 13.57s                                          |

As you can see, results are interesting.
It's obvious that the compilation is not the most time-consuming part of the tests.
The problem is that, even if I skip the compilation part,
and use the results of the previous compilation, the tests are slow.

You may say, that 10s or 12s is not that long.
Let's do the math.
I was able to run 16 tests in about 12s.
In this case, one test took me 0.75s.

So, for an app that contains 300 components,
300 injectables and 100 directives if you want to have a test for every single object, you will spend about 525s.
**It's almost 9 minutes!**.
And having 700 tests, it is not that unusual.

We all heard about testing pyramid.
In theory, we should be able to have a huge number of unit tests, because they are fast.

## The code

After the previous section,
you know that when you use `TestBed` there is not a lot of the difference between the ts-jest and esbuild.
Anyway, I want to show you how I implemented it, despite there is no sense of using it.

Let's start with the CLI.

```typescript
import { program } from 'commander';
import { runCLI } from 'jest';
import { getPaths } from './getPaths';
import { compile } from './compile';

program
	.name('ng-esbuild-tests')
	.description('Run Angular tests faster thanks to Jest and esbuild')
	.option('--dir [dir]', 'directory to scan, default: current directory')
	.option('--pattern [pattern]', 'file pattern, default: ./**/*spec.ts')
	.option('--outDir [outDir]', 'output directory, default: ./dist/ng-esbuild-tests/')
	.option('--skipCompile', 'skip compilation part, only for tests!')
	.parse(process.argv);

export async function cli() {
	const opts = program.opts();
	const dir = opts.dir || './';
	const pattern = opts.pattern || '**/*spec.ts';
	const outDir = opts.outDir || './dist/ng-esbuild-tests/';
	const skipCompile = opts.skipCompile || '';

	const paths = await getPaths(dir, pattern);

	if (!skipCompile) {
		console.log('Files to compile with esbuild:');
		// eslint-disable-next-line no-restricted-syntax
		for (const path of paths) {
			console.log(`  ${path}`);
		}

		await compile(outDir, paths);
	} else {
		console.warn('Compilation skipped!');
	}

	const args = {
		roots: ['./'],
		testRegex: '\\.spec\\.js$',
		testEnvironment: 'jsdom'
	};

	await runCLI(args as any, [outDir]);
}
```

To make my life easier, I create a simple CLI using `commander`.
It shows very well the process.

First of all, I get the CLI arguments and replace with the defaults if necessary.
To run tests I had to get the paths to the test files, it's implemented in `getPaths` function.

```typescript
import { glob } from 'glob';
import { join } from 'path';

export async function getPaths(dir, patten) {
	return new Promise<Array<string>>((resolve, reject) => {
		glob(join(dir, patten), (err, files) => {
			if (err) {
				reject(err);
				return;
			}

			resolve(files);
		});
	});
}
```

After that, I'm using the esbuild to compile the test files.

```typescript
import { build } from 'esbuild';
import { componentMetadataPlugin } from './componentMetadataPlugin';
import { jestPresetAngularPlugin } from './jestPresetAngularPlugin';

export async function compile(outDir: string, paths: Array<string>) {
	const start = new Date().getTime();

	const result = await build({
		outdir: outDir,
		entryPoints: paths,
		bundle: true,
		platform: 'node',
		minify: false,
		format: 'iife',
		plugins: [
			componentMetadataPlugin(),
			jestPresetAngularPlugin()
		],
		loader: {
			'.html': 'text'
		}
	});

	const end = new Date().getTime();
	console.log(`\nCompilation end in ${end - start}ms!\n`);

	return result;
}
```

Here I had to implement two plugins for esbuild.
Fortunately for me, I could use the plugins created in repository [cherryApp/ngc-esbuild](https://github.com/cherryApp/ngc-esbuild).
I really recommend investigating what the owned did there.
Really inspiring stuff. 

So, the first, `jestPresetAngularPlugin` is pretty simple.
It's adding the necessary import as the first of the spec file.

```typescript
import * as fs from 'fs';
import {
	Loader, OnLoadResult, Plugin, PluginBuild
} from 'esbuild';

const tsLoader: Loader = 'ts';

function processSpecFile(source: string): OnLoadResult {
	try {
		if (!source.includes('import \'jest-preset-angular/setup-jest\'')) {
			const result = `import 'jest-preset-angular/setup-jest';\n${source}`;
			return { contents: result, loader: 'ts' };
		}

		return { contents: source, loader: tsLoader };
	} catch (e) {
		return { errors: [e] };
	}
}

export function jestPresetAngularPlugin(): Plugin {
	return {
		name: 'jestPresetAngularPlugin',
		async setup(build: PluginBuild) {
			build.onLoad({ filter: /.*\.(spec)\.ts$/ }, async (args) => {
				const source = await fs.promises.readFile(args.path, 'utf8');
				return processSpecFile(source);
			});
		}
	};
}
```

`import 'jest-preset-angular/setup-jest';` is very important. It provides all the things required by the Angular. Stuff like `zone.js` etc.

The second plugin is more complicated.

```typescript
import * as fs from 'fs';
import {
	Loader, OnLoadResult, Plugin, PluginBuild
} from 'esbuild';

const tsLoader: Loader = 'ts';
const correctNumberOfParts = 2;

const styleUrlsRegExp = /^ *styleUrls *: *\[['"]([^'"\]]*)['"]],*/gm;
const templateUrlRegExp = /^ *templateUrl *: *['"]*([^'"]*)['"]/gm;

function getValueByPattern(regex: RegExp, str: string) {
	const results: Array<string> = [];
	let execArray = regex.exec(str);

	while (execArray !== null) {
		results.push(execArray[1]);
		execArray = regex.exec(str);
	}

	return results.pop();
}

async function removeStyles(contents: string) {
	return contents.replace(styleUrlsRegExp, 'styles: [],');
}

function processConstructor(contents) {
	if (!/constructor *\(([^)]*)/gm.test(contents)) {
		return contents;
	}

	let result = contents;
	const matches = result.matchAll(/constructor *\(([^)]*)/gm);
	// eslint-disable-next-line no-restricted-syntax
	for (const match of matches) {
		if (!match[1] && /:/gm.test(match[1])) {
			continue;
		}

		const flat = match[1].replace(/[\n\r]/gm, '');
		const flatArray = flat.split(',')
			.map((inject) => {
				const parts = inject.split(':');
				const partsCorrect = parts.length === correctNumberOfParts;
				const providerDoesNotContainInject = !/@Inject/.test(inject);
				const shouldAddInject = partsCorrect && providerDoesNotContainInject;
				return shouldAddInject ? `@Inject(${parts[1]}) ${inject}` : inject;
			});

		const constructorRegExp = /constructor *\([^)]*\)/gm;
		result = result.replace(constructorRegExp, `constructor(${flatArray.join(',')})`);
	}

	if (!/Inject[ ,}\n\r].*'@angular\/core.*;/.test(result)) {
		result = `import { Inject } from '@angular/core';\n${result}`;
	}

	return result;
}

async function processComponentMetadata(source: string): Promise<OnLoadResult> {
	try {
		let result = source;

		const hasTemplateUrl = /^ *templateUrl *: *['"]*([^'"]*)/gm.test(result);
		if (hasTemplateUrl) {
			const templateUrl = getValueByPattern(/^ *templateUrl *: *['"]*([^'"]*)/gm, source);
			result = `import templateSource from '${templateUrl}';\n${result}`
				.replace(templateUrlRegExp, 'template: templateSource || ""');
		}

		result = await removeStyles(result || '');
		result = processConstructor(result);

		return { contents: result, loader: tsLoader };
	} catch (e) {
		return { errors: [e] };
	}
}

export function componentMetadataPlugin(): Plugin {
	return {
		name: 'componentMetadataPlugin',
		async setup(build: PluginBuild) {
			build.onLoad({ filter: /src.*\.(component|pipe|service|directive|guard|module)\.ts$/ }, async (args) => {
				const source = await fs.promises.readFile(args.path, 'utf8');
				return await processComponentMetadata(source);
			});
		}
	};
}
```

It does three things:
- removes the `styleUrls` property of the `@Component` decorator. We are not rendering the component in the browser, so there is no need to keep it.
- adds `@Inject` decorator to all arguments in the components, services, pipes, directives, guards and modules. Without this action, Jest is not able to inject anything to your objects.
- replaces `templateUrl` with `template` and import to the files. The bundler loads the template in this case as a text, and then assigns it to the component.

After the compilation, in the `dist` directory I have tests compile, and the only last thing to do is to start Jest.

```typescript
    // ...
	const args = {
        roots: ['./'],
        testRegex: '\\.spec\\.js$',
        testEnvironment: 'jsdom'
    };
    
    await runCLI(args as any, [outDir]);
    // ...
```

## Summary

Basically, this is it.

As I mentioned, probably, there is no sense to fight with `TestBed`.
It's slow.

![](/images/jest-preset-angular.png)

PS. `jest-preset-angular` has support for the esbuild.
It compiles all of the `.mjs` files with esbuild and additional pointed files. 

```typescript
/**
 * Some NPM packages like `tslib` is distributed in such a way that `esbuild` cannot process it, so we fall back to use TypeScript API
 */
const defaultProcessWithEsbuildPatterns = ['**/*.mjs'];

export class NgJestConfig extends ConfigSet {
    readonly processWithEsbuild: ReturnType<typeof globsToMatcher>;

    constructor(jestConfig: ProjectConfigTsJest | undefined, parentLogger?: Logger | undefined) {
        super(jestConfig, parentLogger);
        const jestGlobalsConfig = jestConfig?.globals?.ngJest ?? Object.create(null);
        this.processWithEsbuild = globsToMatcher([
            ...(jestGlobalsConfig.processWithEsbuild ?? []),
            ...defaultProcessWithEsbuildPatterns,
        ]);
    }
    // ...
}

export class NgJestTransformer extends TsJestTransformer {
    
    // ...
    
    process(fileContent: string, filePath: string, transformOptions: TransformOptionsTsJest): TransformedSource {
        // @ts-expect-error we are accessing the private cache to avoid creating new objects all the time
        const configSet = super._configsFor(transformOptions);
        if (configSet.processWithEsbuild(filePath)) {
            this.#ngJestLogger.debug({ filePath }, 'process with esbuild');

            const compilerOpts = configSet.parsedTsConfig.options;
            const { code, map } = this.#esbuildImpl.transformSync(fileContent, {
                loader: 'js',
                format: transformOptions.supportsStaticESM && configSet.useESM ? 'esm' : 'cjs',
                target: compilerOpts.target === configSet.compilerModule.ScriptTarget.ES2015 ? 'es2015' : 'es2016',
                sourcemap: compilerOpts.sourceMap,
                sourcefile: filePath,
                sourcesContent: true,
                sourceRoot: compilerOpts.sourceRoot,
            });

            return {
                code,
                map,
            };
        } else {
            return super.process(fileContent, filePath, transformOptions);
        }
    }
}

```

  
