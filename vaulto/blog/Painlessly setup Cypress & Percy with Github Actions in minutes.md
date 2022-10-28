# # Painlessly setup Cypress & Percy with Github Actions in minutes

We have not seen an up to date, fully documented example using Cypress, Github Actions and Percy and thought this could help others in the future.

  

Any guide is lackluster without reproducible code, so here is the [full repo](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress).

  

The branch prior to the changes can be checked out at `before-visual-testing`. The steps can be followed to reach the final version on `main` branch. The changes in the guide can be found in [this PR](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/pull/99).

  

- [Install Percy locally](#install-percy-locally)

- [Implement a test](#implement-a-test)

- [Sign up](#sign-up)

- [Test locally](#test-locally)

- [Test in CI](#test-in-ci)

- [**Pull Request integration**](#pull-request-integration)

- [Further customization](#further-customization)

- [Custom selector](#custom-selector)

- [Custom viewports and browsers](#custom-viewports-and-browsers)

  

### Install Percy locally

  

`yarn add -D @percy/cli @percy/cypress`

  

At `cypress/support/index.js` add a line : `import '@percy/cypress'`

  

### Implement a test

  

Visual tests can be added to any spec. The spec that makes the most sense in the app's e2e test suite is `cypress/integration/ui-integration/user-context-retainment.spec.js` because we can verify the user's avatar with visual diffing.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rnl9mszbi1uuwr0ytgaf.png)

  

Here is the main idea of visual tests:

  

* All we need is one line; `cy.percySnapshot('any snapshot name')`. This will record a default snapshot and compare that default with the new, in subsequent test executions. We have to accept the initial snapshot once.

* From then on, new snapshots matching the default get auto-accepted.

  

* Non-matching new snapshots prompt a notification on the Percy interface; we either have to reject or accept this new baseline. If we reject, it is a defect. If we accept we have a new base line and the cycle continues.

  

The big selling point of visual snapshot services is the AI; the AI is trainable over time and we can train it to ignore petty snapshot diffs we might not care for. Be mindful that given a snapshot name, we can train the AI to be really bad by accepting any diff. Therefore we need to take care not to thumbs up valid failures. We can reset the training by changing the snapshot name, viewport, or any part of the code.

  

Here is how visual testing looks in a spec.

  

```javascript

// cypress/integration/ui-integration/user-context-retainment.spec.js

  

it('Should keep the user context between routes', () => {

cy.fixture('users').then((users) => {

cy.get('.user-picker').select(users[3].name)

cy.contains('Users').click()

  

cy.wait('@userStub')

cy.url().should('contain', '/users')

cy.get('.item-header').contains(users[3].name)

  

// the visual test

cy.percySnapshot('User selection retainment between routes')

})

})

```

  

Let's try that out with `yarn cy:open-e2e`. In the repo this command does all the legwork of starting the api, the ui and the tests. Select the test `user-context-retainment` and run it. We will see an entry per Percy command, notifiying that it is disabled. Nothing happened here, except that we verified the plugin setup.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8m0frbh8tt5fn1s9vz57.png)

  

To actually run a visual diff, we need a Percy account and token. Visual testing only runs when hooked up to this account and executed via `cy run`.

  

### Sign up

  

[Sign up here](https://www.browserstack.com/users/sign_in?ref=percy).

  

Create a new project.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/57v9y0bphxse2f01gc0e.png)

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tsvmefi37a41b9pn16yo.png)

  

Grab the `PERCY_TOKEN`. This has to be an environment variable for the machine, bash or CI. Mind that it is not a Cypress environment variable; it is not for `cypress.env.json`.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/tv83s9sn0658hmhg65tw.png)

  

> *It makes sense to use [dotenv](https://github.com/motdotla/dotenv) and store this in a .gitignored `.env` file.*

  

We also have to add `PERCY_TOKEN` to CI environment variables.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gcdp7s0xllj2e859k9a8.png)

  

If you are following the guide, by now a Percy verification email should arrive. Make sure to verify the email before proceeding.

  

### Test locally

  

In bash, we need to export the token:

  

```bash

export PERCY_TOKEN=80f74d9b1fdfce3b4f31331849357fcf...

```

  

Then we need to run the spec(s) with visual tests in them.

  

```bash

# start the ui and the api on a different tab

yarn dev

  

# run a visual test

# note: you can use any selection of specs

yarn percy exec -- cypress run --spec 'cypress/integration/ui-integration/user-context-retainment.spec.js'

```

  

At the end of the run, we should see results looking like this:

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ioetereglv5pmrg6g0m7.png)

  

We can see the run at the Percy web interface:

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ro0j7qed72h91eryrlog.png)

  

Because this is the initial execution, we have to let Percy know if it is an acceptable snapshot. We can approve the build by either the big green button - approve everything - or the green thumbs up if we want to approve one snapshot at a time.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ocbq96jqwe669vkpo6ve.png)

  

Let's run another test, this time we want to stop the API so that there is no image. Simply stop the `yarn dev` script, and only start the UI with `yarn start` . The app is launched on localhost:3000 and the image appears broken. Below is a before and after.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/40u7ojew2ltw6k8us0a1.png)

  

Once the test is executed, Percy confirms the broken image by detecting the visual diff, and now the variance needs to be reviewed. In real life, we would reject this alteration and the fail the build.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/soakqphk5pt4o8lijjh6.png)

  

### Test in CI

  

We can either add an additional step to an existing job, or have a complete new job for visual tests. In our opinion a whole new job - the latter - will enable better customizability if teams are utilizing [Github's remote reusable workflows](https://www.youtube.com/watch?v=m03ru99eBuc). In that scenario teams would have filled-in yaml templates which run certain Github Actions jobs common to every team, and at the end of the template they can insert their custom visual testing job.

  

> *We ran out of parallelization during testing, in the source code you will find a slightly different yml.*

>

> *We disabled video and screen shot recording in visual tests, because we already have a replica of the spec(s) doing normal testing and recording those artifacts.*

  

```yml

## .github/workflows/main.yml

  

# option 1: append the visual testing as a step

  

cypress-e2e-tests:

strategy:

matrix:

machines: [1, 2]

needs: [install-dependencies]

runs-on: ubuntu-latest

container: cypress/included:9.4.1

steps:

- uses: actions/checkout@v3

  

- uses: bahmutov/npm-install@v1

with: { useRollingCache: true }

  

- name: Cypress e2e tests 🧪

uses: cypress-io/github-action@v3.0.3

with:

install: false

start: yarn dev

wait-on: 'http://localhost:3000'

browser: chrome

record: true

parallel: true

group: e2e-tests

tag: e2e-tests

env:

CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}

LAUNCH_DARKLY_PROJECT_KEY: ${{ secrets.LAUNCH_DARKLY_PROJECT_KEY }}

LAUNCH_DARKLY_AUTH_TOKEN: ${{ secrets.LAUNCH_DARKLY_AUTH_TOKEN }}

  

# visual test appended as a step

# preferred if not using GHA Remote Reusable Workflows

- name: Cypress visual tests 🧪

uses: cypress-io/github-action@v3.0.3

with:

install: false # no need to install because of the above 2

start: yarn dev # concurrently starts ui and api servers

wait-on: 'http://localhost:3000'

browser: chrome

command: yarn percy exec -- cypress run --spec 'cypress/integration/ui-integration/user-context-retainment.spec.js' --config video=false,screenshotOnRunFailure=false

env:

PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}

  

```

  

```yml

## .github/workflows/main.yml

  

# option 2: use an entire new job for visual testing

# preferred if using remote reusable workflows

# insert at the end of your yml file

  

cypress-visual-tests:

needs: [install-dependencies]

runs-on: ubuntu-latest

container: cypress/included:9.4.1

steps:

- uses: actions/checkout@v3

  

- uses: bahmutov/npm-install@v1

with: { useRollingCache: true }

  

- name: Cypress visual tests 🧪

uses: cypress-io/github-action@v3.0.3

with:

install: false

start: yarn dev

wait-on: 'http://localhost:3000'

browser: chrome

command: yarn percy exec -- cypress run --spec 'cypress/integration/ui-integration/user-context-retainment.spec.js' --config video=false,screenshotOnRunFailure=false

env:

PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}

```

  

Here is the new CI job. When it passes, the Percy dashboard looks just the same as running the script locally.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zs0oa5m50najq57pd0lq.png)

  

### **[Pull Request integration](https://docs.percy.io/docs/github)**

  

We want the ultimate developer experience, not having check CI execution details and an external url when snapshots fail. We want to click on a Github link and end up where we need to be. Let's integrate Github and Percy. Click `Add integration` on Percy interface.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o32xr5irlf8ds7riepbc.png)

  

Follow the prompts to install Percy into the Github repo. It routes to Github where we pick repos and permissions that should have Percy access... After that, we need to link the Percy project with a Repository. At the end things look as such:

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wl9fv756e540egq6uv5l.png)

  

At this point do one more push to see the Percy job. Clicking details will take us straight to the Percy interface for this commit. The Percy job will fail when snapshots do not match. If we update the snapshot and approve the variance as the new baseline, the Github job will immediately turn green. Otherwise it will stay failed. Painless developer experience is always an easy sell.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/waiv6u47ezjnavz7n5vg.png)

  

### Further customization

  

#### Custom selector

  

Let's say that we do not want to take a full screen snapshot, but want to focus on a sub area. Percy does not have a feature for this but we can implement a custom command.

  

```js

// cypress/support/commands.js

import '@percy/cypress'

  

...

  

// https://github.com/percy/percy-cypress/issues/56

Cypress.Commands.add(

'percySnapshotElement',

{ prevSubject: true },

(subject, name, options) => {

cy.percySnapshot(name, {

domTransformation: (documentClone) =>

scope(documentClone, subject.selector),

...options

})

}

)

  

function scope(documentClone, selector) {

const element = documentClone.querySelector(selector)

documentClone.querySelector('body').innerHTML = element.outerHTML

  

return documentClone

}

```

  

Then we can modify our spec. Here we just added a 2nd snapshot:

  

```js

// cypress/integration/ui-integration/user-context-retainment.spec.js

  

it('Should keep the user context between routes', () => {

cy.fixture('users').then((users) => {

cy.get('.user-picker').select(users[3].name)

cy.contains('Users').click()

  

cy.wait('@userStub')

cy.url().should('contain', '/users')

cy.get('.item-header').contains(users[3].name)

  

// the full-page visual test

cy.percySnapshot('User selection retainment between routes')

  

// using custom command for css selector focus

cy.getByCy('user-details').percySnapshotElement(

'user details with custom selector'

)

})

})

```

  

#### Custom viewports and browsers

  

By default 2 viewports (375px & 1280px) are being run on 4 browsers, including Safari. This costs 8 snapshots per run. Browser toggling is done from [Project Configuration Settings](https://docs.percy.io/docs/cross-browser-visual-testing#enabling-and-disabling-browsers), but we need paid version for that feature.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/667boy0fx6ud0dgl5qts.png)

  

Percy has [a few configuration options](https://docs.percy.io/docs/cli-configuration#a-complete-config). We prefer a yaml file for viewport configs. Create a file called `.percy.yml` at repo root:

  

```yml

## .percy.yml

version: 2

snapshot:

widths: [750, 1024, 1920]

```

  

This way each snapshot will be on 3 different viewports.

  

We can confirm how things look on different browsers and viewports and approve this change. In this case each snapshot will assert 4 x 3 checks, and the spec will have 24 checks because we take 2 snapshots.

  

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zqgtfe8d74u696itaevj.png)

  

That wraps up all we have to know about visual testing.