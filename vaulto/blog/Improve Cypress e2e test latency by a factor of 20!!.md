It has been almost 2 years since my friend Gleb Bahmutov published the blog [Fast Cypress spec bundling using ESBuild](https://glebbahmutov.com/blog/fast-esbuild/) and the npm module [cypress-esbuild-preprocessor](https://github.com/bahmutov/cypress-esbuild-preprocessor). He reported incredible gains in test latency when using esbuild as opposed to the Cypress built-in preprocessor that uses Webpack under the hood. In layman terms, test latency is the time it takes to bundle & start a test, the time we see **"Your Tests Are Starting..."** before the execution.

Looking at [sourcegraph.com](https://sourcegraph.com/) or [GitHub code search](https://cs.github.com/) for a search string `from '@bahmutov/cypress-esbuild-preprocessor'`, at the time of writing we only find a handful of open source repos taking advantage of esbuild with Cypress. We thought we should spread the good word and report on the results at scale about the cost savings in engineering and CI feedback time.

Any blog post is lackluster without working code, so here is a [PR from scratch](https://github.com/muratkeremozcan/cypress-crud-api-test/pull/229/files) adding esbuild to a repository with Cypress. You can find the final code on the main branch of [the repository we will use in this example](https://github.com/muratkeremozcan/cypress-crud-api-test). Other examples can be found at [tour-of-heroes-react-cypress-ts](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts) as well as a [VueJS app](https://github.com/muratkeremozcan/appsyncmasterclass-frontend). The framework and the bundler the framework uses are irrelevant, any repo can take advantage of [cypress-esbuild-preprocessor](https://github.com/bahmutov/cypress-esbuild-preprocessor) for e2e tests.

> Check out the cross linked Youtube video at [https://www.youtube.com/watch?v=diB6-jikHvk](https://www.youtube.com/watch?v=diB6-jikHvk)

> Esbuild preprocessor only applies to Cypress e2e tests. Our component tests use the same bundler our framework is using. We are still looking into practical custom bundler recipes, maybe your app uses webpack but you want to use vite in the component tests, and will update this post if there are any possible improvements to component test latency.

-   [TL, DR;](#tl-dr)
    
-   [Long version](#long-version)
    
    -   [Prerequisite: optimize cypress config for plugins, tasks, commands, e2e, ct](#prerequisite-optimize-cypress-config-for-plugins-tasks-commands-e2e-ct)
        
    -   [Step 1: Add the packages](#step-1-add-the-packages)
        
    -   [Step 2: Isolate the preprocessor task in its own file & import into the config file](#step-2-isolate-the-preprocessor-task-in-its-own-file--import-into-the-config-file)
        
    -   [Step 3: If there are any compile related issues, wrap them `cy.task()`](#step-3-if-there-are-any-compile-related-issues-wrap-them-cytask)
        
-   [Local feedback duration](#local-feedback-duration)
    
-   [CI feedback duration and cost savings](#ci-feedback-duration-and-cost-savings)
    
-   [Wrap up](#wrap-up)
    

## TL, DR;

-   Add the packages
    

yarn add -D @bahmutov/cypress-esbuild-preprocessor esbuild @esbuild-plugins/node-globals-polyfill @esbuild-plugins/node-modules-polyfill

> Esbuild polyfills may not be necessary in simpler repos, but if they are necessary, in the absence of them you will get cryptic errors. You can toggle these later to see if they are needed

-   Isolate the preprocessor task in its own file: [TS example](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/cypress/support/esbuild-preprocessor.ts), [JS example](https://github.com/muratkeremozcan/appsyncmasterclass-frontend/blob/main/cypress/support/esbuild-preprocessor.js). Import it into the config file(s): [TS example](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/cypress.config.ts#L22), [JS example](https://github.com/muratkeremozcan/appsyncmasterclass-frontend/blob/main/cypress/config/local.config.js#L20).
    

> This will make it seamless to toggle the esbuild preprocessor at any point in the future, making it easy to isolate webpack vs esbuild compile issues, and to opt out of the workaround later if Cypress makes esbuild the default e2e bundler.

-   If there are any compile related issues, possibly from polyfills not having support for packages intended for Node.js usage, such as [fs or crypto](https://github.com/remorses/esbuild-plugins/blob/d9f6601a24dc4e0470046eda8c772e6523c52b96/node-modules-polyfill/src/polyfills.ts#L142), wrap them in [cy.task](https://docs.cypress.io/api/commands/task#docusaurus_skipToContent_fallback) so that they can be executed in Cypress/browser context.
    

> [Here is an externally reproduced blocker and the workaround to it with `cy.task`](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/pull/219/files#diff-f18e5de0df19ff702ed389b87fbaab9ecfa62544389e805f73ad5ffa8d2764e0R12), about `jsonwebtoken` and crypto. `jwt.sign` from `jsonwebtoken` causes [a compile issue](https://github.com/cypress-io/cypress/issues/25533#issuecomment-1434520618), therefore we wrap it in `cy.task`. We will go through another example below so you can see the error and exercise with `cy.task` to solve it.

## Long version

### Optional prerequisite: optimize cypress config for plugins, tasks, commands, e2e, ct

This optimization will help speed up our test warmup time at scale, further simplify our plugin and task configurations. The process is described elaborately in [this video](https://www.youtube.com/watch?v=9d7zDR3eyB8), [the PR](https://github.com/muratkeremozcan/react-cypress-ts-vite-template/pull/55), and the final code is shown in two simple template examples; [CRA-repo](https://github.com/muratkeremozcan/react-cypress-ts-template), [Vite-repo](https://github.com/muratkeremozcan/react-cypress-ts-vite-template). This is the way we wish Cypress came out of the box.

Here are the main takeaways:

-   `support/commands.ts`, `e2e.ts`, `component.ts`/`tsx` must exist, or they will get created on `cypress open`.
    
    -   `e2e.ts` runs before e2e tests
        
    -   `component.ts` runs before component tests.
        
    -   `commands.ts` is imported in `e2e.ts` and `component.ts` files, therefore it runs before any kind of test.
        
    -   Put commands applicable to both e2e and CT in `commands.ts`.
        
    -   Put e2e-only commands in `e2e.ts`, ct-only commands in `component.ts/tsx`.
        
-   Prefer to import plugins at spec files as opposed to importing them in one of the above files. Only import in the above 3 files if they must be included in every test. I.e. if the plugin must apply to all e2e, import it at `e2e.ts`, if it must apply to all CT, import it at `component.ts`. If it must be everywhere, import it in `commands.ts`.
    
-   Some plugins also have to be included under `setupNodeEvents` function in the `cypress.config` file(s) , for example [cypress-data-session](https://github.com/bahmutov/cypress-data-session#v10) needs this to use the [shareAcrossSpecs](https://github.com/bahmutov/cypress-data-session#shareacrossspecs) option. Isolate all such plugins in one file; [TS example](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/cypress/support/plugins.ts).
    
-   Similar to the previous bullet point, tasks also can be isolated under one file as in this [example](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/cypress/support/tasks.ts#L8). We can enable the tasks with a [one-liner](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/cypress.config.ts#L23) which is particularly useful when we have multiple config files, for example when we have a config per deployment.
    
-   Large imports impact bundle time negatively; for example prefer named imports vs global imports, prefer `en` locale for `fakerJs` vs every locale if you only need English.
    

### Step 1: Add the packages

We will be using a sample repo with only Cypress; [cypress-crud-api-test](https://github.com/muratkeremozcan/cypress-crud-api-test) that has the above prerequisite fulfilled. This just makes esbuild preprocessor easier to bring in to the project, but it is not a requirement.

Clone the repo `https://github.com/muratkeremozcan/cypress-crud-api-test` and check out the branch `before-esbuild` to start from scratch. You can find the [final PR here](https://github.com/muratkeremozcan/cypress-crud-api-test/pull/229/files).

`yarn add -D @bahmutov/cypress-esbuild-preprocessor esbuild @esbuild-plugins/node-globals-polyfill @esbuild-plugins/node-modules-polyfill`

### Step 2: Isolate the preprocessor task in its own file & import into the config file

Copy this code to `cypress/support/esbuild-preprocessor.ts`

// ./cypress/support/esbuild-preprocessor.ts  
​  
import { NodeGlobalsPolyfillPlugin } from "@esbuild-plugins/node-globals-polyfill";  
import { NodeModulesPolyfillPlugin } from "@esbuild-plugins/node-modules-polyfill";  
const createBundler = require('@bahmutov/cypress-esbuild-preprocessor');  
​  
export default function tasks(on: Cypress.PluginEvents) {  
  on(  
    "file:preprocessor",  
    createBundler({  
      plugins: [  
        NodeModulesPolyfillPlugin(),  
        NodeGlobalsPolyfillPlugin({  
          process: true,  
          buffer: true,  
        }),  
      ],  
    })  
  );  
}

Import the task at the config file.

> We can comment out the line any time to opt out of esbuild.

// ./cypress.config.ts  
​  
import { defineConfig } from "cypress";  
import plugins from "./cypress/support/plugins";  
import tasks from "./cypress/support/tasks";  
import esbuildPreprocessor from "./cypress/support/esbuild-preprocessor"; // new  
​  
export default defineConfig({  
  viewportHeight: 1280,  
  viewportWidth: 1280,  
  projectId: "4q6j7j",  
​  
  e2e: {  
    setupNodeEvents(on, config) {  
      esbuildPreprocessor(on); // new  
      tasks(on);  
      return plugins(on, config);  
    },  
    baseUrl: "https://2afo7guwib.execute-api.us-east-1.amazonaws.com/latest",  
  },  
});

### Step 3: If there are any compile related issues, wrap them `cy.task()`

At this point we are done, because in this repo we do not have any compile issues. We are already wrapping the Node.js native package [`jsonwebtoken`](https://github.com/muratkeremozcan/cypress-crud-api-test/blob/main/scripts/cypress-token.ts) in [`cy.task`](https://github.com/muratkeremozcan/cypress-crud-api-test/blob/main/cypress/support/tasks.ts).

Let's suppose we were not doing that and reproduce a compile issue you may run into.

Create a test file `compile-error.cy.ts`

// ./cypress/e2e/compile-error.cy.ts  
​  
import jwt from "jsonwebtoken"; // use version 8.5.1  
​  
// The jwt.sign method expects the payload as the first argument,  
// the secret key as the second argument,  
// options (such as expiration time) as the third argument  
const newToken = () =>  
  jwt.sign(  
    {  
      email: "c",  
      firstName: "b",  
      lastName: "c",  
      accountId: "123",  
      scope: "orders:order:create orders:order:delete orders:order:update",  
    },  
    "TEST",  
    {  
      expiresIn: "10m",  
      subject: "123",  
    }  
  );  
​  
it("fails", () => {  
  console.log(newToken());  
});

Execute the test and we get a cryptic compile error

![compile-error](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wr6dosve74eue1u99bde.png)

Revert back to the webpack preprocessor by disabling the `eslintPreprocessor` :

import { defineConfig } from "cypress";  
import plugins from "./cypress/support/plugins";  
import tasks from "./cypress/support/tasks";  
// import esbuildPreprocessor from './cypress/support/esbuild-preprocessor'  
​  
export default defineConfig({  
  viewportHeight: 1280,  
  viewportWidth: 1280,  
  projectId: "4q6j7j",  
​  
  e2e: {  
    setupNodeEvents(on, config) {  
      // esbuildPreprocessor(on) // DISABLED  
      tasks(on);  
      return plugins(on, config);  
    },  
    baseUrl: "https://2afo7guwib.execute-api.us-east-1.amazonaws.com/latest",  
  },  
});

We see that the test takes a few seconds to start(!) but it compiles. We can even see the encrypted token value in the console.

Enable back the esbuildPreprocessor, and let's work around the issue by wrapping the NodeJs native code in `cy.task`.

Create a new file `cypress/support/newToken.ts`:

// ./cypress/support/newToken.ts  
​  
import jwt from "jsonwebtoken";  
​  
const newToken = () =>  
  jwt.sign(  
    {  
      email: "c",  
      firstName: "b",  
      lastName: "c",  
      accountId: "123",  
      scope: "orders:order:create orders:order:delete orders:order:update",  
    },  
    "TEST",  
    {  
      expiresIn: "10m",  
      subject: "123",  
    }  
  );  
export default newToken;

Add the task to `cypress/support/tasks.ts`:

import log from "./log";  
import newToken from "./newToken"; // the new task  
import * as token from "../../scripts/cypress-token";  
​  
export default function tasks(on: Cypress.PluginEvents) {  
  on("task", { log });  
​  
  on("task", token);  
​  
  on("task", { newToken }); // the new task  
}

Use cy.task in the test, and we are green.

// ./cypress/e2e/compile-error.cy.ts  
it("fails NOT!", () => {  
  cy.task("newToken").then(console.log);  
});

![no-error](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/meauaiobg0qy1vaoo5jl.png)

This approach has worked really well in multiple external as well internal projects at Extend. Let's look at some results at scale.

## Local feedback duration

We demoed some esbuild results on a small project in [this video](https://www.youtube.com/watch?v=Hc_3oLpayOY). At scale, in real world applications, the numbers are even more impressive. Here are some results for local testing with 3 internal applications at Extend. They use lots of plugins and reach over 2 million (it block) executions per year according to Cypress Cloud.

|       | plugin optimization | esbuild-preprocessor | test latency improvement        |  
| ----- | ------------------- | -------------------- | ------------------------------- |  
| App A | none                | yes                  | 20sec -> 2 sec, 10x improvement |  
| App B | yes                 | none                 | 20sec -> 10 sec, 2x improvement |  
| App C | yes                 | yes                  | 20sec -> 1 sec, 20x improvement |

The 8 minute video [Improve Cypress e2e test latency by a factor of 20!!](https://studio.youtube.com/video/diB6-jikHvk/edit) demonstrates the results in action.

Esbuild gave us 10x test latency improvement. The cost was minimal, a factor of [the sample PR here](https://github.com/muratkeremozcan/cypress-crud-api-test/pull/229/files).

Performing the plugin import optimization (described in [this video](https://www.youtube.com/watch?v=9d7zDR3eyB8)) gave us 2x improvement albeit at the cost of 100s, sometimes 1000s of changes in lines of code.

Opinion: if you are starting new or if you do not have too many tests, do both optimizations. If you have many tests, and esbuild optimization is satisfactory then skip the plugin optimization.

## CI feedback duration and cost savings

Mind that Cypress Cloud only reports on test execution duration, which does not include test latency; "Your Tests Are Loading...". We have to look at CI execution results to see the gain. Any improvement on Cypress Cloud reported test duration is a bonus.

The following CI results are only for esbuild preprocessor, in this app we already had test plugins and file imports optimized.

In the before we have 14 parallel machines, each taking around 12.5 minutes:

![CI-before](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x1autgxh3nctpr641mxm.png)

After the esbuild preprocessor improvement, we are saving around 2 minutes per machine which is ~15% improvement in execution time. It also reflects in CI minutes, ~22minutes less in this case.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qnck6amn2mwu1a8bxbab.png)

Here is the before view of the test suite in Cypress Cloud. The duration was 6:09. Mind that the graph looks open on the right side because of component test machines starting later.

![Cy-cloud-before](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uq6a29ggr035dm0d7cf1.png)

Here is the after Cypress Cloud view after esbuild preprocessor improvements. Surprisingly test execution speed also came down to 4:28. This means esbuild also effected the test duration by about 20%. The view is in a different scale because of the component tests being parallelized and finishing faster in this run, but we can notice the reduced gap between the green blocks which are the test spec files. They start faster back to back.

![cy-cloud-after](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sga88eiwfme2fxi4ozpj.png)

We should analyze the results together with Cypress Could engineers, perhaps our assumptions are not entirely accurate, though a conservative estimate would be that per CI run we are saving at least 20% feedback time and cost in CI minutes.

If we look at GitHub workflow runs in the past year in one of the projects, even with very conservative numbers, we can be saving an immense amount of time every year for engineers waiting for their results. Suppose 100k e2e runs happen every year, each saving 2 minutes wait time for the engineer, and ~20ish CI minutes. That's over 100 days of engineering time saved per year, 1000 days of CI minutes.

![workflow-runs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8yoj7ia4g58fxat57v11.png)

## Wrap up

Esbuild preprocessor is easy to implement for current Cypress e2e test suites, giving incredible local and CI time & cost savings.

Plugin and file import tune up is recommended for new projects, or if the cost of refactor is feasible.

We really want Cypress to make esbuild the norm everywhere. Give your thumbs up to the open feature request [https://github.com/cypress-io/cypress/issues/25533](https://github.com/cypress-io/cypress/issues/25533) .

Many thanks to Gleb Bahmutov, Lachlan Miller and all the Cypress engineers making the tool better and more performant.