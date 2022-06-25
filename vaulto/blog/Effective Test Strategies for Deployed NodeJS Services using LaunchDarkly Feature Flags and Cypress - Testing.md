This is part two of a multi-part series. In the previous post we setup the flags, now we will test them. Before diving into testing feature flags, we will setup Cypress and transfer over the final CRUD e2e spec from the repo [cypress-crud-api-test](https://github.com/muratkeremozcan/cypress-crud-api-test). That repo was featured in the blog post [CRUD API testing a deployed service with Cypress](https://dev.to/muratkeremozcan/crud-api-testing-a-deployed-service-with-cypress-using-cy-api-spok-cypress-data-session-cypress-each-4mlg). Note that the said repo and this service used to be separated - that is a known anti-pattern - and now we are combining the two in a whole. The change will provide us with the ability to use the LaunchDarkly (LD) client instance to make flag value assertions. We would not have that capability if the test code was in a separate repo than the source code, unless the common code was moved to a package & was imported to the two repos. In the real world if we had to apply that as a solution, we would want to have valuable trade-offs.

> From our experience, one example of a valid trade-off for having the tests in a different repo than the source code is when the same test code is shared by multiple source code repositories, thereby eliminating the need to duplicate the test suite.

The branch prior to this work can be checked out at `before-cypress-setup`, and the PR for cypress setup can be found [here](https://github.com/muratkeremozcan/pizza-api/pull/6/files). If you are following along, a practical way to accomplish this section is to copy over the PR.

The branch saga through the blog series looks like the below:

1.  `before-feature-flags`
    
2.  `ld-ff-setup-test` : where we fully setup the node SDK for our lambda and showed it working via rest client.
    
3.  `before-cypress-setup`
    
4.  `cypress-setup`: the branch for this section of the guide; [PR](https://github.com/muratkeremozcan/pizza-api/pull/6/files).
    
5.  `after-cypress-setup`: if you want to skip this section, you can start from this branch
    
6.  `ld-ff-ld-e2e`: the branch the blog will be worked on
    

If you do not want to copy the PR but set up Cypress and move over the code yourself, you can follow along.

In the terminal run `npx @bahmutov/cly init` to scaffold Cypress into the repo. We add the Cypress plugins `npm i -D @bahmutov/cy-api cy-spok cypress-data-session cypress-each jsonwebtoken @withshepherd/faker`.

We copy over the files to mirrored locations, and covert the TS to JS. A painless alternative is to look at the PR and copy over the changes.

-   `cypress/support/index.ts`
    
-   `cypress/support/commands.ts`
    
-   `cypress/integration/with-spok.spec.ts`
    
-   `cypress/plugins/index.js`
    
-   `scripts/cypress-token.js`
    
-   `cypress.json`
    

To ensure all is in working order, we do another deploy with `npm run update`. We start and execute the tests with `npm run cypress:open`, we verify CloudWatch for the logs regarding the flag value, since PUT is a part of the CRUD operation in the e2e test.

Here is the high level overview of the blog post:

-   [Controlling FF with `cypress-ld-control` plugin](#controlling-ff-with-cypress-ld-control-plugin)
    
    -   [Plugin setup](#plugin-setup)
        
    -   [`cypress-ld-control`plugin in action](#cypress-ld-controlplugin-in-action)
        
    -   [Using enums for flag values](#using-enums-for-flag-values)
        
    -   [`setFlagVariation` enables a stateless approach](#setflagvariation-enables-a-stateless-approach)
        
-   [Reading FF state using the test plugin vs the LD client instance](#reading-ff-state-using-the-test-plugin-vs-the-ld-client-instance)
    
-   [Test strategies](#test-strategies)
    
    -   [Conditional execution: get flag state, run conditionally](#conditional-execution-get-flag-state-run-conditionally)
        
        -   [Wrap the test code inside the it block with a conditional](#wrap-the-test-code-inside-the-it-block-with-a-conditional)
            
        -   [Disable / Enable a describe/context/it block or the entire test](#disable--enable-a-describecontextit-block-or-the-entire-test)
            
    -   [Controlled flag: set the flag and run the test](#controlled-flag-set-the-flag-and-run-the-test)
        
-   [Summary](#summary)
    
-   [References](#references)
    

## Controlling FF with `cypress-ld-control` plugin

My friend Gleb Bahmutov authored an [excellent blog](https://glebbahmutov.com/blog/cypress-and-launchdarkly/) on testing LD with Cypress, there he revealed his new plugin [cypress-ld-control](https://github.com/bahmutov/cypress-ld-control). We used it in [Effective Test Strategies for Front-end Applications using LaunchDarkly Feature Flags and Cypress. Part2: testing](https://dev.to/muratkeremozcan/effective-test-strategies-for-testing-front-end-applications-using-launchdarkly-feature-flags-and-cypress-part2-testing-2c72). The distinction here is using the plugin for a deployed service and the consequential test strategies.

### Plugin setup

`npm i -D cypress-ld-control` to add the plugin.

Getting ready for this section, previously we took note of the LD auth token, installed `dotenv` and saved environment variables in the `.env` file. Here is how the `.env` file should look with your SDK key and auth token:

LAUNCHDARKLY_SDK_KEY=sdk-***  
LAUNCH_DARKLY_PROJECT_KEY=pizza-api-example  
LAUNCH_DARKLY_AUTH_TOKEN=api-***

The [cypress-ld-control](https://github.com/bahmutov/cypress-ld-control) plugin utilizes [cy.task](https://docs.cypress.io/api/commands/task), which allows Node code to execute within Cypress context. We are using the `.env` file and declaring the auth token below, but we will also show a way to map `.env` file to `cypress.env.json` & vice versa.

In the real world we have many environments. Each environment has its unique `LAUNCHDARKLY_SDK_KEY`, but the `LAUNCH_DARKLY_AUTH_TOKEN` and `LAUNCH_DARKLY_PROJECT_KEY` are uniform throughout. We recommend having project key and auth token in the `.env` file, and the sdk key in a cypress config file. This setup would let us interrogate the flag state in any deployment. Our repo only uses `Test` environment. To keep things simple, we will only use the `.env` file and leave comments where things would vary in the real world.

> References for the ideas: [Set All Cypress Env Values Using A Single GitHub Actions Secret](https://glebbahmutov.com/blog/secrets-to-env/), [https://twitter.com/howtocode_io/status/1520137904047144960](https://twitter.com/howtocode_io/status/1520137904047144960)

// cypress/plugins/index.js  
​  
/// <reference types="cypress" />  
​  
const cyDataSession = require("cypress-data-session/src/plugin");  
const token = require("../../scripts/cypress-token");  
// cypress-ld-control setup  
const { initLaunchDarklyApiTasks } = require("cypress-ld-control");  
require("dotenv").config();  
​  
module.exports = (on, config) => {  
  const combinedTasks = {  
    // add your other Cypress tasks if any  
    token: () => token,  
    log(x) {  
      // prints into the terminal's console  
      console.log(x);  
      return null;  
    },  
  };  
    
  // if you have many environments, grab the env var from cypress/config/<env>.json file,   
  // since the key changes per deployment  
  // process.env.LAUNCHDARKLY_SDK_KEY = config.env.LAUNCHDARKLY_SDK_KEY  
  // as a side note, you can map .env file to cypress.env with a reverse assignment  
  // the only requirement there would be to wrap the .env values in double quotes  
  // config.env.LAUNCHDARKLY_SDK_KEY = process.env.LAUNCHDARKLY_SDK_KEY   
​  
  // if no env vars, don't load the plugin  
  if (  
    process.env.LAUNCH_DARKLY_PROJECT_KEY &&  
    process.env.LAUNCH_DARKLY_AUTH_TOKEN  
  ) {  
    const ldApiTasks = initLaunchDarklyApiTasks({  
      projectKey: process.env.LAUNCH_DARKLY_PROJECT_KEY,  
      authToken: process.env.LAUNCH_DARKLY_AUTH_TOKEN,  
      environment: "test", // the name of your environment to use  
    });  
    // copy all LaunchDarkly methods as individual tasks  
    Object.assign(combinedTasks, ldApiTasks);  
    // set an environment variable for specs to use  
    // to check if the LaunchDarkly can be controlled  
    config.env.launchDarklyApiAvailable = true;  
  } else {  
    console.log("Skipping cypress-ld-control plugin");  
  }  
​  
  // register all tasks with Cypress  
  on("task", combinedTasks);  
​  
  return Object.assign(  
    {},  
    config, // make sure to return the updated config object  
    // add any other plugins here  
    cyDataSession(on, config)  
  );  
};

> See other plugin file examples [here](https://github.com/bahmutov/cypress-ld-control/blob/main/cypress/plugins/index.js.) and [here](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/blob/main/cypress/plugins/index.js).
> 
> Check out the [5 mechanics around cy.task and the plugin file](https://www.youtube.com/watch?v=2HdPreqZhgk&t=279s).

We can quickly setup the CI and include LD project key, LD auth token and LD SDK key as environment variables. We need the first two for `cypress-ld-control`, and we need the SDK key for being able to use the LD client instance in the tests.

# .github/workflows/main.yml  
​  
name: cypress-crud-api-test  
on:  
  push:  
  workflow_dispatch:  
​  
# if this branch is pushed back to back, cancel the older branch's workflow  
concurrency:  
  group: ${{ github.ref }} && ${{ github.workflow }}  
  cancel-in-progress: true  
​  
jobs:  
  test:  
    strategy:  
      # uses 1 CI machine  
      matrix:  
        machines: [1]  
    runs-on: ubuntu-20.04  
    steps:  
      - name: Checkout 🛎  
        uses: actions/checkout@v2  
​  
      # https://github.com/cypress-io/github-action  
      - name: Run api tests 🧪  
        uses: cypress-io/github-action@v3.0.2  
        with:  
          browser: chrome  
          record: true  
          group: crud api test  
        env:  
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}  
          LAUNCH_DARKLY_PROJECT_KEY: ${{ secrets.LAUNCH_DARKLY_PROJECT_KEY }}  
          LAUNCH_DARKLY_AUTH_TOKEN: ${{ secrets.LAUNCH_DARKLY_AUTH_TOKEN }}  
          LAUNCHDARKLY_SDK_KEY: ${{ secrets.LAUNCHDARKLY_SDK_KEY }} #{{  
        
   # Here we are running the unit tests after the e2e  
   # taking advantage of npm install in Cypress GHA.  
   # Ideally we install first, and carry over the cache  
   # to unit and e2e jobs.  
   # Check this link for the better way:  
   # https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/blob/main/.github/workflows/main.yml  
      - name: run unit tests  
        run: npm run test

We can quickly setup Cypress Dashboard, and create the project:

![Cypress Dashboard project setup](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3kioueq7aw8nmwar0wup.png)

Grab the projectId (gets copied to `cypress.json`) and the record key (gets copied to Github secrets).

![Project id and record key](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1lq333pb06kymrvfwcax.png)

Configure the GitHub repo secrets at Settings > Actions > Action Secrets.

![GHA secrets](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qu8k9vvg99s1lpp3fvao.png)

Because of eventual consistency, when testing lambdas we prefer to increase the default command timeout from 4 to 10 seconds. We also add retries for good measure. Here is how `cypress.json` looks:

{  
  "projectId": "4q6j7j",  
  "baseUrl": "https://2afo7guwib.execute-api.us-east-1.amazonaws.com/latest",  
  "viewportWidth": 1000,  
  "retries": {  
    "runMode": 2,  
    "openMode": 0  
  },  
  "defaultCommandTimeout": 10000  
}

> Cypress has 3 levels of retries; function level, api level, and CI level. It is not only a great tool for UI e2e testing handling [network requests in a web app](https://docs.cypress.io/guides/guides/network-requests#Testing-Strategies), but it is also a great api testing framework thanks to [cy.request](https://docs.cypress.io/api/commands/request#Syntax). Check out the post [API e2e testing event driven systems](https://dev.to/muratkeremozcan/api-testing-event-driven-systems-7fe) for a better perspective.

### [`cypress-ld-control`](https://github.com/bahmutov/cypress-ld-control)plugin in action

[The plugin API](https://github.com/bahmutov/cypress-ld-control#api) provides these functions:

-   getFeatureFlags
    
-   getFeatureFlag
    
-   setFeatureFlagForUser
    
-   removeUserTarget
    
-   removeTarget (works like a deleteAll version of the previous)
    

The idempotent calls are safe anywhere:

// cypress/integration/feature-flags/ff-sanity.spec.js  
​  
it("get flags", () => {  
  // get one flag  
  cy.task("cypress-ld-control:getFeatureFlag", "update-order").then(  
    console.log  
  );  
    
  // get all flags (in an array)  
  cy.task("cypress-ld-control:getFeatureFlags").then(console.log);  
});

The sanity test confirms the flag configuration we have at the LD interface.

![FF sanity](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sf4dzxkd6osn8n4tahfx.png)

We like making helper functions out of the frequently used plugin commands. In Cypress, `cy.task` cannot be used inside a command, but it is perfectly fine in a utility function. We add some logging to make the test runner easier to reason about. You can reuse these utilities anywhere.

// cypress/support/ff-helper.js  
​  
import { datatype, name } from "@withshepherd/faker";  
​  
/** Used for stateless testing in our examples.  
It may not be needed other projects */  
export const randomUserId = `FF_${name  
  .firstName()  
  .toLowerCase()}${datatype.number()}`;  
​  
/**  
 * Gets a feature flag by name  
 * @param featureFlagKey this is usually a kebab-case string, or an enum representation of it */  
export const getFeatureFlag = (featureFlagKey) =>  
  cy.log(`**getFeatureFlag** flag: ${featureFlagKey}`)  
    .task("cypress-ld-control:getFeatureFlag", featureFlagKey);  
​  
/** Gets all feature flags */  
export const getFeatureFlags = () =>  
  cy.log("**getFeatureFlags**").task("cypress-ld-control:getFeatureFlags");  
​  
/**  
 * Sets a feature flag variation for a user.  
 * @param featureFlagKey this is usually a kebab-case string, or an enum representation of it  
 * @param userId LD user id, for anonymous users it is randomly set  
 * @param variationIndex index of the flag; 0 and 1 for boolean, can be more for string, number or Json flag variants */  
export const setFlagVariation = (featureFlagKey, userId, variationIndex) =>  
  cy.log(`**setFlagVariation** flag: ${featureFlagKey} user: ${userId} variation: ${variationIndex}`)  
    .task("cypress-ld-control:setFeatureFlagForUser", {  
      featureFlagKey,  
      userId,  
      variationIndex,  
    });  
​  
/**  
 * Removes feature flag for a user.  
 * @param featureFlagKey this is usually a kebab-case string, or an enum representation of it  
 * @param userId LD user id, for anonymous users it is randomly set */  
export const removeUserTarget = (featureFlagKey, userId) =>  
  cy.log(`**removeUserTarget** flag: ${featureFlagKey} user: ${userId}`)  
    .task("cypress-ld-control:removeUserTarget", {  
      featureFlagKey,  
      userId,  
    });  
​  
/**  
 * Can be used like a deleteAll in case we have multiple users being targeted  
 * @param featureFlagKey  
 * @param targetIndex */  
export const removeTarget = (featureFlagKey, targetIndex = 0) =>  
  cy.log(`**removeTarget** flag: ${featureFlagKey} targetIndex:${targetIndex}`)  
    .task("cypress-ld-control:removeTarget", {  
      featureFlagKey,  
      targetIndex,  
    });

We can use the helper functions from now onwards. While verifying the data we can even do deeper assertions with `cy-spok`.

// cypress/integration/feature-flags/ff-sanity.spec.js  
​  
import { getFeatureFlags, getFeatureFlag } from "../support/ff-helper";  
import spok from "cy-spok";  
​  
describe("FF sanity", () => {  
  it("should get flags", () => {  
    getFeatureFlag("update-order").its("key").should("eq", "update-order");  
​  
    getFeatureFlags().its("items.0.key").should("eq", "update-order");  
  });  
​  
  it("should get flag variations", () => {  
    getFeatureFlag("update-order")  
      .its("variations")  
      .should((variations) => {  
        expect(variations[0].value).to.eq(true);  
        expect(variations[1].value).to.eq(false);  
      });  
  });  
​  
 it('should make deeper assertions with spok', () => {  
    getFeatureFlag("update-order")  
      .its("variations")  
      .should(  
        spok([  
          {  
            description: "PUT endpoint available",  
            value: true,  
          },  
          {  
            description: "PUT endpoint is not available",  
            value: false,  
          },  
        ])  
      );  
 })  
});

Spok is great for mirroring the data into concise, comprehensive and flexible assertions. Here the data is just an array of objects.

![spok data](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ci6vmvtw1q9qiamaz28n.png)

### Using enums for flag values

We are using the the string `update-order` often. In the previous blog where the LD feature flag was setup, we even used it at the lambda `./handlers/update-order.js`. When there are so many flags in our code base, it is possible to use an incorrect string. It would be great if we had a central location of flags, we imported those enums and could only get the flag name wrong in one spot.

There are a few benefits of using enums and having a variable convention to hold their values:

-   We have a high level view of all our flags since they are at a central location.
    
-   We cannot get them wrong while using the flags in lambdas or tests; string vs enum.
    
-   In any file it would be clear which flags are relevant.
    
-   It would be easy to search for the flags and where they are used, which makes maintenance seamless.
    

In JS `Object.freeze` can be used to replicate TS' enum behavior. Now is also a good time to move the `get-ld-flag-value.js` from `./handlers` into `./flag-utils`, it will make life easier when using the utility for test assertions. Here is the refactor:

// ./flag-utils/flags.js  
​  
const FLAGS = Object.freeze({  
  UPDATE_ORDER: 'update-order'  
})  
module.exports = {  
  FLAGS  
};  
​  
​  
// At the spec file import the constant & replace the string arg  
// ./cypress/integration/feature-flags/ff-sanity.spec.js  
import { FLAGS } from "../../flag-utils/flags";  
​  
it("should get flags", () => {  
  getFeatureFlag(FLAGS.UPDATE_ORDER)  
  // ...  
​  
​  
// At the handler file, do the same  
// ./handlers/update-order.js  
const getLDFlagValue = require("../ff-utils/get-ld-flag-value");  
const { FLAGS } = require("../flag-utils/flags");  
​  
async function updateOrder(orderId, options) {  
  const FF_UPDATE_ORDER = await getLDFlagValue(FLAGS.UPDATE_ORDER);  
  //...

After the refactor, we can quickly deploy the code with `npm run update` and run the run the tests with `npm run cy:run`. Having API e2e tests for lambda functions gives us confidence about code and deployment quality.

### `setFlagVariation` enables a stateless approach

At first it may not be obvious from `cypress-ld-control` [api docs](https://github.com/bahmutov/cypress-ld-control#api) , but `setFeatureFlagForUser` takes a `userId` argument and **creates that userId if it does not exist**. If we use an arbitrary string, that key will appear on the LD Targeting tab. In case we are not using randomized users, emails or other randomized entities in our tests, we can utilize a function for generating random flag user ids. We can prefix that with `FF_` so that if there is any clean up needed later in flag management, those specific users can be cleared easily from the LD interface.

// ./cypress/support/ff-helper.js  
import { datatype, name } from "@withshepherd/faker";  
​  
export const randomUserId = `FF_${name  
  .firstName()  
  .toLowerCase()}${datatype.number()}`;

// cypress/integration/feature-flags/ff-sanity.spec.js  
​  
it.only("should set the flag for a random user", () => {  
  setFlagVariation(FLAGS.UPDATE_ORDER, randomUserId, 0);  
});

Setting the flag by the user, we can view the flag being set to this targeted individual. It would be trivial to randomize a user per test and target them. How can we prove that all other users still get served one value, while the targeted user gets served another?

## Reading FF state using the test plugin vs the LD client instance

Recall our flag utility at `./flag-utils/get-ld-flag-value` which we also use in the lambda handler. At a high level it gets the flag value using the LD client, and makes abstractions under the hood:

1.  Initializes the LD client & waits for the initialization to complete.*
    

2.  Gets the flag value using the LD client.*
    

3.  If a user is not provided while getting the flag value, populates an anonymous user generic users.*
    

4.  The code calling the LD client cannot be observed by any other part of the application.*
    

That is a very useful bit of code, and the part we need for test assertions is **how it can get the flag value for a targeted user, versus all other users**. We can run any Node code within Cypress context via `cy.task`. Let's import `getLDFlagValue` to our plugins file at`cypress/plugins/index.js` and add it as a Cypress task.

Our original `getLDFlagValue` function took three arguments (_key_, _user_, _defaultValue_). There is a key bit of knowledge needed to convert it to a task.

-   When `cy.task` calls a function without any arguments, life is easy; `cy.task('functionName')`.
    
-   When `cy.task` calls a function with a single argument things are simple; `cy.task('functionName', arg)`.
    
-   When there are multiple arguments, we have to wrap them in an object; `cy.task('functionName', { arg1, arg2 })`
    

On the LD side LD client accepts a user object as `{ key: 'userId' }`. We have to do some wrangling to make the api easy to use. We want:

-   `cy.task('getLDFlagValue', 'my-flag-value' )` to get the flag value for generic users on any environment.
    
-   `cy.task('getLdFlagValue', { key: 'my-flag-value', userId: 'abc123' })` to get the flag value for a targeted user in any environment.
    

// ./cypress/plugins/index.js  
​  
const getLDFlagValue = require("../flag-utils/get-ld-flag-value");  
// ... other imports  
​  
function isObject(value) {  
  const type = typeof value;  
  return value != null && (type === "object" || type === "function");  
}  
​  
module.exports = (on, config) => {  
  const combinedTasks = {  
    // add your other Cypress tasks if any  
    token: () => token,  
    log(x) {  
      // prints into the terminal's console  
      console.log(x);  
      return null;  
    },  
    getLDFlagValue: (arg) => {  
      // cy.task api changes whether there is 1 arg or multiple args;  
      // it takes a string for a single arg, it takes and object for multiple args.  
      // LD client accepts a user object as { key: 'userId' }.  
      // We have to do some wrangling to make the api easy to use  
      // we want an api like :   
      // cy.task('getLDFlagValue', 'my-flag-value' ) for generic users  
      // cy.task('getLdFlagValue', { key: 'my-flag-value', userId: 'abc123'  }) for targeted users   
      if (isObject(arg)) {  
        const { key, userId } = arg  
        console.log(`cy.task args: key: ${key} user.key: ${userId}`)  
        return getLDFlagValue(key, { key: userId })  
      }  
      console.log(`cy.task arg: ${arg}`)  
      return getLDFlagValue(arg)  
    }  
  };  
​  
    
  // ... the rest of the file

We will use the LD client instance to confirm the flag state for a targeted user vs generic users. Let's check out the the task in a basic test.

// ./cypress/integration/feature-flags/ff-sanity.spec.js  
​  
it.only("should get a different flag value for a specified user", () => {  
  setFlagVariation(FLAGS.UPDATE_ORDER, "foo", 1);  
​  
  cy.log("**getLDFlagValue(key)** : gets value for any other user");  
  cy.task("getLDFlagValue", FLAGS.UPDATE_ORDER).then(cy.log);  
​  
  cy.log("**getLDFlagValue(key, user)** : just gets the value for that user");  
  cy.task("getLDFlagValue", { key: FLAGS.UPDATE_ORDER, user: "foo" }).then(  
    cy.log  
  );  
});

**KEY:** Executing that code, we realize the enabler for stateless feature flag testing. We prove that the flag can be set for a targeted user, that value can be read by our `getLDFlagValue` lambda utility using the LD client, which can either focus on the targeted user or any other generic user while reading the flag value. **That ability can fully de-couple feature flag testing from feature flag management**.

> In contrast, in a front-end application the key enabler for stateless testing was targeting anonymous users and the user's id - `ld:$anonUserId` - getting created by LD in local storage.

![Stateless test 1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fl8oln5kug2hx279j4r9.png)

`cypress-ld-control` plugin allows us to set a flag for a targeted user. If it allowed changing the flag value for everyone, mutating a shared state for every flag reader would not be ideal. On the other hand, the plugin can only be used to get the flag value for generic users vs a targeted user. _(If Gleb disagrees or adds support for it later, we stand corrected)_. Reading the flag value for a targeted user wasn't a need when feature flag testing a UI application; while using anonymous users LD would set local storage with `ld:$anonUserId` enabling a unique browser instance which would we would make UI assertions against. Consequently, `getLDFlagValue` lambda utility using the LD client instance is also needed for user-targeted test assertions when statelessly testing feature flags in deployed services.

Here is the high level summary of our feature flag testing tool set:

`cypress-ld-control` test plugin:

-   Our primary tool to set a feature flag: `setFlagVariation('my-flag', 'user123', 1)`
    
-   Our primary tool to clean up feature flags: `removeUserTarget('my-flag', 'user123')`
    
-   Can read the flag value for generic users: `getFeatureFlag('my-flag'`)
    

`getLDFlagValue` LD client instance:

-   Our primary Feature Flag development enabler, used to read flag state.
    
-   In tests, it can read the flag value for generic users: `cy.task('getLDFlagValue', 'my-flag')`
    
-   In tests, it can read the flag value for a targeted user: `cy.task('getLDFlagValue', { key: 'my-flag', user: 'user123' })`
    

Let's prove out the theory and show a harmonious usage of these utilities in a succinct test.

context("flag toggle using the test plugin", () => {  
    const TRUE_VARIANT = 0; // generic users get this  
    const FALSE_VARIANT = 1; // targeted users get this  
​  
    afterEach("user-targeted-flag clean up", () =>  
      removeUserTarget(FLAGS.UPDATE_ORDER, randomUserId)  
    );  
​  
    it("should get the flag value for generic users using Cypress test plugin", () => {  
      getFeatureFlag(FLAGS.UPDATE_ORDER)  
        .its("environments.test.fallthrough.variation")  
        .should("eq", TRUE_VARIANT);  
    });  
​  
    it("should get the flag value for generic users using the LD instance", () => {  
      cy.task("getLDFlagValue", FLAGS.UPDATE_ORDER).should("eq", true);  
    });  
​  
    it("should get the flag value TRUE using the LD instance", () => {  
      setFlagVariation(FLAGS.UPDATE_ORDER, randomUserId, TRUE_VARIANT);  
​  
      cy.task("getLDFlagValue", {  
        key: FLAGS.UPDATE_ORDER,  
        userId: randomUserId,  
      }).should("eq", true);  
        
      // in the real world we can have real tests here   
      // testing the feature per flag state  
    });  
​  
    it("should get the flag value FALSE using the LD instance", () => {  
      setFlagVariation(FLAGS.UPDATE_ORDER, randomUserId, FALSE_VARIANT);  
​  
      cy.task("getLDFlagValue", {  
        key: FLAGS.UPDATE_ORDER,  
        userId: randomUserId,  
      }).should("eq", false);  
        
      // in the real world we can have real tests here   
      // testing the feature per flag state  
    });  
  });

It is important to toggle the flag to each state and verify it, because if the LD instance cannot get the flag value, it will return a default `false` per our setup.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/eaedl3bwubhnqze59mhc.png)

We can confirm our `cy.task` vs LD client instance data in each test.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sma5s9qdyg43ef6ng0ej.png)

## Test strategies

Now that we have stateless feature flag setting & removing capabilities coupled with feature flag value reading - which is an idempotent operation - how can we use them in e2e tests? In the blog post [Effective Test Strategies for Front-end Applications using LaunchDarkly Feature Flags and Cypress. Part2: testing](https://dev.to/muratkeremozcan/effective-test-strategies-for-testing-front-end-applications-using-launchdarkly-feature-flags-and-cypress-part2-testing-2c72) there were effectively two strategies; stub the network & test vs control the flag & test. With an API client we can do the latter the same way. There is no stubbing the network however, what other approach can we have?

### Conditional execution: get flag state, run conditionally

Although conditional testing is usually an anti-pattern, when testing feature flags in a deployed service it gives us a read-only, idempotent approach worth exploring. After all we have to have some maintenance-free, non-feature flag related tests that need to work in every deployment regardless of flag states. Let's focus on our CRUD e2e test for the API `cypress/integration/with-spok.spec.js` where we have the flagged Update feature.

#### Wrap the test code inside the it block with a conditional

We can wrap the relevant part of the test with a conditional driven by the flag value:

// here we can also use the getFeatureFlag plugin function  
cy.task("getLDFlagValue", FLAGS.UPDATE_ORDER).then((flagValue) => {  
  if (flagValue) {  
    cy.updateOrder(token, orderId, putPayload)  
      .its("body")  
      .should(satisfyAssertions);  
  } else {  
    cy.log('**the flag is disabled, so the update will not be done**');  
  }  
});

With this tweak, our specs which are not flag relevant will work on any deployment regardless of flag status.

![Flag condition inside it block](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dayff6sc0blkei9ok3vg.png)

![Flag enabled inside it block](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fdldvh900tmrgajwssas.png)

#### Disable / Enable a describe/context/it block or the entire test

We can take advantage of another one of Gleb's fantastic plugins [cypress-skip-test](https://github.com/cypress-io/cypress-skip-test/blob/master/README.md). `npm install -D @cypress/skip-test` and Add the below line to `cypress/support/index.js:`

require('@cypress/skip-test/support')

It has a key feature which allows us to run Cypress commands before deciding to skip or continue. We can utilize it in a describe / context / it block, but if we want to disable the whole suite without running anything, inside the before block is the way to go.

  before(() => {  
    cy.task("token").then((t) => (token = t));  
    cy.task("getLDFlagValue", FLAGS.UPDATE_ORDER).then((flagValue) =>  
      cy.onlyOn(flagValue === true)  
    );  
  });

Toggle the flag on, and things work as normal:

![onlyOn flag === true](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1ayzet39z9u0l2zoxbr5.png)

If the flag is off, the test is skipped.

![onlyOn flag === false](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/71cowpmvmpykuyjoa5iq.png)

Here is the entire spec:

/// <reference types="cypress"/>  
// @ts-nocheck  
​  
import spok from "cy-spok";  
import { datatype, address } from "@withshepherd/faker";  
import { FLAGS } from "../../flag-utils/flags";  
​  
describe("Crud operations with cy spok", () => {  
  let token;  
  before(() => {  
    cy.task("token").then((t) => (token = t));  
    // we can control the the entire test,   
    // a describe / context / it block with cy.onlyOn or cy.skipOn  
    // Note that it is redundant to have the 2 variants of flag-conditionals in the same test  
    // they are both enabled here for easier blog readbility  
    cy.task("getLDFlagValue", FLAGS.UPDATE_ORDER).then((flagValue) =>  
      cy.onlyOn(flagValue === true)  
    );  
  });  
​  
  const pizzaId = datatype.number();  
  const editedPizzaId = +pizzaId;  
  const postPayload = { pizza: pizzaId, address: address.streetAddress() };  
  const putPayload = {  
    pizza: editedPizzaId,  
    address: address.streetAddress(),  
  };  
​  
  // the common properties between the assertions  
  const commonProperties = {  
    address: spok.string,  
    orderId: spok.test(/\w{8}-\w{4}-\w{4}-\w{4}-\w{12}/), // regex pattern to match any id  
    status: (s) => expect(s).to.be.oneOf(["pending", "delivered"]),  
  };  
​  
  // common spok assertions between put and get  
  const satisfyAssertions = spok({  
    pizza: editedPizzaId,  
    ...commonProperties,  
  });  
​  
  it("cruds an order, uses spok assertions", () => {  
    cy.task("log", "HELLO!");  
​  
    cy.createOrder(token, postPayload).its("status").should("eq", 201);  
​  
    cy.getOrders(token)  
      .should((res) => expect(res.status).to.eq(200))  
      .its("body")  
      .then((orders) => {  
        const ourPizza = Cypress._.filter(  
          orders,  
          (order) => order.pizza === pizzaId  
        );  
        cy.wrap(ourPizza.length).should("eq", 1);  
        const orderId = ourPizza[0].orderId;  
​  
        cy.getOrder(token, orderId)  
          .its("body")  
          .should(  
            spok({  
              pizza: pizzaId,  
              ...commonProperties,  
            })  
          );  
​  
        cy.log(  
          "**wrap the relevant functionality in the flag value, only run if the flag is enabled**"  
        );  
        cy.task("getLDFlagValue", FLAGS.UPDATE_ORDER).then((flagValue) => {  
          if (flagValue) {  
            cy.log("**the flag is enabled, updating now**");  
            cy.updateOrder(token, orderId, putPayload)  
              .its("body")  
              .should(satisfyAssertions);  
          } else {  
            cy.log("**the flag is disabled, so the update will not be done**");  
          }  
        });  
​  
        cy.getOrder(token, orderId).its("body").should(satisfyAssertions);  
​  
        cy.deleteOrder(token, orderId).its("status").should("eq", 200);  
      });  
  });  
});  
​

### Controlled flag: set the flag and run the test

We also want to gain confidence that no matter how flags are controlled in any environment, they will work with our service. This will enable us to fully de-couple the testing of feature flags from the management of feature flags, thereby de-coupling continuous deployment from continuous delivery. The key here is to be able control and verify the flag state for a scoped user.

Similar to the UI approach, we can set the feature flag in the beginning of a test and clean up at the end. This would be an exclusive feature flag test which we only need to run on one deployment; if we can control and verify the flag value's consequences on one deployment, things will work the same on any deployment. Later, the spec would be converted to a permanent one, where which we can tweak it to not need flag controls, or the spec can get removed entirely. Therefore it is a good practice to house the spec under `./cypress/integration/feature-flags` and control in which deployment it executes with config files using `ignoreTestFiles` property in the JSON.

In our example demoing this test would require a token and user scope; create a pizza for a scoped user and try to update the pizza as that user. Since we did not implement authorization to our lambda, this test is not able to be shown in a satisfactory manner. We can set the flag for a user but since the update is not scoped to that user, verifying whether that user can update a pizza or not is not possible. We are confident that the test scenario will be trivial in the real world where APIs are secured and tokens are scoped to users.

## Summary

We covered how to utilize `cypress-ld-control` to set and remove flags for targeted users, how to take advantage of the LD client instance in Cypress tests to read the flag value for targeted users, and how these capabilities enable two main test strategies: conditional execution and controlled flag. Similar to the front-end flavor of testing feature flags with Cypress, we have showed a way fully de-couple stateless feature flag testing from feature flag control.

We are opinionated that the presented feature flag configuration and test strategies for a deployed service are an ideal approach that can be applied universally. The source code has been shared, please let us know your thoughts and help in improving the approach.

## References

-   [https://glebbahmutov.com/blog/cypress-and-launchdarkly/](https://glebbahmutov.com/blog/cypress-and-launchdarkly/)