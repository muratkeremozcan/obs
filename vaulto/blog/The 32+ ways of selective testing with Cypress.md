- [Building the test architecture, increasing adoption, improving the developer experience](#building-the-test-architecture-increasing-adoption-improving-the-developer-experience)
- [The many ways of selecting tests in Cypress](#the-many-ways-of-selecting-tests-in-cypress)
  - [Built-in ways of selective testing](#built-in-ways-of-selective-testing)
  - [Selective tests with plugins](#selective-tests-with-plugins)
- [Combined ways of selecting tests](#combined-ways-of-selecting-tests)
  - [Notice that there are a few test negators](#notice-that-there-are-a-few-test-negators)
  - [Notice that only certain methods can work in combination](#notice-that-only-certain-methods-can-work-in-combination)
  - [An example for a very specific case with 5 combinations](#an-example-for-a-very-specific-case-with-5-combinations)
- [How can it work in the CI as it works locally with Cypress runner?](#how-can-it-work-in-the-ci-as-it-works-locally-with-cypress-runner)
  - [Handle the environments in config files, and define a custom environment variable](#handle-the-environments-in-config-files-and-define-a-custom-environment-variable)
  - [Abstract away the logic in the test](#abstract-away-the-logic-in-the-test)
  - [Use it with the GitHub Action](#use-it-with-the-github-action)
    - [Testing against localhost in CI](#testing-against-localhost-in-ci)
    - [Testing against deployments in CI](#testing-against-deployments-in-ci)

## Building the test architecture, increasing adoption, improving the developer experience

A common challenge faced while building the test architecture is deciding which e2e tests to execute or skip per deployment, and maybe when to add secondary combinations of browser and viewports. Once these are identified, the goal is to increase adoption and improve the developer experience when executing e2e tests locally and in CI.

In the context of Cypress, let's explore built-in ways of selecting tests, plugins that expand the possibilities, the GitHub action that provides CI conveniences, and how all of these can combine harmoniously for a similar developer experience between local machines and CI.

## The many ways of selecting tests in Cypress

### Built-in ways of selective testing

There are a few options for selective testing that comes built-in with Cypress.

- Using [config files](https://docs.cypress.io/guides/references/configuration#Folders-Files) - My personal favorite way of doing it because we have one file per deployment, and most if not all the configuration can be done here.

  For example, when running on dev deployment you want to ignore test files under the prod folder. At  `cypress/config/dev.json`:

  ```json
  {
    "extends": "../../cypress.json",
    "baseUrl": "https://www.deployed-dev.com",
    "projectId": "123abc",
    "ignoreTestFiles": ["**/prod/*"]
  }
  ```

  The `testFiles` property would work the opposite way, only running specific tests for that configuration.

  > Assume we are using [cypress-extends](https://github.com/bahmutov/cypress-extends) to inherit from base `cypress.json` config file, which may look like:
  >
  > ```json
  > {
  >   "retries": {
  >     "runMode": 2,
  >     "openMode": 0
  >   },
  >   "projectId": "123abc"
  >   "defaultCommandTimeout": 10000,
  > }
  > ```
  >
  >
  >
  > Assume We are passing in `--config-file` when running or opening Cypress.
  > `"cypress:open-dev": "cypress open --config-file cypress/config/dev.json"`

- Using [command line](https://docs.cypress.io/guides/references/configuration#Command-Line) :

  > takes precedence over the config file

  `cypress run --ignoreTestFiles="**/prod/*"` - would overwrite the config file.
  `cypress run --browser firefox` - would add to the config file. You can use this when you want the config file to apply to all common deployments -ex:  dev and stage deployments- but you want to control the browser choice in CI.

- Using CLI [environment variables](https://docs.cypress.io/guides/references/configuration#Environment-Variables)

  > takes precedence over the config file

  `export CYPRESS_IGNORE_TEST_FILES="**/prod/*" cypress run` - very similar to command line style.

- Using [configuration API](https://docs.cypress.io/api/plugins/configuration-api#Usage) - advanced, can be overkill. We have not needed to use it at work, yet.

- Within the test, [Cypress.config()](https://docs.cypress.io/guides/references/configuration#Cypress-config) - useful if you need one off specs or individual tests to be an exception.

  > takes precedence over other ways of configuration

  `Cypress.config(viewportWidth: 1280, viewportHeight: 720)`

- Within the test, using the Configuration Object - similar usage as `Cypress.config()` , and very practical.

  > takes precedence over other ways of configuration

  ```javascript
  describe('login', { viewportWidth: 1280, viewportHeight: 720}, () => {
      it('should login', () => {..}
  ```

### Selective tests with plugins

These are some personal favorites, for the control and specificity they provide and how they can combine with built-in ways of selecting tests.

- [cypress-grep](https://github.com/cypress-io/cypress-grep), - for example for a certain deployment, you want to only run tests with a certain string in the title or only run the tests that have a tag.

  Assume you have a few tests, and one of them is this:

  ```javascript
  it('auth user login', { tags: 'smoke' }, () => {
    ...
  })
  ```

  ```bash
  # run only the tests with "auth user" in the title
  $ npx cypress run --env grep="auth user"
  # run tests with "login" or "auth user" in their titles
  # by separating them with ";" character
  $ npx cypress run --env grep="login; auth user"
  # run only the tests tagged "smoke"
  $ npx cypress run --env grepTags=@smoke
  # but also those that have "auth" in their titles
  $ npx cypress run --env grep=auth,grepTags=smoke
  ```

- [cypress-skip-test](https://github.com/cypress-io/cypress-skip-test) - this is a special one for ability to negate tests and have combinations in itself

  ```javascript
  it('combination of skip and only', () => {
    cy.skipOn('firefox')
    cy.onlyOn('electron').onlyOn('mac')
    cy.log('running test')
  })
  ```

## Combined ways of selecting tests

### Notice that there are a few test negators

You have two options to skip tests. If the rest of the test selection methods are synonymous to `array.filter`, these would be synonymous to `array.unfilter`/`array.skip`

1. `ignoreTestFiles` property in a config file, can skip folders or tests

2. `cy.skipOn()` / `skipOn()` can skip test blocks -`describe`, `context`- or individual tests -`it`.

### Notice that only certain methods can work in combination

The built-in group methods take precedence over each other, or add-on to the configuration where they do not overlap. For example, we can use the config-file for most of the deployment configurations, and add on which browser to run the tests via command line.

The plugin group methods can work with the built-in methods, expanding our choices.

Not including the [configuration API](https://docs.cypress.io/api/plugins/configuration-api#Usage), there are 5 practical ways of combining configurations: config file, CLI, within the test with configuration object, cypress-grep, cypress-skip-test. This is 2^5, 32 combinations at least! Thanks Cypress!

### An example for a very specific case with 5 combinations

Imagine you have a very specific need for a test execution:

- needs to run against dev deployment
- needs to run with Firefox
- needs to only run the smoke tests
- needs to run in a certain viewport for a spec/top describe block *(usually viewport can be better as a CLI param, but assume we only need it for one spec so we can show a use case here for the configuration object)*
- needs to skip on mac, and still execute on other OS'

How would we tackle this?

In CLI:
 `cypress run --config-file cypress/config/dev.json --browser firefox --env grepTags=@smoke`

In the test:

```javascript
describe('login', { viewportHeight: 600, viewportWidth: 1000,}, () => {
   
it('auth user login', { tags: 'smoke' }, () => {
  cy.skipOn('mac')
  // the rest of the test
})
```

As you can see we have so much control over how we execute our tests, and even five combinations is overkill for most use cases. Usually we will be ok with 3 combinations of selective test configuration.

## How can it work in the CI as it works locally with Cypress runner?

Assume we have many applications and services we are using Cypress with. We will have many yml files, or someday a fully working templating functionality (thank you in advance GitHub Actions).

Wouldn't it be nice if our engineers  could have the same experience with their specific CI configurations as they have locally on their laptops with Cypress runner? In the Cypress runner, the engineer picks a browser and just executes the test(s). How can we abstract away CI configuration complexities to mininal sources of truth so that we can have a unified, concise approach to do selective testing?

> The ideas are borrowed from an [older pattern of doing selective testing](https://cypress.slides.com/cypress-io/siemens-case-study#/3/1/1 ) and [cypress-skip-test plugin](https://github.com/cypress-io/cypress-skip-test#environment).

### Handle the environments in config files, and define a custom environment variable

We are able to set custom env vars per a cypress config file, abstract away the logic, and have a declarative syntax to manipulate selective testing.

From [Cypress docs](https://docs.cypress.io/guides/guides/environment-variables#Option-1-configuration-file), we know that *any key/value you set in your [configuration file (`cypress.json` by default)](https://docs.cypress.io/guides/references/configuration) under the `env` key will become an environment variable.*

At `cypress/config/dev.json`, we can have a custom variable ENVIRONMENT, and match the value with the name of the config file `dev`:

```json
{
  "extends": "../../cypress.json",
  "baseUrl": "https://www.deployed-dev.com",
  "projectId": "123abc",
  "ignoreTestFiles": ["**/prod/*"]
  "env": {
    "ENVIRONMENT": "dev"
  }
}
```

### Abstract away the logic in the test

After this, cypress-skip-test has access to the custom variable. Instead of something long and imperative such as:

```javascript
cy.skipOn(Cypress.config('baseUrl') === 'https://www.deployed-dev.com')
```

We can use:

```javascript
cy.skipOn('dev')
```

> note that [cypress-grep](https://www.deployed-dev.com) can also use the "env" property

### Use it with the GitHub Action

We know three facts, and we want a way to combine them.

1. From [Cypress docs](https://docs.cypress.io/guides/guides/environment-variables#Option-1-configuration-file), we know that we can set any custom environment variable in the [configuration file](https://docs.cypress.io/guides/references/configuration) under the `env` key.

2. Cypress maintains a [very neat GitHub Action](https://github.com/cypress-io/github-action) that makes CI usage convenient with custom parameters.

3. With GitHub actions we can [set a custom environment variable](https://docs.github.com/en/actions/learn-github-actions/workflow-commands-for-github-actions#setting-an-environment-variable) with `echo "{name}={value}" >> $GITHUB_ENV`

If these ideas can combine somehow, and the CI could figure out what deployment it is executing the test for, we can have the same experience for the engineers using the CI as they would when they use the Cypress runner locally on their laptops.

#### Testing against localhost in CI

Let us do the simpler case first, where CI does not have to figure out what deployment it is running against yet. For testing against localhost in CI, we can just combine setting a custom variable in the Cypress config file (fact 1) with the custom properties from the Cypress GitHub Action (fact 2). After all it is a good practice to have separate ymls for PR testing and deployment testing.

Assume we have a config file for local testing `cypress/configs/local.json`:

```json
{
  "extends": "../../cypress.json",
  "baseUrl": "http://localhost:3000'",
  "projectId": "123abc",
  "ignoreTestFiles": ["**/prod/*"]
  "env": {
    "ENVIRONMENT": "local"
  }
}
```

With this config file setup we tell Cypress which config it should use and which environment it should run against (1).

In the below GitHub workflow configuration file, let's say `local-e2e.yml`, we specify which config file to use with the Cypress GitHub Action's `config-file` property (2).

Just like how we use Cypress runner as a local user, we pick which browser to use with the `browser` property of the action.

At this point, once Cypress launches in the CI step, it  knows there is a property `"ENVIRONMENT"` with the value of `"local"`. We have linked facts (1) and (2); cypress-config file can easily work with the Cypress GitHub Action.

This way we can have the same experience in the CI as we would on our laptop with Cypress runner, and use something like `cy.skipOn('dev')` in the tests without worrying about any additional CI configuration.

```yaml
# tests against the app being served locally while running in CI
name: e2e local
on:
  pull_request:
    types: [ opened, reopened, edited, synchronize ]
    
# if this branch is pushed back to back, cancel the older branch's workflow      
concurrency:
  group: ${{ github.ref }} && ${{ github.workflow }}
  cancel-in-progress: true

jobs:
  test:
    strategy:
      # uses 2 CI machines to run tests in parallel
      matrix:
        machines: [1, 2]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Cypress tests 🧪 on local
        uses: cypress-io/github-action@v2
        with:
          # the action gives us the ability to pick the browser  
          browser: chrome 
          # for localhost testing, we need a command that serves the application 
          start: yarn start-my-app 
          # for localhost testing, the action waits the app to be ready, on any port 
          wait-on: 'http://localhost:3000'
          # KEY: we can specify the config file in the GHA and establish the linkage
          config-file: cypress/config/local.json 
          record: true # records on cypress dashboard
          parallel: true # parallelizes tests
          group: 'local ui e2e' # for nice labelling on the dashboard
          # adds a convenience filter when using Cypress Dashboard; 
          # i.e. we can filter test executions by environment name 
          tag: local 
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}  
```

#### Testing against deployments in CI

We need a way for CI to figure out which deployment we are executing tests against (3), and use that to drive the Cypress GitHub Action (2) which is already working with the Cypress config file (1).  

Assume we are using two config files `cypress/config/dev.json` and `cypress/config/stage.json`:

```json
{
  "extends": "../../cypress.json",
  "baseUrl": "https://www.deployed-dev.com",
  "projectId": "123abc",
  "ignoreTestFiles": ["**/prod/*"]
  "env": {
    "ENVIRONMENT": "dev"
  }
}
```

```json
{
  "extends": "../../cypress.json",
  "baseUrl": "https://www.deployed-stage.com",
  "projectId": "123abc",
  "ignoreTestFiles": ["**/prod/*"]
  "env": {
    "ENVIRONMENT": "stage"
  }
}
```

In the below GitHub workflow configuration file, let's say `deployment-e2e.yml`, we specify which config file to use with the GitHub Action's `config-file` property. Exactly how we did for `local-e2e.yml`, facts (1) and (2) are now linked.

To derive fact (3) we add some bash logic which figures out what environment the CI is in, the details of how is in the yml sample - credit goes to [Christopher Lawrence](https://www.linkedin.com/in/chrstphrjlawrence/) for figuring this out. All we need is for this logic to provide us with a deployment name such as `dev` or `stage`, then we can use that value in the GitHub Action.

Just like how we use Cypress runner as a local user, we pick which browser to use with the `browser` property of the action.

At this point, when starting the GitHub Action, all 3 facts compose together:

- Cypress launches in the CI step driven by the custom environment value -`dev` or `stage`- we achieved from the bash logic - *fact (3)*,
- The config file is selected per this value via the config-file property of the GitHub Action - *fact (2)*,
- The config file has a matching value in the ENVIRONMENT property which in turn drives the tests - *fact (1)*.

Again, we use the same  `cy.skipOn('dev')` in the test, not have to change a thing in the spec file, not worry about any additional CI configuration.

```yaml
# assume trunk based deployment
# we are pushing to master, deploying master, running tests against dev deployment
# we also want to push tags, deploy, run tests against stage deployment
# additionally we want to execute e2e against any deployment on demand with dispatch
name: e2e deployment
on:
  push:
    branches: ['master']
    tags: [ 'v*.*.*' ]
   # the workflow dispatch takes the value of the environment, 
   # which matches cypress/config/<environmentName>.json : dev or stage
   workflow_dispatch: 
    inputs:
      environment:
        description: 'Env to run tests in'
        required: true

# if this branch is pushed back to back, cancel the older branch's workflow      
concurrency:
  group: ${{ github.ref }} && ${{ github.workflow }}
  cancel-in-progress: true

jobs:
  test:
    strategy:
      # uses 2 CI machines to run tests in parallel
      matrix:
        machines: [1, 2]
    runs-on: ubuntu-latest
    env:
      # (3): we add custom variables to the environment
      # we will use 2 variables to showcase a complex logic in bash
      # with a switch statement and if else  
      EVENT_NAME: ${{ github.event_name }} # helps identify if this is a push or dispatch
      REF: ${{ github.ref }} # helps identify branch name, github ref is just your branch name
    steps:
      - uses: actions/checkout@v2

      # (fact 3): our custom step which figures out what environment the CI is in
      # if this is a workflow dispatch we set a custom variable ENVIRONMENT 
      # with the value of what is entered in GitHub UI: dev, stage, or wrong text
      # if this is push event, we identify from the branch name
      # whether it is master branch, or this is a tag
      # if master, we set the custom variable ENVIRONMENT as dev, otherwise as stage
      - name: Set environment variable
        run: |
          case $EVENT_NAME in
            workflow_dispatch)
              ENVIRONMENT=${{ github.event.inputs.environment }}
              ;;
            push)
              if [[ $REF == *"master" ]]
              then
                ENVIRONMENT=dev
              else
                ENVIRONMENT=stage
              fi
              ;;
          esac

          echo "ENVIRONMENT=$ENVIRONMENT" >> $GITHUB_ENV
      # from GitHub docs we know how to set a custom environment variable
      # echo "{name}={value}" >> $GITHUB_ENV
      # https://docs.github.com/en/actions/learn-github-actions/workflow-commands-for-github-actions#setting-an-environment-variable

      # (fact 2): we use the appropriate cypress config file
      # to run tests against a deployment
      # we can use the value ${{ env.ENVIRONMENT }} from the previous step
      - name: Cypress tests 🧪 on ${{ env.ENVIRONMENT }} 
        uses: cypress-io/github-action@v2
        with:
          browser: chrome
          # (fact 1): the config file has a variable ENVIRONMENT
          # with a value that matches the name of the config file 
          # we don't have to define anything additional in CI
          config-file: cypress/config/${{ env.ENVIRONMENT }}.json 
          record: true # records on cypress dashboard
          parallel: true # parallelizes tests
          # for nice labelling on the dashboard
          group: ${{ env.ENVIRONMENT } ui e2e
          # adds a convenience filter when using Cypress Dashboard;
          # i.e. we can filter test executions by environment name
          tag: ${{ env.ENVIRONMENT }} 
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}  
```

With this setup the engineers are able to use Cypress in the CI the same way they would use Cypress runner locally on their laptops:

- They opened (or ran) the tests against the deployment they want: `cypress:open-local` | `cypress:open-dev` | `cypress:open-stage`

- They selected the browser

- They were able to work at test level, skip or execute  not worrying about any other CI specific configuration parameters

> We are working on a sample repo similar to [angular-playground](https://github.com/muratkeremozcan/angular-playground) which would help internal training, external information sharing and collaborating with Cypress on reproducible issues. This post will be updated when the repo is ready.  
