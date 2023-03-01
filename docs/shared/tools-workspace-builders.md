{% callout type="check" title="Local Nx plugins" %}
In Nx 13.10+, local nx plugins can contain executors that are used in the workspace. When creating a custom executor for your workspace, look into the [nx-plugin guide](/packages/nx-plugin) to simplify the build process.
{% /callout %}

# Creating Custom Executors

Creating Executors for your workspace standardizes scripts that are run during your development/building/deploying tasks in order to enable Nx's `affected` command and caching capabilities.

This guide shows you how to create, run, and customize executors within your Nx workspace. The examples use the trivial use-case of an `echo` command.

## Creating an executor

Your executor should be created within the `tools` directory of your Nx workspace like so:

```treeview
happynrwl/
├── apps/
├── libs/
├── tools/
│   └── executors/
│       └── echo/
│           ├── executor.json
│           ├── impl.ts
│           ├── package.json
│           └── schema.json
├── nx.json
├── package.json
└── tsconfig.base.json
```

### schema.json

This file describes the options being sent to the executor (very similar to the `schema.json` file of generators). Setting the `cli` property to `nx` indicates that you're using the Nx Devkit to make this executor.

```json
{
  "$schema": "http://json-schema.org/schema",
  "type": "object",
  "cli": "nx",
  "properties": {
    "textToEcho": {
      "type": "string",
      "description": "Text To Echo"
    }
  }
}
```

This example describes a single option for the executor that is a `string` called `textToEcho`. When using this executor, specify a `textToEcho` property inside the options.

In our `impl.ts` file, we're creating an `Options` interface that matches the json object being described here.

### impl.ts

The `impl.ts` contains the actual code for your executor. Your executor's implementation must export a function that takes an options object and returns a `Promise<{ success: boolean }>`.

```typescript
import type { ExecutorContext } from '@nrwl/devkit';
import { exec } from 'child_process';
import { promisify } from 'util';

export interface EchoExecutorOptions {
  textToEcho: string;
}

export default async function echoExecutor(
  options: EchoExecutorOptions,
  context: ExecutorContext
): Promise<{ success: boolean }> {
  console.info(`Executing "echo"...`);
  console.info(`Options: ${JSON.stringify(options, null, 2)}`);

  const { stdout, stderr } = await promisify(exec)(
    `echo ${options.textToEcho}`
  );
  console.log(stdout);
  console.error(stderr);

  const success = !stderr;
  return { success };
}
```

### executor.json

The `executor.json` file provides the description of your executor to the CLI.

```json
{
  "executors": {
    "echo": {
      "implementation": "./impl",
      "schema": "./schema.json",
      "description": "Runs `echo` (to test executors out)."
    }
  }
}
```

Note that this `executor.json` file is naming our executor 'echo' for the CLI's purposes, and mapping that name to the given implementation file and schema.

### package.json

This is all that’s required from the `package.json` file:

```json
{
  "executors": "./executor.json"
}
```

## Compiling and Running your Executor

After your files are created, compile your executor with `tsc` (which is available locally in any Nx workspace):

```bash
npx tsc tools/executors/echo/impl
```

This will create the `impl.js` file in your file directory, which will serve as the artifact used by the CLI.

Our last step is to add this executor to a given project’s `targets` object in your project's `project.json` file:

```json
{
  //...
  "targets": {
    "build": {
      // ...
    },
    "serve": {
      // ...
    },
    "lint": {
      // ,,,
    },
    "echo": {
      "executor": "./tools/executors/echo:echo",
      "options": {
        "textToEcho": "Hello World"
      }
    }
  }
}
```

Note that the format of the `executor` string here is: `${Path to directory containing the executor's package.json}:${executor name}`.

Finally, you run the executor via the CLI as follows:

```bash
nx run platform:echo
```

To which we'll see the console output:

```bash
> nx run platform:echo
Executing "echo"...
Options: {
  "textToEcho": "Hello World"
}
Hello World
```

## Debugging Executors

As part of Nx's computation cache process, Nx forks the node process, which can make it difficult to debug an executor command. Follow these steps to debug an executor:

1. Use VS Code's command pallette to open a `Javascript Debug Terminal`
2. Find the compiled (`*.js`) executor code and set a breakpoint.
3. Run the executor in the debug terminal

```bash
nx run platform:echo
```

## Using Node Child Process

[Node’s `childProcess`](https://nodejs.org/api/child_process.html) is often useful in executors.

Part of the power of the executor API is the ability to compose executors via existing targets. This way you can combine other executors from your workspace into one which could be helpful when the process you’re scripting is a combination of other existing executors provided by the CLI or other custom executors in your workspace.

Here's an example of this (from a hypothetical project), that serves an api (project name: "api") in watch mode, then serves a frontend app (project name: "web-client") in watch mode:

```typescript
import { ExecutorContext, runExecutor } from '@nrwl/devkit';

export interface MultipleExecutorOptions {}

export default async function multipleExecutor(
  options: MultipleExecutorOptions,
  context: ExecutorContext
): Promise<{ success: boolean }> {
  const result = await Promise.race([
    await runExecutor(
      { project: 'api', target: 'serve' },
      { watch: true },
      context
    ),
    await runExecutor(
      { project: 'web-client', target: 'serve' },
      { watch: true },
      context
    ),
  ]);
  for await (const res of result) {
    if (!res.success) return res;
  }

  return { success: true };
}
```

For other ideas on how to create your own executors, you can always check out Nx's own open-source executors as well!

(For example, our [cypress executor](https://github.com/nrwl/nx/blob/master/packages/cypress/src/executors/cypress/cypress.impl.ts))

## Using Custom Hashers

For most executors, the default hashing in Nx makes sense. The output of the executor is dependent on the files in the project that it is being run for, or that project's dependencies, and nothing else. Changing a miscellaneous file at the workspace root will not affect that executor, and changing _*any*_ file inside of the project may affect the executor. When dealing with targets which only depend on a small subset of the files in a project, or may depend on arbitrary data that is not stored within the project, the default hasher may not make sense anymore. In these cases, the target will either experience more frequent cache misses than necessary or not be able to be cached.

Executors can provide a custom hasher that Nx uses when determining if a target run should be a cache hit, or if it must be run. When generating an executor for a plugin, you can use `nx g @nrwl/nx-plugin:executor my-executor --project my-plugin --includeHasher` to automatically add a custom hasher.

If you want to add a custom hasher manually, create a new file beside your executor's implementation. We will use `hasher.ts` as an example here. You'll also need to update `executors.json`, so that it resembles something like this:

```json
{
  "executors": {
    "echo": {
      "implementation": "./src/executors/my-executor/executor",
      "hasher": "./src/executors/my-executor/hasher",
      "schema": "./src/executors/my-executor/schema.json"
    }
  }
}
```

This would allow you to write a custom function in `hasher.ts`, which Nx would use to calculate the target's hash. As an example, consider the below hasher which mimics the behavior of Nx's default hashing algorithm.

```typescript
import { CustomHasher, Task, HasherContext } from '@nrwl/devkit';

export const mimicNxHasher: CustomHasher = async (
  task: Task,
  context: HasherContext
) => {
  return context.hasher.hashTask(task);
};

export default mimicNxHasher;
```

The hash function can do anything it wants, but it is important to remember that the hasher replaces the hashing done normally by Nx. If you change the hasher, Nx may return cache hits when you do not anticipate it. Imagine the below custom hasher:

```typescript
import { CustomHasher, Task, HasherContext } from '@nrwl/devkit';

export const badHasher: CustomHasher = async (
  task: Task,
  context: HasherContext
) => {
  return {
    value: 'my-static-hash',
  };
};

export default badHasher;
```

This hasher would never return a different hash, so every run of a task that consumes the executor would be a cache hit. It is important that anything that would change the result of your executor's implementation is accounted for in the hasher.