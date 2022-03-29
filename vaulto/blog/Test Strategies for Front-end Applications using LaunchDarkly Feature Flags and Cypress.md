This is part two of a multi-part series. In the previous post we setup the flags, now we will test them. If you have feature flags already implemented in your app, this post may be more interesting to you. Mind that the framework of choice is inconsequential when testing feature flags; the app used here is React but it could be Angular, Vue, Svelte, Solid or plain JS.

Testing the application, its feature flags, the deployments of the app, combinations of it all may seem intimidating at first. At unit/component test level things are straightforward; stub the FF and test all variants. For e2e, often teams may disable tests on an environment with/without FFs, because it is just a different application at that point. We cannot expect an app to pass the same tests on different deployments if the FF are different.

Thinking about the test strategy at a high level, we can treat e2e testing FFs like the UI login scenario; test the FFs in isolation with due diligence and stub it everywhere else.

- [Stubbing a feature flag](#stubbing-a-feature-flag)
  - [Stub the api calls to the LD events endpoint](#stub-the-api-calls-to-the-ld-events-endpoint)
  - [Stub the push updates from LaunchDarkly (EventSource)](#stub-the-push-updates-from-launchdarkly-eventsource)
  - [Stub our custom FeatureFlags into the app](#stub-our-custom-featureflags-into-the-app)
  - [How to use the stubs](#how-to-use-the-stubs)
- [Controlling FFs with cypress-ld-control plugin](#controlling-ffs-with-cypress-ld-control-plugin)
  - [Plugin setup](#plugin-setup)
  - [Plugin in action](#plugin-in-action)
  - [`getFeatureFlag` & `getFeatureFlags`](#getfeatureflag--getfeatureflags)
  - [Simple boolean flag (`date-and-week`) with `setFeatureFlagForUser` & `removeUserTarget`](#simple-boolean-flag-date-and-week-with-setfeatureflagforuser--removeusertarget)
  - [Boolean flag `slide-show`](#boolean-flag-slide-show)
  - [Json flag `prev-next`](#json-flag-prev-next)
  - [Numeric flag nex-prev](#numeric-flag-nex-prev)
- [Managing FF state with concurrent tests](#managing-ff-state-with-concurrent-tests)
  - [The tests are stateful](#the-tests-are-stateful)
  - [Randomization can help statefulness](#randomization-can-help-statefulness)
  - [Randomizing the LD user key](#randomizing-the-ld-user-key)
  - [Handling multiple `it` blocks](#handling-multiple-it-blocks)
- [Summary](#summary)

## Stubbing a feature flag

[In the repo](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress) let's try out an ui-(component)integration test that focuses on next and previous buttons for Bookables . These features are related to the feature flag `prev-next-bookable`. None of the features are network relevant, therefore all network calls are stubbed. We still get real calls from/to LD though.

```js
// cypress/integration/ui-integration/bookables-list.spec.js`

describe('Bookables', () => {
  before(() => {
    // ui-(component)integration test,
    // the network does not matter for these features
    cy.stubNetwork()
    cy.visit('/bookables')
    cy.url().should('contain', '/bookables')
    cy.get('.bookables-page')
  })

  // note that cy.intercept() needs to be applied
  // independently per it block,
  // as well as on initial load above
  // because we are hitting the network there too
  beforeEach(cy.stubNetwork)
  const defaultIndex = 0

  ...

  // @FF_prevNextBookable
  context('Previous and Next buttons', () => {
    it('should switch to the previous bookable and cycle', () => {
      cy.getByCy('bookables-list').within(() => {
        cy.getByCyLike('list-item').eq(defaultIndex).click()

        cy.getByCy('prev-btn').click()
        cy.checkBtnColor(defaultIndex + 3, 'rgb(23, 63, 95)')
        cy.checkBtnColor(defaultIndex, 'rgb(255, 255, 255)')

        cy.getByCy('prev-btn').click().click().click()
        cy.checkBtnColor(defaultIndex, 'rgb(23, 63, 95)')
      })
    })

    it('should switch to the next bookable and cycle', () => {
      cy.getByCy('bookables-list').within(() => {
        cy.getByCyLike('list-item').eq(defaultIndex).click()

        cy.getByCy('next-btn').click().click().click()
        cy.checkBtnColor(defaultIndex + 3, 'rgb(23, 63, 95)')

        cy.getByCy('next-btn').click()
        cy.checkBtnColor(defaultIndex, 'rgb(23, 63, 95)')
        cy.checkBtnColor(defaultIndex + 1, 'rgb(255, 255, 255)')
      })
    })
  })

  ...
})
```

When running the spec, we immediately notice a few LD calls. Any component with LD FFs will have these.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jldxet16qugakowenhoe.png)

We can use [cy.intercept](https://docs.cypress.io/api/commands/intercept) api to spy or stub any network request or response.

### Stub the api calls to the LD events endpoint

Let's look at the post request going out to the events endpoint. Our app is not doing much with it.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/u9ue5kq7yaqm77990bec.png)

We can stub any post request going out to that url to return an empty response body. The status does not even matter. We use a regex for the url because the usual minify approach with `**/events.launchdarkly` would try to stub out our baseUrl and be inaccurate.

```js
before(() => {
  cy.stubNetwork()
  cy.intercept(
    { method: 'POST', hostname: /.*events.launchdarkly.com/ },
    { body: {} }
  ).as('LDEvents')
  cy.visit()
```

Notice the stubbed post call:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t51widqivqyzeevatbmm.png)

### Stub the push updates from LaunchDarkly (EventSource)

Before tackling the next call, let's talk about `cy.intercept`'s `req.reply()`.

Per [the docs](https://docs.cypress.io/api/commands/intercept#Ending-the-response-with-res-send) you can supply a `StaticResponse` to Cypress in 4 ways:

- `cy.intercept()` with [`an argument`](https://docs.cypress.io/api/commands/intercept#staticResponse-lt-code-gtStaticResponselt-code-gt): to stub a response to a route; `cy.intercept('/url', staticResponse)`
- [`req.reply()`](https://docs.cypress.io/api/commands/intercept#Providing-a-stub-response-with-req-reply): to stub a response from a request handler; `req.reply(staticResponse)`
- [`req.continue()`](https://docs.cypress.io/api/commands/intercept#Controlling-the-outbound-request-with-req-continue): to stub a response from a request handler, while letting the request continue to the destination server; `req.continue(res => {..} )`
- [`res.send()`](https://docs.cypress.io/api/commands/intercept#Ending-the-response-with-res-send): to stub a response from a response handler; `res.send(staticResponse)`

That means we can use `req.reply()` to turn off the push updates from LD, because `req.reply()` lets us access the request handler and stub a response.

```js
// non-LD related network (users, bookables etc.)
cy.stubNetwork();

// we already stubbed LDEvents
cy.intercept(
  { method: "POST", hostname: /.*events.launchdarkly.com/ },
  { body: {} }
).as("LDEvents");

// turn off push updates from LaunchDarkly (EventSource)
cy.intercept(
  { method: "GET", hostname: /.*clientstream.launchdarkly.com/ },
  // access the request handler and stub a response
  (req) =>
    req.reply("data: no streaming feature flag data here\n\n", {
      "content-type": "text/event-stream; charset=utf-8",
    })
).as("LDClientStream");
```

This is how the network looks at this point:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/klg4g6lnowdkyeyv015v.png)

### Stub our custom FeatureFlags into the app

The most interesting network call is the one going out to LD itself. In the response we can see all our FFs.

> Mind that once a flag is defined, we cannot change the key and we can only change the name. Looking at the network response makes us wish we followed a convention; `kebab-case-<component name>` would be a good one.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3jgg17111ym2ndjn9mnq.png)

Let's intercept it and see that response in another form. `req.reply` can be used to intercept the data; here we are intercepting any GET requests to `app.launchdarkly.com` and just logging it out.

```js
cy.intercept({ method: "GET", hostname: /.*app.launchdarkly.com/ }, (req) =>
  req.reply((data) => {
    console.log(data);
  })
);
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ttwgvs7yc5xw3d9i96l6.png)

The interesting part is the body property. Let's de-structure it:

```js
cy.intercept({ method: "GET", hostname: /.*app.launchdarkly.com/ }, (req) =>
  req.reply(({ body }) => {
    console.log(body);
  })
);
```

It is our feature flags, the exact same thing we saw on the browser's Network tab!

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h1nvd43nllpsflm9bssq.png)

All right then, let's over-simplify this. Let's say that the custom feature flag object we want is something like this:

```js
const featureFlags = {
  "prev-next-bookable": { Next: false, Previous: true },
  "slide-show": false,
  // ... the rest of the relative flags, if any...
};
```

If we took `{body}`  - the real network response we just logged out - replaced the keys and values with what we want above, that would be a perfect feature flag stub.

> We can iterate through an object with lodash map (lodash is built into Cypress).

Here is the approach:

- Iterate through our desired FF object `featureFlags`
- Take the real response `body` as a table sample
- Declare our desired `featureFlags` keys into the table: `body[ffKey]`
- Assign our desired `featureFlags` values into the table `body[ffKey] = { value: ffValue }`
- Build our stubbed `body` and return it

```js
cy.intercept({ method: "GET", hostname: /.*app.launchdarkly.com/ }, (req) =>
  req.reply(({ body }) =>
    Cypress._.map(featureFlags, (ffValue, ffKey) => {
      body[ffKey] = { value: ffValue };
      return body;
    })
  )
).as("LDApp");
```

Let's wrap all that in a command which you can copy and use anywhere.

```js
Cypress.Commands.add("stubFeatureFlags", (featureFlags) => {
  // ignore api calls to events endpoint
  cy.intercept(
    { method: "POST", hostname: /.*events.launchdarkly.com/ },
    { body: {} }
  ).as("LDEvents");

  // turn off push updates from LaunchDarkly (EventSource)
  cy.intercept(
    { method: "GET", hostname: /.*clientstream.launchdarkly.com/ },
    // access the request handler and stub a response
    (req) =>
      req.reply("data: no streaming feature flag data here\n\n", {
        "content-type": "text/event-stream; charset=utf-8",
      })
  ).as("LDClientStream");

  /** Stubs the FF with the specification
   * Iterate through our desired FF object `featureFlags`
   * Take the real response `body` as a table sample
   * Declare our desired `featureFlags` keys into the table: `body[ffKey]`
   * Assign our desired `featureFlags` values into the table `body[ffKey] = { value: ffValue }`
   * Build our stubbed `body` and return it
   */
  return cy
    .intercept({ method: "GET", hostname: /.*app.launchdarkly.com/ }, (req) =>
      req.reply(({ body }) =>
        Cypress._.map(featureFlags, (ffValue, ffKey) => {
          body[ffKey] = { value: ffValue };
          return body;
        })
      )
    )
    .as("LDApp");
});
```

Let's try it out in our spec. Toggle the booleans to see it in action

```js
// cypress/integration/ui-integration/bookables-list.spec.js`

describe('Bookables', () => {
  const allStubs = () => {
    cy.stubNetwork()
    return cy.stubFeatureFlags({
      'prev-next-bookable': { Next: true, Previous: true },
      'slide-show': true
    })
  }

  before(() => {
    allStubs()

    cy.visit('/bookables')
    cy.url().should('contain', '/bookables')
    cy.get('.bookables-page')
  })

  beforeEach(allStubs)

  const defaultIndex = 0

  ...

  // @FF_prevNextBookable
  context('Previous and Next buttons', () => {
    it('should switch to the previous bookable and cycle', () => {
      cy.getByCy('bookables-list').within(() => {
        cy.getByCyLike('list-item').eq(defaultIndex).click()

        cy.getByCy('prev-btn').click()
        cy.checkBtnColor(defaultIndex + 3, 'rgb(23, 63, 95)')
        cy.checkBtnColor(defaultIndex, 'rgb(255, 255, 255)')

        cy.getByCy('prev-btn').click().click().click()
        cy.checkBtnColor(defaultIndex, 'rgb(23, 63, 95)')
      })
    })

    it('should switch to the next bookable and cycle', () => {
      cy.getByCy('bookables-list').within(() => {
        cy.getByCyLike('list-item').eq(defaultIndex).click()

        cy.getByCy('next-btn').click().click().click()
        cy.checkBtnColor(defaultIndex + 3, 'rgb(23, 63, 95)')

        cy.getByCy('next-btn').click()
        cy.checkBtnColor(defaultIndex, 'rgb(23, 63, 95)')
        cy.checkBtnColor(defaultIndex + 1, 'rgb(255, 255, 255)')
      })
    })
  })

  ...
})
```

> You might want to disable the assertions because they will fail as you are removing the features.

We toggle `Next` and `Previous` between true and false to display the buttons or not. We also toggle `slide-show` to start the slideshow and display the stop button or not. This way we are able to fully ui test all states of the flags on the page.

<img width="100%" style="width:100%" src="https://media.giphy.com/media/MC1oZh3tHahmi8otpx/giphy.gif">

> The inspiration for the command is from [Tim Kutnick](https://medium.com/@kutnickclose?source=post_page-----897349b7f976-----------------------------------) who authored this [medium post](https://medium.com/@kutnickclose/how-to-use-cypress-with-launchdarkly-897349b7f976).

### How to use the stubs

While playing around with the spec you might have noticed that there are really 8 versions of the app on this page; 2^3 with the 3 booleans. Should we extract the feature flag relevant tests into its own spec and test the varieties? Sounds like a fun, and terrible idea. But, maybe someone has to have this kind of a flag configuration and it can be simplified. Let's theory-craft.

| slide-show | prev-btn | next-btn |
| ---------- | -------- | -------- |
| OFF        | OFF      | OFF      |
| OFF        | OFF      | ON       |
| OFF        | ON       | OFF      |
| OFF        | ON       | ON       |
| ON         | OFF      | OFF      |
| ON         | OFF      | ON       |
| ON         | ON       | OFF      |
| ON         | ON       | ON       |

With this we would be exhaustively e2e testing all feature flags on this Bookings page.

Here is the [combinatorial approach](https://github.com/NoriSte/ui-testing-best-practices/blob/master/sections/advanced/combinatorial-testing.md) to reduce the exhaustive test suite. Paste combinatorial test (CT) model into the web app [CTWedge](https://foselab.unibg.it/ctwedge/):

```
Model FF_Bookings
 Parameters:
   slideShow : Boolean
   prevBtn:  Boolean
   nextBtn : Boolean

Constraints:
  // we do not want to test all 3 flags off
 # ( slideShow=false AND prevBtn=false <=> nextBtn!=false) #
```

And we get the test suite of 4:

| slide-show | prev-btn | next-btn |
| ---------- | -------- | -------- |
| ON | ON | OFF |
| ON | OFF | OFF |
| OFF | ON | OFF |
| OFF | OFF | ON |

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wj5ec8ipe5wwfw1c0luk.png)

Theoretical math and your tax dollars - if you are in the USA - have already proven that [the above suite will find a majority of the bugs](https://cypress.slides.com/cypress-io/siemens-case-study#/16/0/2) that may appear in this scenario. If you need further convincing, you can download the CSV, and upload to [CAMetrics](https://matris.sba-research.org/tools/cametrics/#/metrics/growth/visualization); an online tool to measure and visualize combinatorial coverage.

> *Coverage is an assessment for the thoroughness or completeness of testing with respect to a model. Our model can be unit coverage, feature coverage, mutation score, combinatorial coverage, non-functional requirement coverage, anything!*

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1p9p2ppjdiux1xjpr0uz.png)

If in a time crunch, you could apply risk-based testing and just test the first case plus one more for good measure.

> *Always think, "If A works, what are the chances B can fail?" . If the answer is "Low," then apply **risk-based-testing**. In other words, there is a cost to quality; do not over-test and try to get away with minimal testing that gives enough release confidence.*

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fbnoup96y7jkqhhffi3m.png)

Does this mean we should use combinatorial testing CT and stubbing to cover feature flags? Combinatorial testing may be better suited for the next section, when testing real feature flags that have too many variants. As mentioned before, we treat e2e testing FFs like UI login; test the FFs with due diligence in isolation and stub it everywhere else. Stubbing is better suited for turning on the flags and testing the features in full. It helps us shift left, test the latest and greatest consistently throughout the deployments without disabling the tests in a deployment they may not apply in.

> ***While stubbing feature flags, we want to always test the latest and greatest features while shifting as left as possible; test the latest and greatest version of the feature, and test it as early as you can. Finally, ensure that you can execute the same suite regardless of the deployment.***

We will be testing all the variants of the flags, without stubbing, in the next section, and they all have either 2 or 4 variants. We do not really need combinatorial testing for that, but if there had to be a more complex case, combinatorial testing can be used to reduce it. Keep it as a tool in your testing arsenal.  

Before moving on to controlling FFs, we should turn off all the LD flags and execute the e2e suite. Any tests that fail must have depended on real FFs and we should stub them.

```js
// cypress/integration/ui-integration/bookable-details-retainment.spec.js
describe('Bookable details retainment', () => {
  before(() => {
    // ui-integration tests stub the network
    // ui-e2e does not
    // this stub is irrelevant of feature flags
    cy.stubNetwork()

    // this feature only relies on Next button being available
    cy.stubFeatureFlags({
      'prev-next-bookable': { Next: true }
    })
```

> For Cypress component tests, we already should have [`cy.stub`](https://docs.cypress.io/api/commands/stub)'d the feature flags. The flags work great with a served app, but in a component or unit test, there is no network call to LD; stubbing the hooks is the way to go. Mind that at the time of writing, because of [issue 18552](https://github.com/cypress-io/cypress/issues/18662) stubbing modules isn't working in the component runner. The same thing is ok in the e2e runner. In the sample repo this will be updated with Cypress 10.

## Controlling FFs with cypress-ld-control plugin

My friend Gleb Bahmutov authored an [excellent blog](https://glebbahmutov.com/blog/cypress-and-launchdarkly/) on testing LD with Cypress, there he revealed his new plugin [cypress-ld-control](https://github.com/bahmutov/cypress-ld-control) that abstracts away the complexities with LD flags controls.

### Plugin setup

- `yarn add -D cypress-ld-control`.

- Create an access token at LD, to be used by the tests to access the LD api.

   ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jq80qt72lku9fasnvjaq.png)

- Create the `.env` file, or `.as-a.ini` if you are using Gleb's package

   The [cypress-ld-control](https://github.com/bahmutov/cypress-ld-control) plugin utilizes [cy.task](https://docs.cypress.io/api/commands/task), which allows node code to execute within Cypress context. Therefore we will not be able to use `cypress.env.json` to store these LD related environment variables locally.

   For our use case any method for accessing `process.env` will do. Gleb showed how to use [as-a](https://github.com/bahmutov/as-a) to make things neat. We can show a [dotenv](https://www.npmjs.com/package/dotenv) alternative, less neat but will do for a single repo use case. `yarn add -D dotenv` and create a gitignored `.env` file in the root of your project. The idea is exactly the same as `cypress.env.json` file; add values here for local use, gitignore and store them securely in CI.

   Per convention, we can create a `.env.example` file in the root, and that should communicate to repo users that they need an `.env` file with real values in place of wildcards. Populate the project key and the auth token in the `.env` file .

   ```.env
   LAUNCH_DARKLY_PROJECT_KEY=hooks-in-action
   LAUNCH_DARKLY_AUTH_TOKEN=api-********-****-****-****-************
   ```

   > We get the project key from Projects tab.
   >
   > ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5s63kuh3fjq949n7z3mh.png)

- Setup the plugins file.

   ```js
   // cypress/plugins/index.js

   // only needed if using dotenv package
   require("dotenv").config();
   // any other imports
   const reactScripts = require("@cypress/react/plugins/react-scripts");
   const cyGrep = require("cypress-grep/src/plugin");
   const codeCoverageTask = require("@cypress/code-coverage/task");
   // cypress-ld-control setup
   const { initLaunchDarklyApiTasks } = require("cypress-ld-control");

   module.exports = (on, config) => {
     // any other needed code (ex: CRA component test setup)
     const injectDevServer =
       config.testingType === "component" ? reactScripts : () => ({});

     const combinedTasks = {
       // add your other Cypress tasks if any
     };

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

     // register all tasks with Cypress
     on("task", combinedTasks);

     return Object.assign(
       {},
       config, // make sure to return the updated config object
       codeCoverageTask(on, config),
       injectDevServer(on, config),
       cyGrep
     );
   };
   ```

   > See another example at <https://github.com/bahmutov/cypress-ld-control/blob/main/cypress/plugins/index.js>.
   >
   > Check out the [5 mechanics around cy.task and the plugin file](https://www.youtube.com/watch?v=2HdPreqZhgk&t=279s).

- If running tests in the CI, set the secrets at the CI provider interface and inject the secrets to the yml setup.

   ```yml
   // .github/workflows/main.yml
   
   ...
   
   - name: Cypress e2e tests 🧪
    uses: cypress-io/github-action@v3.0.2
     with:
       install: false # a needed job installed already...
       start: yarn dev # concurrently starts ui and api servers
       wait-on: 'http://localhost:3000'
       browser: chrome
   env:
     CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
     LAUNCH_DARKLY_PROJECT_KEY: ${{ secrets.LAUNCH_DARKLY_PROJECT_KEY }}
     LAUNCH_DARKLY_AUTH_TOKEN: ${{ secrets.LAUNCH_DARKLY_AUTH_TOKEN }}
   ```

   ![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3vgc23i3bip8fvujzqo3.png)

### Plugin in action

We are opinionated that feature flag tests should be isolated in their own folder, this will help with selective testing logic further down the line when considering flags and deployments.

```bash
## cypress/integration

├── integration
    ├── feature-flags
    │   └── ff-sanity.spec.js
    ├── ui-e2e
    │   └── crud-bookable.spec.js
    └── ui-integration
        ├── bookable-details-retainment.spec.js
        ├── bookables-list.spec.js
        ├── direct-nav.spec.js
        ├── routes.spec.js
        └── user-context-retainment.spec.js
```

> Check out [The 32+ ways of selective testing with Cypress](https://dev.to/muratkeremozcan/the-32-ways-of-selective-testing-with-cypress-a-unified-concise-approach-to-selective-testing-in-ci-and-local-machines-1c19).

[The plugin API](https://github.com/bahmutov/cypress-ld-control#api) provides these functions:

- getFeatureFlags
- getFeatureFlag
- setFeatureFlagForUser
- removeUserTarget
- removeTarget (works like a deleteAll version of the previous)

### `getFeatureFlag` & `getFeatureFlags`

The idempotent calls should be safe anywhere:

```js
// cypress/integration/feature-flags/ff-sanity.spec.js

it("get flags", () => {
  // get one flag
  cy.task("cypress-ld-control:getFeatureFlag", "prev-next-bookable").then(
    console.log
  );
  // get all flags (in an array)
  cy.task("cypress-ld-control:getFeatureFlags").then(console.log);
});
```

The setup and the plugin api work great. Even this much enables a potential UI app test strategy where we just read and assert the flag states in isolation in a spec like this one, and test the app features via stubbed flags in other specs. Since all calls are idempotent, there would not be any clashes between the specs or the entities executing them.

Let's write a test confirming that all our feature flags are being loaded into the app, while showcasing a little bit of the Cypress api.

```js
// cypress/integration/feature-flags/ff-sanity.spec.js

it("should get all flags", () => {
  cy.task("cypress-ld-control:getFeatureFlags")
    .its("items")
    .as("flags")
    .should("have.length", 4);

  // we can get the data once above, and alias it
  // then we can refer to it with with @
  cy.get("@flags").its(0).its("key").should("eq", "date-and-week");
  cy.get("@flags").its(1).its("key").should("eq", "next-prev");
  cy.get("@flags").its(2).its("key").should("eq", "slide-show");
  cy.get("@flags").its(3).its("key").should("eq", "prev-next-bookable");

  // or we could refactor the above block of 4 lines like below
  const flags = [
    "date-and-week",
    "next-prev",
    "slide-show",
    "prev-next-bookable",
  ];

  cy.wrap(flags).each((value, index) =>
    cy.get("@flags").its(index).its("key").should("eq", value)
  );
});
```

The most concise version would be as such:

```js
// cypress/integration/feature-flags/ff-sanity.spec.js

it("should get all flags", () => {
  const flags = [
    "date-and-week",
    "next-prev",
    "slide-show",
    "prev-next-bookable",
  ];

  cy.task("cypress-ld-control:getFeatureFlags")
    .its("items")
    .should("have.length", 4)
    .each((value, index, items) =>
      cy.wrap(items[index]).its("key").should("eq", flags[index])
    );
});
```

Note that the most recently added flag is the highest index, and on the LD interface the most recently added flag is on the top by default. It can be sorted by Oldest if that makes things more comfortable.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0tgnnx34u6n4fl1es7pe.png)

### Simple boolean flag (`date-and-week`) with `setFeatureFlagForUser` & `removeUserTarget`

Before setting one, let's try to get a simple flag. `date-and-week` toggles the beginning and the end of the week for a given date. Recall [Use a boolean variant FF in a component](#use-a-boolean-variant-ff-in-a-component) from the previous post in the series.

```js
// cypress/integration/feature-flags/bookings-date-and-week.spec.js

context("Bookings Date and Week", () => {
  before(() => {
    // make sure the page fully loads first
    cy.intercept("GET", "**/bookings*").as("getBookings*");
    cy.visit("/bookings");
    cy.wait("@getBookings*");
  });

  it("should toggle date-and-week", () => {
    cy.task("cypress-ld-control:getFeatureFlag", "slide-show")
      .its("variations")
      // log it out to get a feel
      .then((variations) => {
        Cypress._.map(variations, (variation, i) =>
          cy.log(`${i}: ${variation.value}`)
        );
      })
      .should("have.length", 2)
      // and is an alias for should, should + expect will retry
      // so would then + cy.wrap or its()
      .and((variations) => {
        expect(variations[0].value).to.eq(true);
        expect(variations[1].value).to.eq(false);
      });
  });
});
```

So far, so good.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/g072bxig0ng3vzxqr4hh.png)

The [API for `setFeatureFlagForUser`](https://github.com/bahmutov/cypress-ld-control#setfeatureflagforuser) requires that *the feature flag must have "Targeting: on" for user-level targeting to work.* Recall [Connect the app with LD section](#connect-the-app-with-ld) from the previous post of the series. We added a user at that time, and now it can be useful.

```js
// src/index.js
  ...

  const LDProvider = await asyncWithLDProvider({
    clientSideID: '62346a0d87293a1355565b20',
    // we do not want the React SDK to change flag keys to camel case
    // https://docs.launchdarkly.com/sdk/client-side/react/react-web#flag-keys
    reactOptions: {
      useCamelCaseFlagKeys: false
    },
    // https://docs.launchdarkly.com/sdk/client-side/react/react-web#configuring-the-react-sdk
    user: {
      key: 'aa0ceb',
      name: 'Grace Hopper',
      email: 'gracehopper@example.com'
    }
  })

  ...
```

Let's utilize the user key to test out `setFeatureFlagForUser`

```js
// cypress/integration/feature-flags/bookings-date-and-week.spec.js

it("should toggle date-and-week", () => {
  const featureFlagKey = "date-and-week";
  const userId = "aa0ceb";

  cy.task("cypress-ld-control:getFeatureFlag", featureFlagKey)
    .its("variations")
    .then((variations) => {
      Cypress._.map(variations, (variation, i) =>
        cy.log(`${i}: ${variation.value}`)
      );
    })
    .should("have.length", 2)
    .and((variations) => {
      expect(variations[0].value).to.eq(true);
      expect(variations[1].value).to.eq(false);
    });

  cy.log("**variation 0: True**");
  cy.task("cypress-ld-control:setFeatureFlagForUser", {
    featureFlagKey,
    userId,
    variationIndex: 0,
  });

  cy.getByCy("week-interval").should("be.visible");

  cy.log("**variation 1: False**");
  cy.task("cypress-ld-control:setFeatureFlagForUser", {
    featureFlagKey,
    userId,
    variationIndex: 1,
  });

  cy.getByCy("week-interval").should("not.exist");

  // no clean up!?
});
```

<img width="100%" style="width:100%" src="https://media.giphy.com/media/DHR6kSfdZQNu4eLXzO/giphy.gif">

The test works pretty well, but there is a concern at the LD interface; after execution we left the flag there for this user.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bms82fbmbhd243du4ngx.png)

We should end the test with a clean up so that we do not leave any state behind.

```js
// cypress/integration/feature-flags/bookings-date-and-week.spec.js
...
// add to the end of the it block
// better: add to an after block so that it runs
// even when the test may fail halfway through
cy.task('cypress-ld-control:removeUserTarget', { featureFlagKey, userId })
```

### Boolean flag `slide-show`

The slide show rotates through the items every 3 seconds and can be stopped. When the flag is on, we want the rotation with the stop button available and fully feature tested. When the flag is off, the stop button should be gone and there should be no rotation. We also do not want to wait 3 seconds per rotation, we can use [`cy.clock`](https://docs.cypress.io/api/commands/clock) and [`cy.tick`](https://docs.cypress.io/api/commands/tick). This much already requires a spec file of its own and we see a pattern; a spec file per page and/or feature flag is not a bad idea.

We start with a sanity test for the flag, with an idempotent get call. After the sanity, we want to fully test the feature when the flag is on, and then off. Later when the feature becomes permanent, the flag-on case can be minified into its own spec by removing the FF portions, something to keep in mind for test structure.

```js
// cypress/integration/feature-flags/bookings-slide-show.spec.js

describe("Bookings slide-show", () => {
  const featureFlagKey = "slide-show";
  const userId = "aa0ceb";

  const testBtnColor = (i) =>
    cy
      .getByCy("bookables-list")
      .within(() => cy.checkBtnColor(i, "rgb(23, 63, 95)"));

  // a sanity test per flag is a good idea
  // would be removed when the flag is retired
  it("should get slide-show flags", () => {
    cy.task("cypress-ld-control:getFeatureFlag", featureFlagKey)
      .its("variations")
      .should("have.length", 2)
      .and((variations) => {
        expect(variations[0].value).to.eq(true);
        expect(variations[1].value).to.eq(false);
      });
  });

  context("Flag on off", () => {
    // the common state needs to happen after setting the flag
    const setupState = () => {
      cy.clock();
      cy.stubNetwork();
      cy.visit("/bookables");
      cy.tick(1000);
      return cy.wait("@userStub").wait("@bookablesStub");
    };

    const initialIndex = 0;

    it("should slide show through and stop the presentation", () => {
      // would be removed when the flag is retired
      cy.log("**variation 0: True**");
      cy.task("cypress-ld-control:setFeatureFlagForUser", {
        featureFlagKey,
        userId,
        variationIndex: 0,
      });

      setupState();

      // rotate through the items
      for (let i = initialIndex; i < 4; i++) {
        testBtnColor(i);
        cy.tick(3000);
      }
      // end up on the initial
      testBtnColor(initialIndex);

      // stop and make sure slide show doesn't go on
      cy.getByCy("stop-btn").click();
      cy.tick(3000).tick(3000);
      testBtnColor(0);
    });

    // the it block would be removed when the flag is retired
    it("should not show stop button or rotate bookables on a timer", () => {
      cy.log("**variation 1: False**");
      cy.task("cypress-ld-control:setFeatureFlagForUser", {
        featureFlagKey,
        userId,
        variationIndex: 1,
      });
      setupState();

      // no slide show or stop button
      cy.getByCy("stop-btn").should("not.exist");
      cy.tick(3000).tick(3000);
      testBtnColor(initialIndex);
    });

    // we need to clean up the flag after the tests
    // would be removed when the flag is retired
    after(() =>
      cy.task("cypress-ld-control:removeUserTarget", {
        featureFlagKey,
        userId,
      })
    );
  });
});
```

### Json flag `prev-next`

This flag toggles the four states of Previous and Next buttons. Similar to the `slide-show`, it applies to both Bookings and Bookables pages. That is realistic because LD FFs control React components, and in turn those components may be used on multiple pages. When testing FFs, we already stub the flag and test at component level. For e2e we can choose any page in which that component is used on. Unless there are extreme edge cases it should be ok not to test the same flag on multiple pages.

> *Always consider the possibility of risk based testing because however many tests you have, you can never prove your software is good. But one failing test means your software isn't good enough. Therefore use proven [test methodologies](https://dev.to/muratkeremozcan/mostly-incomplete-list-of-test-methodologies-52no) to gain the highest confidence with the minimal investment.*

Let's start with a sanity test; we want to get the flags and make sure they match the config we expect.

```js
// cypress/integration/feature-flags/bookables-prev-next.spec.js

describe("Bookables prev-next-bookable", () => {
  before(() => {
    cy.intercept("GET", "**/bookables").as("bookables");
    cy.visit("/bookables");
    cy.wait("@bookables").wait("@bookables");
  });

  const featureFlagKey = "prev-next-bookable";
  const userId = "aa0ceb";

  it("should get prev-next-bookable flags", () => {
    cy.task("cypress-ld-control:getFeatureFlag", featureFlagKey)
      .its("variations")
      .should("have.length", 4);
  });
});
```

This FF is a Json variant, therefore we will not be able to use a simple check like `expect(variations[0].value).to.eq(something)`. Time to shape the data. The part we are interested in is the `value` property for each one of the flags.

```js
cy.task("cypress-ld-control:getFeatureFlag", featureFlagKey)
  .its("variations")
  .should("have.length", 4)
  .and((variations) => {
    console.log(Cypress._.map(variations, (variation) => variation.value));
  });
```

That yields a neat array of 4 objects; exactly what we need:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k2vuweupx4gazv0dvw6l.png)

Here is one way we can assert it:

```js
const expectedFFs = [
  {
    Next: false,
    Previous: false,
  },
  {
    Next: true,
    Previous: false,
  },
  {
    Next: false,
    Previous: true,
  },
  {
    Next: true,
    Previous: true,
  },
];

it("should get prev-next-bookable flags v1", () => {
  cy.task("cypress-ld-control:getFeatureFlag", featureFlagKey)
    .its("variations")
    .should("have.length", expectedFFs.length)
    .and((variations) => {
      const values = Cypress._.map(variations, (variation) => variation.value);
      expect(values).to.deep.eq(expectedFFs);
    });
});
```

Here are 3 neater ways without variable assignments, showcasing TDD vs BDD assertions and our favorite; [cy-spok](https://github.com/bahmutov/cy-spok):

```js
import spok from 'cy-spok'

...
it('should get prev-next-bookable flags v2', () => {
  cy.task('cypress-ld-control:getFeatureFlag', featureFlagKey)
    .its('variations')
    .should('have.length', expectedFFs.length)
    .then((variations) =>
          Cypress._.map(variations, (variation) => variation.value)
         )
    // with TDD syntax, using should instead of then will ensure retry ability
    // .should((values) => expect(values).to.deep.eq(expectedFFs))
    // alternatively we can use the BDD syntax, same retry ability
    // .then((values) => cy.wrap(values).should('deep.eq', expectedFFs))
    // much concise versions with deep.eq or spok
    // .should('deep.eq', expectedFFs)
    .should(spok(expectedFFs))
})
```

We can even take it further up a notch by using another toy from Gleb; [cypress-should-really](https://github.com/bahmutov/cypress-should-really);

> [cypress-should-really](https://github.com/bahmutov/cypress-should-really) is a functional helper for Cypress, capable of a lot more than this simple usage.

```js
/// <reference types="cypress" />
import spok from 'cy-spok'
import { map } from 'cypress-should-really'

...

it('should get prev-next-bookable flags v3 (favorite)', () => {
  cy.task('cypress-ld-control:getFeatureFlag', featureFlagKey)
    .its('variations')
    .should('have.length', expectedFFs.length)
    .then(map('value'))
    .should(spok(expectedFFs))
})
```

All that is left is to test the flag variations. As usual, we control the flag, verify the UI and clean up the flag at the end.

```js
context("flag variations", () => {
  const flagVariation = (variationIndex) =>
    cy.task("cypress-ld-control:setFeatureFlagForUser", {
      featureFlagKey,
      userId,
      variationIndex,
    });

  it("should toggle the flag to off off", () => {
    flagVariation(0);

    cy.getByCy("prev-btn").should("not.exist");
    cy.getByCy("next-btn").should("not.exist");
  });

  it("should toggle the flag to off on", () => {
    flagVariation(1);

    cy.getByCy("prev-btn").should("not.exist");
    cy.getByCy("next-btn").should("be.visible");
  });

  it("should toggle the flag to on off", () => {
    flagVariation(2);

    cy.getByCy("prev-btn").should("be.visible");
    cy.getByCy("next-btn").should("not.exist");
  });

  it("should toggle the flag to on on", () => {
    flagVariation(3);

    cy.getByCy("prev-btn").should("be.visible");
    cy.getByCy("next-btn").should("be.visible");
  });

  after(() =>
    cy.task("cypress-ld-control:removeUserTarget", {
      featureFlagKey,
      userId,
    })
  );
});
```

### Numeric flag nex-prev

This is a similar functionality to the previous; Previous and Next buttons, effecting different components, and it is a numeric FF variant vs Json. The data is much simpler; values 0 through 3 vs an array of objects.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ypoj3pwzjrs25rxfplqq.png)

We can use the same exact assertion approach:

```js
// cypress/integration/feature-flags/users-next-prev.spec.js

import spok from "cy-spok";
import { map } from "cypress-should-really";

describe("Users nex-prev", () => {
  before(() => {
    cy.intercept("GET", "**/users").as("users");
    cy.visit("/users");
    cy.wait("@users").wait("@users");
  });

  const featureFlagKey = "next-prev";
  const userId = "aa0ceb";
  const expectedFFs = Cypress._.range(0, 4); // [0, 1, 2, 3]

  it("should get prev-next-user flags", () => {
    cy.task("cypress-ld-control:getFeatureFlag", featureFlagKey)
      .its("variations")
      .should("have.length", 4)
      .then(map("value"))
      .should(spok(expectedFFs));
  });
});
```

At this point, we can wrap the `cypress-ld-control` `cy.task` functions in helpers. Mind that `cy.task` cannot be included in a Cypress command, but a function is always fine.

```js
export const setFlagVariation = (featureFlagKey, userId, variationIndex) =>
  cy.task('cypress-ld-control:setFeatureFlagForUser', {
    featureFlagKey,
    userId,
    variationIndex
  })

export const removeUserTarget = (featureFlagKey, userId) =>
  cy.task('cypress-ld-control:removeUserTarget', {
    featureFlagKey,
    userId
  })

/** Can be used for clearing multiple user targets */
export const removeTarget = (featureFlagKey, targetIndex = 0) =>
  cy.task('cypress-ld-control:removeTarget', {
    featureFlagKey,
    targetIndex
  })
```

This part of the test is very similar to the previous feature:

```js
context("flag variations", () => {
  it("should toggle the flag to off off", () => {
    setFlagVariation(featureFlagKey, userId, 0);

    cy.getByCy("prev-btn").should("not.exist");
    cy.getByCy("next-btn").should("not.exist");
  });

  it("should toggle the flag to off on", () => {
    setFlagVariation(featureFlagKey, userId, 1);

    cy.getByCy("prev-btn").should("not.exist");
    cy.getByCy("next-btn").should("be.visible");
  });

  it("should toggle the flag to on off", () => {
    setFlagVariation(featureFlagKey, userId, 2);

    cy.getByCy("prev-btn").should("be.visible");
    cy.getByCy("next-btn").should("not.exist");
  });

  it("should toggle the flag to on on", () => {
    setFlagVariation(featureFlagKey, userId, 3);

    cy.getByCy("prev-btn").should("be.visible");
    cy.getByCy("next-btn").should("be.visible");
  });

  after(() => removeUserTarget(featureFlagKey, userId));
  
  // we could also use removeTarget()
  // which is like a deleteAll in case we have multiple users
  // mind that it will impact other tests that are concurrently running
  // after(() => removeTarget(featureFlagKey))
});
```

## Managing FF state with concurrent tests

Shared mutable state is the root of all evil. What would happen if a test was being concurrently executed by different entities?

Here is a killer way to qualify your tests for statelessness:

1. Wrap the it block *(could be describe block too)* with `Cypress._.times` *(or use [cypress-grep](https://github.com/cypress-io/cypress-grep))*
2. Start the app (*in this case the api and the app on one tab with `yarn dev`)*
3. On a second tab start Cypress *(`yarn cy:open`)*, have a browser selected.
4. On a third tab start Cypress again, but select a different browser.
5. repeat 4 *(optional)*

> For the record, the below is a Definition of Done for tests that will scale anywhere in the world. It can apply to any kind of testing and is particularly easier to achieve on lower level tests such as unit tests.
>
> Test Definition of Done
>
> - no flake
> - no hard waits/sleeps
> - stateless, multiple entities can execute - cron job or semaphore where not possible
>
> - no order dependency; each *it/describe/context* block can run with .only in isolation
>
> - tests handle their own state and clean up after themselves - deleted or de-activated entities
>
> - tests live near the source code
>
> - shifted left, as possible - begins with local server, sandbox, or ephemeral instance, works throughout deployments
>
> - low/minimal maintenance; no repeated test code, utilizes risk-based-testing
>
> - enough testing per feature to give us release confidence
>
> - execution evidence in CI
>
> - some visibility, as in a test report

### The tests are stateful

Let's have a look at one of the tests again. They are all in the same format after all.

```js
// cypress/integration/feature-flags/bookings-date-and-week.spec.js

describe("Bookings date-and-week", () => {
  before(() => {
    cy.intercept("GET", "**/bookables").as("bookables");
    cy.visit("/bookings");
    cy.wait("@bookables");
  });

  Cypress._.times(10, () => {
    it("should toggle date-and-week", () => {
      const featureFlagKey = "date-and-week";
      const userId = "aa0ceb";

      // .... sanity test

      setFlagVariation(featureFlagKey, userId, 0);
      cy.getByCy("week-interval").should("be.visible");

      setFlagVariation(featureFlagKey, userId, 1);
      cy.getByCy("week-interval").should("not.exist");

      cy.task("cypress-ld-control:removeUserTarget", {
        featureFlagKey,
        userId,
      });
    });
  });
});
```

Although the test is extremely stable -it is 10x repeatable- when multiple entities are executing it, they clash because there is a shared mutable state between them on LD side.

<img width="100%" style="width:100%" src="https://media.giphy.com/media/mUv6w3nxW8FrCdanrS/giphy.gif">

### Randomization can help statefulness

One way to address tests that have to be stateful - for example testing hardware - is to make the spec a semaphore; ensure only one entity can execute the test at a time. This means we probably would not run it on feature branches (we can use`ignoreTestFiles` in Cypress config file for local), and have some CI logic that allows only one master to run at a time. Still, the engineers would need to take care not to execute the test concurrently on a deployment while the matching CI pipeline is running.

The proper solution to tests sharing state would be randomization. Unless we are locked to *real* hardware - even then there is virtualization - we can randomize anything. We saw an example of this in the blog post about [email testing](https://dev.to/muratkeremozcan/test-emails-effortlessly-with-cypress-mailosaur-and-cy-spok-56lm) , under the section *Achieving Stateless tests with unique emails*. With [mailosaur](https://mailosaur.com/)  `any-name@unique-serverId.mailosaur.io` went to that unique email server inbox, and we differentiated between the emails by the randomized name.

 In LD context we have similar entities; **project key** - similar to email serverId - and **user key** - similar to the randomized `any-name` section of the email. For project key recall section 4 under [Controlling FFs with cypress-ld-control plugin](#controlling-ffs-with-cypress-ld-control-plugin) from the previous post in the series. For user key recall [Connect the app with LD section](#connect-the-app-with-ld). We have the project key taken care of but how do we randomize the user key?

### Randomizing the LD user key

 Per [LD docs](https://docs.launchdarkly.com/sdk/features/user-config#javascript) we either specify a user to target - which we have setup as Grace Hopper with key `aa0ceb` until now - or we can set an `anonymous: true` property so that LD creates randomized users and stored that user in local storage.  

> "Per the docs: *To create an anonymous user you can specify the "anonymous" property and omit the "key" property. In doing so, the LaunchDarkly client auto-generates a unique identifier for this user. The identifier is saved in local storage and reused in future browser sessions to ensure a constant experience.*"

```js
// src/index.js

...

;(async () => {
  const LDProvider = await asyncWithLDProvider({
    clientSideID: '62346a0d87293a1355565b20',
    // we do not want the React SDK to change flag keys to camel case
    // https://docs.launchdarkly.com/sdk/client-side/react/react-web#flag-keys
    reactOptions: {
      useCamelCaseFlagKeys: false
    },
    // https://docs.launchdarkly.com/sdk/client-side/react/react-web#configuring-the-react-sdk
    user: {
      // key: 'aa0ceb',
      // name: 'Grace Hopper',
      // email: 'gracehopper@example.com'

      // to create an anonymous user you can specify the "anonymous" property 
      // and omit the "key" property. 
      // In doing so, the LaunchDarkly client
      // auto-generates a unique identifier for this user.
      // The identifier is saved in local storage and reused in future
      // browser sessions to ensure a constant experience.
      anonymous: true
    }
  })

```

Toggling anonymous vs defined user, we can see that a local storage variable is created by LD upon visiting the page.

<div style="height: 0; padding-bottom: calc(62.48%); position:relative; width: 100%;"><iframe allow="autoplay; gyroscope;" allowfullscreen height="100%" referrerpolicy="strict-origin" src="https://www.kapwing.com/e/624360eafea8da006d90d822" style="border:0; height:100%; left:0; overflow:hidden; position:absolute; top:0; width:100%" title="Embedded content made on Kapwing" width="100%"></iframe></div><p style="font-size: 12px; text-align: right;">Video edited on <a href="https://www.kapwing.com/video-editor">Kapwing</a></p>

In the beginning of the test, if we can get that value from local storage, we will have solved one part of the puzzle. We can utilize [cypress-localstorage-commands](https://github.com/javierbrea/cypress-localstorage-commands) plugin. Install with `yarn add -D cypress-localstorage-commands` and add it to the index file.

```javascript
// cypress/support/index.js
import "cypress-localstorage-commands"
```

At first it may not be obvious from `cypress-ld-control` [api docs](https://github.com/bahmutov/cypress-ld-control#api) , but `setFeatureFlagForUser` takes a `userId` argument and **creates that userId if it does not exist**. Until now we kept it simple and used `const userId = 'aa0ceb'` in every spec, which points to the already existing LD user. If we instead use an arbitrary string, that key will appear on the LD Targeting tab.

<img width="100%" style="width:100%" src="https://media.giphy.com/media/gIayve1UiqRT5wmvxS/giphy.gif">

We have 3 facts down

1. We can  have an anonymous user per browser  and the user's id gets created by LD and stored in local storage.
2. We can access local storage via [cypress-localstorage-commands](https://github.com/javierbrea/cypress-localstorage-commands).
3. We can use [cypress-ld-control](https://github.com/bahmutov/cypress-ld-control#api) to set and remove new keys/Ids.

All we have to do is access local storage, make a variable assignment, and use that variable throughout the test. Cypress clears local storage between tests, so we will automatically have stateless executions with unique flags. For tests with multiple `it` blocks, we can utilize local storage commands to control what we need.

Let's refactor the `date-and-week` spec accordingly. There are [multiple ways to handle variable assignments in a test](https://www.youtube.com/watch?v=VUx-Ztkbtaw) and the below is our favorite:

```js
// cypress/integration/feature-flags/bookings-date-and-week.spec.js

import { setFlagVariation, removeUserTarget } from '../../support/ff-helper'

describe('Bookings date-and-week', () => {
  const featureFlagKey = 'date-and-week'
  // the variable will be available throughout the spec
  let userId

  before(() => {
    cy.intercept('GET', '**/bookables').as('bookables')
    cy.visit('/bookings')
    cy.wait('@bookables')

    // grab the local storage value LD set 
    // assign it to our variable
    cy.getLocalStorage('ld:$anonUserId').then((id) => (userId = id))
  })

  it('should toggle date-and-week', () => {
    cy.log(`user ID is: ${userId}`)

    cy.task('cypress-ld-control:getFeatureFlag', featureFlagKey)
      .its('variations')
      .then((variations) => {
        Cypress._.map(variations, (variation, i) =>
          cy.log(`${i}: ${variation.value}`)
        )
      })
      .should('have.length', 2)
      .and((variations) => {
        expect(variations[0].value).to.eq(true)
        expect(variations[1].value).to.eq(false)
      })

    cy.log('**variation 0: True**')
    setFlagVariation(featureFlagKey, userId, 0)
    cy.getByCy('week-interval').should('be.visible')

    cy.log('**variation 1: False**')
    setFlagVariation(featureFlagKey, userId, 1)
    cy.getByCy('week-interval').should('not.exist')
  })

  after(() => removeUserTarget(featureFlagKey, userId))
})

```

Every time the test runs, there is a unique LD user id, consequently our initial concurrency test will pass with this setup.

### Handling multiple `it` blocks

Cypress clears local storage between tests — `it` blocks — and LD sets a random user in local storage. This works great when a spec file has a single it block, but what happens when there are multiple it blocks? We can handle that with [cypress-localstorage-commands](https://github.com/javierbrea/cypress-localstorage-commands) as well.

There are only a few things we have to do:

1. Like before, get the anonymous user id from local storage, assign it to a variable (ex: `userId`) and make it available throughout the tests.

2. Before each it block, restore a snapshot of the whole local storage. Any name will do for the snapshot identifier, we can even use the unique `userId` we get from local storage.

3. After each it block, save a snapshot of the whole local storage. Again, `userId` variable will be fine.

```js
// cypress/integration/feature-flags/bookables-prev-next.spec.js

import { setFlagVariation, removeUserTarget } from '../../support/ff-helper'

describe('Bookables prev-next-bookable', () => {
  /* expectedFFs are not impacted */
  const featureFlagKey = 'prev-next-bookable'
  // the variable will be available throughout the spec
  let userId

  before(() => {
    cy.intercept('GET', '**/bookables').as('bookables')
    cy.visit('/bookables')
    cy.wait('@bookables').wait('@bookables')

    // assign the variable in the beginning
    cy.getLocalStorage('ld:$anonUserId').then((id) => (userId = id))
  })

  // restore & take a snapshot
  // we can name that snapshot anything
  // therefore we can use the unique userId for it without issues
  beforeEach(() => cy.restoreLocalStorage([userId]))
  afterEach(() => cy.saveLocalStorage([userId]))

  context('flag sanity', () => {
  /* not impacted */
  })

  context('flag variations', () => {
    it('should toggle the flag to off off', () => {
      setFlagVariation(featureFlagKey, userId, 0)

      cy.getByCy('prev-btn').should('not.exist')
      cy.getByCy('next-btn').should('not.exist')
    })

    it('should toggle the flag to off on', () => {
      setFlagVariation(featureFlagKey, userId, 1)

      cy.getByCy('prev-btn').should('not.exist')
      cy.getByCy('next-btn').should('be.visible')
    })

    it('should toggle the flag to on off', () => {
      setFlagVariation(featureFlagKey, userId, 2)

      cy.getByCy('prev-btn').should('be.visible')
      cy.getByCy('next-btn').should('not.exist')
    })

    it('should toggle the flag to on on', () => {
      setFlagVariation(featureFlagKey, userId, 3)

      cy.getByCy('prev-btn').should('be.visible')
      cy.getByCy('next-btn').should('be.visible')
    })
  })

  after(() => removeUserTarget(featureFlagKey, userId))
})

```

Here is the key refactor from `slide-show` spec. The main idea is that LD only sets the local storage after having visited the page, therefore we have to arrange our test hooks accordingly. Here are the relevant parts of the spec:

```js
// cypress/integration/feature-flags/bookings-slide-show.spec.js

context('Flag on off', () => {
  const initialIndex = 0
  let userId

  beforeEach(() => {
    // nothing to restore for the first test, 
    // but we need it for subsequent tests
    cy.restoreLocalStorage([userId])

    // setting up state for the test
    cy.clock()
    cy.stubNetwork()
    cy.visit('/bookables')
    cy.tick(1000)
    cy.wait('@userStub').wait('@bookablesStub')

    // assign the variable and use it throughout the spec
    cy.getLocalStorage('ld:$anonUserId').then((id) => (userId = id))
  })

  afterEach(() => cy.saveLocalStorage([userId]))

  it('should slide show through and stop the presentation', () => {
    setFlagVariation(featureFlagKey, userId, 0)

    for (let i = initialIndex; i < 4; i++) {
      testBtnColor(i)
      cy.tick(3000)
    }
    testBtnColor(initialIndex)

    cy.getByCy('stop-btn').click()
    cy.tick(3000).tick(3000)
    testBtnColor(0)
  })

  it('should not show stop button or rotate bookables on a timer', () => {
    setFlagVariation(featureFlagKey, userId, 1)

    cy.getByCy('stop-btn').should('not.exist')
    cy.tick(3000).tick(3000)
    testBtnColor(initialIndex)
  })

  after(() => removeUserTarget(featureFlagKey, userId))
})
```

Here is the relevant refactor from `users-next-prev` spec.

```js
// cypress/integration/feature-flags/users-next-prev.spec.js

  let userId

  before(() => {
    cy.intercept('GET', '**/users').as('users')
    cy.visit('/users')
    cy.wait('@users').wait('@users')

    // assign the variable in the beginning
    cy.getLocalStorage('ld:$anonUserId').then((id) => (userId = id))
  })

  // preserve the local storage between tests
  beforeEach(() => cy.restoreLocalStorage([userId]))
  afterEach(() => cy.saveLocalStorage([userId]))
```



## Summary

We have two powerful ways to deal with LaunchDarkly Feature flags; stubbing the FFs with a custom command, and controlling the FFs in a stateless manner with `cypress-ld-control-plugin`.

- When not testing the FFs, stub them, just as we stub the network while testing non-network-relevant features. Test the latest and greatest version of the features on every deployment, as early as possible; shift left.

- Test the FFs in isolation with due diligence, as early as possible; again shift left. The tests are stateless, so they could run as early as feature branches, on localhost.

- Have a spec per feature flag, preferably in a FF related folder, and test every variant of that flag.

- Use combinatorial testing if the flag has too many variants, in order to reduce effort while keeping high confidence.

- When the feature is permanent, you can re-use parts of the FF specs, or discard them, whichever is appropriate.

Once we have accomplished the above, testing the consequences of toggling a flag on various environments is superfluous; we already have enough confidence that the flags work really well. Therefore we can freely toggle them in any environment, and they should work as expected.

> Note: the above is the recommended approach, however, if the team is not comfortable, FFs could be tested in `dev` environment vs feature branches.
> The teams could even opt for a fully idempotent strategy; only test getting the feature flags, and test the variants of a flag through network stubbing. In our opinion, stubbing the flag is already done at unit or component level, therefore the value-add is questionable.  

Stay tuned for a blog testing LaunchDarkly feature flags with a deployed service.
