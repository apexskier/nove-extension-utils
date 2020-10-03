# `nova-extension-utils`

This package provides sharable utilities for building [Nova](http://nova.app) extensions.

## Features

### `installWrappedDependencies`

This function provides a concurrent-safe, reproducible mechanism for installing external nodejs dependencies executed by your extension, without bundling them within the extension artifact and increasing extension size.

To use it, you _must_ have a valid [`package.json`](https://docs.npmjs.com/files/package.json) and [`npm-shrinkwrap.json`](https://docs.npmjs.com/configuring-npm/shrinkwrap-json.html) file in your `.novaextension` directory. They'll be copied into your extension's global storage and used to install dependencies specified in the shrinkwrap file. An atomic file lock is used to prevent multiple workspaces writing over each other.

#### `installWrappedDependencies`

This is an async function that handles installation. Call it during extension activation. It takes a single parameter: a [composite disposable](https://docs.nova.app/api-reference/composite-disposable/) that on disposal kills the installation process and unlocks. Be sure to dispose when the extension deactivates.

#### `registerDependencyUnlockCommand`

Registers a global command that will force unlock. Not required, but can be useful to cleanup if crashes happen. Make sure to [define the command](https://docs.nova.app/extensions/commands/) to give users access.

#### `dependencyDirectory`

The path to the directory containing the installed `node_modules` directory.

<details>

<summary>Here's a full example of usage</summary>

```ts
import { dependencyManagement } from "nova-extension-utils";

const compositeDisposable = new CompositeDisposable();

dependencyManagement.registerDependencyUnlockCommand(
  "com.example.extension.unlock"
);

async function asyncActivate() {
  await dependencyManagement.installWrappedDependencies(compositeDisposable);

  const execPath = nova.path.join(
    dependencyManagement.dependencyDirectory,
    "node_modules",
    ".bin",
    "executable"
  );
  const process = new Process(execPath);
  compositeDisposable.add({
    dispose() {
      process.terminate();
    },
  });
  process.start();
}

export function activate() {
  console.log("activating...");
  return asyncActivate()
    .catch((err) => {
      console.error(err);
    })
    .then(() => {
      console.log("activated");
    });
}

export function deactivate() {
  compositeDisposable.dispose();
}
```

</details>

### `asyncNova`

#### `showChoicePalette`

Asyncronous, generic, non-index-based access to the choice palette.

<details>

<summary>Example of use</summary>

```ts
import type * as lspTypes from "vscode-languageserver-protocol";
import { asyncNova } from "nova-extension-utils";

async function foo(items: lspTypes.CompletionItem[]) {
  const choice: lspTypes.CompletionItem | null = await asyncNova(
    items,
    (item) => `${item.label}${item.detail ? `- ${item.detail}` : ""}`,
    { placeholder: "suggestions" }
  );
  if (!choice) {
    return;
  }
  console.log(choice);
}
```

</details>
