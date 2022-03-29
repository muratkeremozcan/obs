Assume you are in a large organization with many services and applications. You are cranking out Cypress ui-e2e and api-e2e tests. Suddenly you realize lots of code is being duplicated between the teams for common tasks; setup, teardown, authentication, or anything pertaining to your domain.

One solution is to be in a monorepo. Another is to have the tests in a repo of its own. While monorepo can be done right - _I have not been in any team that has succeeded with it_ - the latter is an anti-pattern you want to stay away from for many reasons. Leaving that discussion aside, we will assume you are a wise engineer, you want individual repositories and you want the tests living near the source code.

You know how to create Cypress commands, and you wish you knew how to publish these internally as a Cypress plugin, so that other teams do not duplicate any test code and easily abstract away the complexities. But, you are a large org and may be using Typescript - _which makes it all more difficult_ - and you may be using many other Cypress plugins in your Cypress commands, because plugins are great.

You have a situation of plugins within plugins, plans within plans, problems within problems.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uk3vgq90aw09o55bp8jp.png)

The example used is a hypothetical test suite for an authentication service. The tests are api tests. You can find the repo at https://github.com/muratkeremozcan/cypress-test-plugin-example and use it as a template for your future needs. It is also a good example for setting up industry best practices like eslint, prettier, husky, Renovate, badges etc.

> At [Extend](https://www.extend.com/) we have created multiple internal test plugins following these patterns.

- [Create & configure the repo](#create--configure-the-repo)
- [Setup badges](#setup-badges)
  - [Renovate badges:](#renovate-badges)
  - [Badges for your package.json dependencies](#badges-for-your-packagejson-dependencies)
  - [Badge for the repo itself](#badge-for-the-repo-itself)
  - [Badge for cypress dashboard (only works with public dashboards)](#badge-for-cypress-dashboard-only-works-with-public-dashboards)
- [Command Resource Refactoring](#command-resource-refactoring)
  - [`cypress/support/index.js`](#cypresssupportindexjs)
- [Typescript settings](#typescript-settings)
  - [`./src/tsconfig.json`](#srctsconfigjson)
  - [`./src/index.ts`](#srcindexts)
  - [./cypress/tsconfig.json](#cypresstsconfigjson)
  - [./tsconfig.json](#tsconfigjson)
- [Release](#release)
  - [Build the app (dist folder)](#build-the-app-dist-folder)
  - [Update the package version at `./package.json`](#update-the-package-version-at-packagejson)
  - [Create a `./package.json` copy in dist folder, also copy the types and dependency types to the dist folder](#create-a-packagejson-copy-in-dist-folder-also-copy-the-types-and-dependency-types-to-the-dist-folder)
  - [Publish](#publish)
  - [Create a `./package.json` script out of these commands](#create-a-packagejson-script-out-of-these-commands)
  - [Run the script](#run-the-script)
- [Install the plugin at another repo](#install-the-plugin-at-another-repo)
  - [References](#references)

## Create & configure the repo

> You can replicate most of these settings from the sample repo.

`npm init -y`

Set these `package.json` properties:

```json
"main": "src",
"types": "src",
"files": [ "**/*.js", "index.d.ts" ],
```

Add scripts, dependencies, devDependencies.

Setup generic repo configuration: eslintrc.js, .nvmrc, prettier, husky, .vscode/settings.json, renovate.json.

[Rename master to main](https://www.git-tower.com/learn/git/faq/git-rename-master-to-main):

```bash
git branch -m master main
# at github/.../settings/branches, switch default branch to main
git push -u origin main
```

Install & open Cypress, implement the commands, tests,` cypress.json`,` cypress.env.json`, `plugins/index.js` like usual. Start with the [default typescript settings](https://docs.cypress.io/guides/tooling/typescript-support#Configure-tsconfig-json). Add secrets to github if using `cypress.env.json`.

## Setup badges

### Renovate badges:

Add anywhere on your readme, usually the bottom.

```markdown
[renovate-badge]: https://img.shields.io/badge/renovate-app-blue.svg
[renovate-app]: https://renovateapp.com/
```

### Badges for your package.json dependencies

> `npx -p` just installs it on the fly\_

`npx -p dependency-version-badge update-badge <name-of-package> <other-package>`

### Badge for the repo itself

You can use Github to create a badge: Actions > Workflows > click on a workflow > upper right ... > Create Status badge. This will make a badge out of the status of a yml file. Add the query param `?branch=main&event=push` to it to make it only report on main branch's status

```markdown
[![yml-file-name](url-to-worfklow-yml-file/badge.svg?branch=main&event=push)](url-to-worfklow-yml-file)

[![cypress-test-plugin-example](https://github.com/muratkeremozcan/cypress-test-plugin-example/actions/workflows/cypress-test-plugin-example.yml/badge.svg?branch=main&event=push)](https://github.com/muratkeremozcan/cypress-test-plugin-example/actions/workflows/cypress-test-plugin-example.yml)
```

### Badge for cypress dashboard (only works with public dashboards)

```
[![any-name-usually-your-package-dashboard](https://img.shields.io/endpoint?url=https://dashboard.cypress.io/badge/simple/<your-project-id>/main&style=flat&logo=cypress)](https://dashboard.cypress.io/projects/<your-project-id>/runs)
```

## Command Resource Refactoring

If all is working well at this point, Cypress commands have been implemented, you have specs, the repo is setup to your liking, badges are good, push the changes up then proceed.

At the repo root, create a `src` folder. Move `cypress/support/` folder contents to it. Also move the type definitions at `cypress/index.d.ts` to the root of the src folder.

### `cypress/support/index.js`

Create this file with a one-liner import. This will import the top level package folder. We previously set `package.json` "main" to "src" for this reason.

```
import '../..'
```

## Typescript settings

### `./src/tsconfig.json`

Create a `src/tsconfig.json`.
Include all the plugins types you are using in the command files, and include the folder itself with "./"

```json
"compilerOptions": {
  "types": ["@bahmutov/cy-api", "./"]
},
"include": ["**/*.ts"],
"extends": "../tsconfig.json"
```

### `./src/index.ts`

Rename the `index.js` file to `index.ts`. Import all the plugins here.

Convert all the Cypress commands to functions, export the ones that will be used as commands.

Import them at `index.ts` , and wrap them in Cypress commands.

Not only this gives us cleaner looking functions, but also makes our api more obvious.

```typescript
import "@bahmutov/cy-api/support";
import "cypress-data-session";

import { getToken, me, createUser } from "./commands/generic";
import { maybeGetToken } from "./commands/data-session";

Cypress.Commands.add("getToken", getToken);
Cypress.Commands.add("me", me);
Cypress.Commands.add("createUser", createUser);
Cypress.Commands.add("maybeGetToken", maybeGetToken);
```

### `./cypress/tsconfig.json`

The spec files will use Cypress types, and the types from the `./src`. We might also need additional types for independent usage of the plugins in the specs, for example `cypress-data-session` is used to clear the data session in the specs so we need the types here.

```json
  "compilerOptions": {
    "types": ["cypress", "cypress-data-session", "../src/"]
  },
  "include": ["**/*.ts"],
  "extends": "../tsconfig.json"
```

### `./tsconfig.json`

Update `tsconfig.json` at the project root to include the types from `./src`.

In order to run typecheck, you may need to have a subset of the types at the root tsconfig.json. Copy the types from one of the other 2 jsons and follow a process of elimination.

```json
"types": [
  "cypress",
  "node",
  "@bahmutov/cy-api",
  "cypress-data-session",
  "./src"
]
```

## Release

### Build the app (dist folder)

We need to complie the TS into JS.

```bash
tsc --skipLibCheck
```

### Update the package version at `./package.json`

```bash
yarn version
```

### Create a `./package.json` copy in dist folder, also copy the types and dependency types to the dist folder

For this we will write a script at `./scripts/release.js`

```javascript
const fs = require("fs-extra");
const path = require("path");
const packageJson = require("../package.json");

const newPackage = {
  ...packageJson,
};

// whether dist/package.json exists or not, get it created and synced with root package.json
fs.outputFileSync(
  path.resolve(__dirname, "..", "dist", "package.json"),
  JSON.stringify(newPackage, null, 2)
);

// copy the types and dependency types
fs.copy(
  path.resolve(__dirname, "..", "src", "index.d.ts"),
  path.resolve(__dirname, "..", "dist", "index.d.ts")
);

// if your types need additional files, copy them too
fs.copy(
  path.resolve(__dirname, "..", "src", "api-version.ts"),
  path.resolve(__dirname, "..", "dist", "api-version.ts")
);
```

### Publish

Publish step is different per environment, because of internal registry needs.

Internally we use AWS CodeArtifact, and all it takes to release is `yarn publish dist`. For publishing to npm, take a look at [Gleb's instructions](https://glebbahmutov.com/blog/publishing-cypress-command/).

### Create a `./package.json` script out of these commands

```json
"build": "tsc --skipLibCheck",
"release": "yarn build && yarn version && node scripts/release.js && yarn publish dist"
```

### Run the script

```bash
# builds the app (dist folder)
# updates package version in the root package.json
# creates a package.json copy at dist folder
# copies the types and dependency types to the dist folder
# publishes to AWS CodeArtifact
yarn release
```

> This plugin has not been released anywhere at the time of writing. The first person who releases to a registry gets to do it there for the first time!

## Install the plugin at another repo

```bash
yarn add -D cypress-test-plugin-example
# or
npm i -D cypress-test-plugin-example
```

Setup the types at `cypress/tsconfig.json` :

```json
{
  "compilerOptions": {
    "types": ["cypress", "cypress-test-plugin-example"],
    "target": "esnext",
    "lib": ["esnext", "dom"],
    "allowJs": true,
    "resolveJsonModule": true
  },
  "include": ["**/*.ts"]
}
```

Import the package at `cypress/support/index.js` :

```typescript
import cypress-test-plugin-example
```

Try out the commands! You can copy the specs from this repo's only spec and see how they work. The commands you have implemented in your plugin should work just the same way.

### References

https://glebbahmutov.com/blog/publishing-cypress-command/ - Gleb Bahmutov

[https://github.com/bahmutov/cypress-get-by-label JS](https://glebbahmutov.com/blog/publishing-cypress-command/) JS - Gleb Bahmutov

https://github.com/dmtrKovalenko/cypress-real-events TS - Dmitry Kovalenko