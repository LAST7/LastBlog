---
title: Full Stack Open Part9
date: 2024-07-08 17:52:12
categories: 技术
tags:
  - JavaScript
  - TypeScript
  - ESLint
excerpt: Notes taken for Part9 of Full Stack Open
---

## Preparation

### Type Definition for Third Party Libraries

-   In TypeScript, we want every variable to have a type defined in order to perform further static checks.
-   But in some cases, we would need third party libraries which are not written in TypeScript(therefore don't contain types).
-   Under this circumstance, we could look for the community-maintained type packages, for example, `express` and `cors`.

    ```bash
    yarn add express
    yarn add -D @types/express
    yarn add -D cors @types/cors
    ```

### Eslint

-   To configure a project, for example an Express.js backend, we will need these packages installed:

    ```bash
    yarn add -D eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser
    ```

-   Then, create a `eslint.config.mjs` as the config file for eslint:

    ```js
    import typescriptEslint from "@typescript-eslint/eslint-plugin";
    import globals from "globals";
    import tsParser from "@typescript-eslint/parser";
    import path from "node:path";
    import { fileURLToPath } from "node:url";
    import js from "@eslint/js";
    import { FlatCompat } from "@eslint/eslintrc";

    const **filename = fileURLToPath(import.meta.url);
    const **dirname = path.dirname(**filename);
    const compat = new FlatCompat({
    baseDirectory: **dirname,
    recommendedConfig: js.configs.recommended,
    allConfig: js.configs.all,
    });

    export default [
        ...compat.extends(
        "eslint:recommended",
        "plugin:@typescript-eslint/recommended",
        "plugin:@typescript-eslint/recommended-requiring-type-checking",
        ),
        {
            plugins: {
            "@typescript-eslint": typescriptEslint,
            },
            languageOptions: {
                globals: {
                    ...globals.browser,
                    ...globals.node,
                },

                parser: tsParser,
                ecmaVersion: 5,
                sourceType: "commonjs",

                parserOptions: {
                    project: "./tsconfig.json",
                },
            },
            rules: {
                "@typescript-eslint/semi": ["error"],
                "@typescript-eslint/explicit-function-return-type": "off",
                "@typescript-eslint/explicit-module-boundary-types": "off",
                "@typescript-eslint/restrict-template-expressions": "off",
                "@typescript-eslint/restrict-plus-operands": "off",
                "@typescript-eslint/no-unsafe-member-access": "off",

                "@typescript-eslint/no-unused-vars": [
                    "error",
                    {
                        argsIgnorePattern: "^_",
                    },
                ],
                "no-case-declarations": "off",
            },
        },
        {
            ignores: ["eslint.config.mjs", "node_modules/", "build/"],
        },
    ];
    ```

-   For more information about configuration of Eslint above version 9.0, refer to [Configure ESLint](https://eslint.org/docs/latest/use/configure/).

### `tsconfig.json`

-   To set up some compile(transpile) options for the current project, we would need a `tsconfig.json` under the root directory.
-   We can use `tsc` to auto-generate one:

    -   Firstly, install the native compiler of TypeScript `tsc` by:

        ```bash
        yarn add -D typescript
        ```

    -   Secondly, add related script in `package.json`:

        ```json
        {
            ...,
            "scripts": {
                "tsc": "tsc",
                ...
            },
            ...
        }
        ```

    -   Finally, run `yarn tsc --init` to generate the `tsconfig.json` file automatically.

-   As the course required, these compiler options should be written to `tsconfig.json`:

    ```json
    {
        "compilerOptions": {
            "target": "ES6",
            "outDir": "./build/",
            "module": "commonjs",
            "strict": true,
            "noUnusedLocals": true,
            "noUnusedParameters": true,
            "noImplicitReturns": true,
            "noFallthroughCasesInSwitch": true,
            "esModuleInterop": true
        }
    }
    ```

-   For the auto-generated config file, we could just search for the related options and uncomment them.

## CORS Issue

-   Discussed before in [part3.b](https://fullstackopen.com/en/part3/deploying_app_to_internet#same-origin-policy-and-cors)
-   If we want the resources returned by the backend to be accessed by a different origin or domain, we should install `cors` package and 'use' it in `express`:

    ```bash
    yarn add -D cors @type/cors
    ```

    ```ts
    import express from "express";
    import cors = require("cors");

    const app: Express = express();
    app.use(cors()); // this line is requried
    ```

## Parse JSON HTTP Request Body

-   To accept data transmitted from the frontend in the http request body, we need to 'use' the json module of `express`:

    ```ts
    import express from "express";

    const app: Express = express();
    app.use(json()); // this line is requried
    ```

## Node and JSON Modules

-   There's a way to import JSON files as a module and assign 'type' to the content imported.
-   To do this, we will need to enable the `resolveJsonModule` option in `tsconfig.json`:

    ```json
    "resolveJsonModule": true /* Enable importing .json files. */,
    ```

-   By then, we can import data from JSON file just as we import it from a ts/js file:

    ```ts
    import diaries from "../../data/entries";

    import { DiaryEntry } from "../types";
    ```

-   But be noted of a problem that may arise when using the tsconfig `resolveJsonModule` option:

    -   By default, node will try to resolve modules in order of extensions:

        ```plaintext
        ["js", "json", "node"]
        ```

    -   By enabling `resolveJsonModule`, the list of possible node module extensions is extended to:

        ```plaintext
        ["js", "json", "node", "ts", "tsx"]
        ```

    -   Then, consider a flat folder structure containing files:

        ```plaintext
          ├── myModule.json
          └── myModule.ts
        ```

    -   In TypeScript, with the resolveJsonModule option set to true, the file myModule.json becomes a valid node module. Now, imagine a scenario where we wish to take the file myModule.ts into use:

        ```ts
        import myModule from "./myModule";
        ```

    -   We notice that the .json file extension takes precedence over .ts(from the extended node module extensions). So, myModule.json will be imported and not myModule.ts.

-   To avoid time-eating bugs, it is recommended that within a flat directory, each file with a valid node module extension has a **unique** filename.

## Path Aliases(Mappings)

-   We can setup path aliases in `tsconfig.json` file to avoid related pathes import or long import statement. For example:

    ```json
    "baseUrl": "./", /* Specify the base directory to resolve non-relative module names. */
    "paths": {
        "@/*": ["src/*"],
        "@data/*": ["data/*"],
        "@utils/*": ["src/utils/*"],
        "@services/*": ["src/services/*"],
        "@routers/*": ["src/routers/*"]
    } /* Specify a set of entries that re-map imports to additional lookup locations. */,
    ```

### `tsc` & `ts-node` Ignoring Path Aliases

-   `ts-node` and `ts-node-dev` will ignore the path aliases declared in `tsconfig.json`.

#### Dev

-   My preferred workaround for `ts-node-dev` is to use the `tsconfig-paths` package:

    -   install `tsconfig-paths`:

        ```bash
        yarn add -D tsconfig-paths
        ```

    -   then modify the command in `package.json`:

        ```json
        "scripts": {
           ...,
           "dev": "tsnd --respawn -r tsconfig-paths/register index.ts",
           ...
        }
        ```

#### Build

-   When it comes to the build stage, one more package needs to be added: `@ef-carbon/tspm`. For example:

    -   install `tsconfig-paths` and `@ef-carbon/tspm`: ...
    -   then modify the command in `package.json`:

        ```json
        ...,
        "build": "tsc && ef-tspm",
        ...
        ```

---

-   See [Stack Overflow](https://stackoverflow.com/questions/60067281/typescript-path-aliases-not-resolved-correctly-at-runtime), [GitHub](https://github.com/wclr/ts-node-dev/issues/95#issuecomment-743435649), and also the github page of the two related libraries:

    -   [tsconfig-paths](https://github.com/dividab/tsconfig-paths)
    -   [tspm](https://github.com/ef-carbon/tspm)

## `yarn create`

-   I want to run this `npm` command with `yarn`:

    ```bash
    npm create vite@latest my-app-name -- --template react-ts
    ```

-   The identified commands for `yarn` are:

    ```bash
    yarn global add create-vite
    create-vite --template react-ts
    ```

## To Be Continued...
