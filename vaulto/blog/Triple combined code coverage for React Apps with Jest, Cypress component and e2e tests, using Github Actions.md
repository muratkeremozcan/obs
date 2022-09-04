Testing, engineering and the scientific method are all bound together. _"Engineering is about assuming stuff will go wrong and learn from that (then write tests) and predict the ways it might go wrong and defend against that (and write more tests)"_ - says Dave Farley. These are my golden rules in testing:

- It’s always cost vs confidence
- cost = creation + execution + maintenance
- What you can avoid testing is more important than what you are testing

In the front-end testing domain of 2022, we have a few layers of approach to our test strategy. Unit test coverage gives us high confidence at source code level, Jest & React Testing Library are dominant in that space. UI-(component)-integration tests with Cypress can stub out the network using the `intercept` api and cover an interaction of components; whether these are state transitions, feature use cases or application workflows. The idea is to isolate the application from the back-end. With e2e tests we can let the UI client interact with the backend and gain an overall confidence on the entire system. With Cypress 10, we have a new super power called component testing, where we can mount a component itself to a real DOM and interact with it. In our humble opinion, this will shift most testing to a lower level from e2e, without loosing confidence, and to a higher level than Jest, without increasing the cost.

Given we are growing on the shoulders of giants with all these tools that enable comprehensive test strategies, what metrics can we evaluate our confidence with? Coverage is an assessment for the thoroughness or completeness of testing with respect to a model. Our model can be source code coverage, feature coverage, mutation score, combinatorial coverage, non-functional requirement coverage, anything. Although source code coverage is not a be all end all metric to pursue, we cannot deny its popularity and potency. We are used to gaining code coverage from unit tests, what if we could also gain source code coverage from Cypress e2e tests, as well as Cypress component tests?. [We have had combined unit & e2e coverage for a while](https://dev.to/muratkeremozcan/combined-unit-e2e-code-coverage-case-study-on-a-real-life-system-using-angular-jest-cypress-gitlab-35nk) and bringing Cypress component testing to it is new in Cypress 10. Imagine being able to add any kind of testing of your choice for new features, and retain above 95% code coverage effortlessly. Would we need to trace every requirement to every test? How much would we have to worry about the changes we introduce while all tests pass and coverage does not regress? Let's walk through a midsize React app and showcase how to achieve that. As always, a blog is lackluster without code, so the code for this blog can be found [in this repo](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress), and the component test code coverage PR can be found [here](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/pull/168).



## Setup Cypress Component & E2e coverage

### Add the packages

Assuming we have a React app (created with CRA), with Jest & Cypress already in it, we need a few packages and their peer dependencies for Cypress e2e and component test code coverage:

```bash
yarn add -D @bahmutov/cypress-code-coverage istanbul-lib-coverage  @cypress/instrument-cra nyc babel-loader @babel/preset-env @babel/preset-react
```

### Instrument the app for E2e

Modify `package.json`/`scripts`/`start` so that our CRA application instruments the code without ejecting react-scripts.

```json
"start": "react-scripts -r @cypress/instrument-cra start"
```

### Configure `nyc` for local coverage evaluation

Add a`.nycrc` file for config. We are setting coverage report directory as `coverage-cy` to isolate it from Jest. `all` property instruments even the files not touched by tests. `excludeAfterRemap` is set to true, per the [Cypress code coverage package docs](https://github.com/cypress-io/code-coverage#exclude-files-and-folders), to not let any excluded files through. Here's a quick [reference to nyc docs](https://github.com/istanbuljs/nyc#common-configuration-options).

```json
  "all": true,
  "excludeAfterRemap": true,
  "report-dir": "coverage-cy",
  "reporter": ["text", "json", "html"],
  "extension": [".js"],
  "include": "src/**/*.js",
  "exclude": [
    "any files you want excluded"
  ]
```

### Configure `cypress.config.js` for code coverage, instrument the app for component testing

The key enabler here is from Gleb Bahmutov's [Component Code Coverage in Cypress v10](https://glebbahmutov.com/blog/component-code-coverage/) blog post. He went through in detail how to achieve component test code coverage in Cypress 10. He also wrote an enhanced version of the code-coverage plugin with additional fixes. Note that the below is subject to change if the Cypress team enables code coverage for component tests with newer versions of Cypress.

```js
const { defineConfig } = require("cypress");
const codeCoverageTask = require("@bahmutov/cypress-code-coverage/plugin");

module.exports = defineConfig({
  projectId: "your cypress dashboard project id",
  e2e: {
    setupNodeEvents(on, config) {
      // note: in the linked repo, the plugins/index.js was large
      // it did not get migrated, but instead gets imported here
      // the below is how we would do it from scratch
      return Object.assign({}, config, codeCoverageTask(on, config));
    },
    baseUrl: "http://localhost:3000",
    specPattern: "cypress/e2e/**/*.{js,jsx,ts,tsx}",
  },
  component: {
    devServer: {
      framework: "create-react-app",
      bundler: "webpack",
      // here are the additional settings from Gleb's instructions
      webpackConfig: {
        mode: "development",
        devtool: false,
        module: {
          rules: [
            // application and Cypress files are bundled like React components
            // and instrumented using the babel-plugin-istanbul
            {
              test: /\.js$/,
              exclude: /node_modules/,
              use: {
                loader: "babel-loader",
                options: {
                  presets: ["@babel/preset-env", "@babel/preset-react"],
                  plugins: [
                    "istanbul",
                    [
                      "@babel/plugin-transform-modules-commonjs",
                      { loose: true },
                    ],
                  ],
                },
              },
            },
          ],
        },
      },
    },
    setupNodeEvents(on, config) {
      return Object.assign({}, config, codeCoverageTask(on, config));
    },
    specPattern: "src/**/**/*.cy.{js,ts,jsx,tsx}",
  },
});
```

> The above also helps resolve an [open issue](https://github.com/cypress-io/cypress/issues/18662) with stubbing imported modules in Cypress component test runner. `@babel/plugin-transform-modules-commonjs` lets us customize webpack config, so that we can make all imports accessible from any file including specs. [Here's how we applied it](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/pull/175/files) to mock LaunchDarkly feature flags in Cypress component tests.

### Configure both `cypress/support/e2e.js` and `cypress/support/component.js`

```js
import "@bahmutov/cypress-code-coverage/support";
```

### Test the setup

At this point, when we execute e2e or component tests, we should be seeing an after block in the test runner, and we should be seeing `coverage-cy` folder populate.

![code cov in after block](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r83s2z913q04lio3y5vk.png)

> Remember to gitignore coverage related files. You can replicate the sample repo's [.gitignore](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/blob/main/.gitignore).

![coverage folders](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iqwyowlf7dvl4pjlxx3a.png)


### Add coverage convenience scripts to package.json

The main ones we will use here are `cov:reset` to clear out all coverage files, and `cov:combined` to generate the combined report locally after having run unit, Cypress component and Cypress e2e tests. The rest of the scripts compose into each other and may also get used in CI. We will explain them throughly in the next section

```json
"cov:combined": "yarn copy:reports && yarn combine:reports && yarn finalize:combined-report",
"copy:reports": "(mkdir reports || true) && cp coverage-cy/coverage-final.json reports/from-cypress.json && cp coverage/coverage-final.json reports/from-jest.json",
"combine:reports": "(mkdir .nyc_output || true) && yarn nyc merge reports && mv coverage.json .nyc_output/out.json",
"finalize:combined-report": "yarn nyc report --reporter html --reporter text --reporter json-summary --report-dir combined-coverage",
"cov:reset": "rm -rf .nyc_output && rm -rf reports && rm -rf coverage && rm -rf coverage-cy && rm -rf combined-coverage",
```

## Combine the unit, e2e & component test coverage (local machine execution)

We execute the unit test suite with `yarn test` using Jest, and get a report under `coverage` folder. Similarly, we execute e2e tests or component tests with `yarn cy:run-e2e` & `cy:run-ct` then get a report under `coverage-cy` folder. We can execute the component & e2e tests in any order, the coverage combines out of the box in `coverage-cy` folder. This is because `nyc` cannot tell what kind of tests generated the coverage files, and adds to it if there is any additional coverage. It is similar to running a unit test back to back, not getting additional coverage, and then running some different unit tests and getting more coverage. Theoretically this means we could have Jest and Cypress share the `coverage` folder, but we like things a bit more orderly.

Let us run a full local workflow to verify all our coverage numbers.

Reset coverage with `yarn cov:reset`. This deletes all the relevant folders.

Run the component tests with `yarn cy:run-ct`. `coverage-cy` folder gets populated. We used TDD with Cypress component tests while creating the app, and above 88% coverage shows how powerful that is.

![ct coverage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nu6ikl9g14nrzygdozcw.png)

If we execute the e2e tests on top of the CT, or vice versa, the coverage will combine. We can do that later, but for now let's back up the component coverage and reset coverage again with `yarn cov:reset` to see how much source code coverage the e2e tests provide. Run `yarn cy:run-e2e` while there is no pre-existing `coverage-cy` folder and we get a pretty good number at above 78%. We only used e2e tests in this app when component testing wasn't enough, or things we wanted confidence on could not be covered at a low level. The ratio of the component tests to e2e is 80 : 20, and we believe this might become a real life ratio for production apps in the future. For the minuscule number of tests and amount of code written for e2e tests, 78% source code coverage is quite high. This means they can be a good choice when filling in missing source code coverage.

> This repo has tests for Applitools, Percy, and LaunchDarkly. You can ask me for the `.env` file, or simply disable those e2e tests. The e2e coverage will be slightly lower.

![e2e coverage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/45n1x1ggeeivbef1pphx.png)

Having populated `coverage-cy` folder with e2e tests, now we can re-excute component tests to see the combined coverage between E2e and CT. Execute `yarn cy:run-ct` and observe `coverage-cy/lcov/index.html`. 94.5% combined e2e and ct coverage is respectable. There is some redundant source code coverage between the tests, but we know that we wrote the e2e tests to gain confidence on features we could not effectively test with component tests. We did not try to pad the coverage numbers, we did what we should and we are getting the code coverage as a side benefit.

![ct & e2e coverage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f33zqo56jdntpderr7gm.png)

Finally, we can run the Jest unit tests with `yarn test`. This covers a simple sum function, replicating pre-existing suite of unit tests in a production app. `coverage` folder gets generated with an html report at `coverage/lcov/index.html`.

![unit coverage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3rnm7xrg6njtpar9j9q3.png)

Now we have to extract certain files from `coverage` & `coverage-cy` folders and combine them into a single report. All we need is to run `yarn cov:combined`. A `combined-coverage` folder gets generated. We can verify that the files combined nicely from the total number of statements/lines and functions.

![triple combined coverage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/llp06kk9fxgebvbmqrpo.png)

### Imperative walkthrough of the local scripts

* We need the `coverage-final.json` files from the relevant coverage folders inside a new folder called `reports`. Create a temp folder `reports`. We use `|| true` so that there are no errors on repeated script executions: `(mkdir reports || true)`. Save the two `coverage-final.json` files from the 2 folders `coverage` & `coverage-cy`. Rename them so that they do not overwrite each other. `cp coverage-cy/coverage-final.json reports/from-cypress.json && cp coverage/coverage-final.json reports/from-jest.json`.

```json
"copy:reports": "(mkdir reports || true) && cp coverage-cy/coverage-final.json reports/from-cypress.json && cp coverage/coverage-final.json reports/from-jest.json",
```

* Combine the reports using `nyc`. Nyc has a utility to specify the folder location for the reports to be merged. Our coverage files are under `reports` folder. After merging, by default, nyc generates a file named `coverage.json` at project root. We rename it and overwrite the `.nyc/` folder. Note that `.nyc` folder gets populated with `out.json` as we run Cypress e2e or CT tests because Cypress code coverage uses it under the hood. We can overwrite that with combined coverage data without a worry.

```json
"combine:reports": "(mkdir .nyc_output || true) && yarn nyc merge reports && mv coverage.json .nyc_output/out.json",
```

* Finalize the report. `nyc` has a command to generate the report using `yarn nyc report`. It uses the `.nyc_output/out.json` file for this. (Do not confuse this with our temporary `reports` folder we used to combine the reports). We can specify multiple report types and also the output directory. We will save the final report in a folder called `combined-coverage`.

```json
 "finalize:combined-report": "yarn nyc report --reporter html --reporter text --reporter json-summary --report-dir combined-coverage",
```

The `yarn cov:combined` script we used is simply those 3 sub-scripts chained together. Make sure to run some tests first before trying to combine their coverage, otherwise the script will complain that it cannot find the files it is trying to move or copy.

## Combined Coverage in CI

### DYI

In an earlier blog post [Combined Unit & E2E Code Coverage: case study on a real life system using Angular, Jest, Cypress & GitLab / CircleCi](https://dev.to/muratkeremozcan/combined-unit-e2e-code-coverage-case-study-on-a-real-life-system-using-angular-jest-cypress-gitlab-35nk) this topic was covered in detail, showcasing how to combine coverage ourselves. It is a lot of work, and the results depend on the success of open source tools in our project. We could not yet get this to work in React context, and started [a discussion](https://github.com/cypress-io/cypress/discussions/22828) at Cypress forums, because nyc merge reports yields an empty `coverage.json` file while working in CI as opposed to local machine. There is a full repro and even a video walkthrough of the imperative steps, [which are in the yml file](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/blob/main/.github/workflows/main.yml#L216) and run with every PR. If there is new information, the repo and the blog post will be updated

### [CodeCov](https://about.codecov.io/) service

In our honest opinion, this is the way to go. Yes, [it is paid](https://about.codecov.io/codecov-pricing/), but pays off dividends in setup, config, maintenance, insight, analytics, and the plethora of features. Let's go through the setup together.

Login with Github at https://app.codecov.io/login/gh, and we see the repositories that we give CodeCov access to.

![codecov setup repo](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/p7x8r71wjc1qysdbd755.png)

Click on setup repo, and if we are using Github Actions, from here all we need is the token in Step 2.

![codecov repo setup-token](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/utg8q7xte187c9estxhm.png)

Paste that under Github Settings > Secrets, into a variable called `CODECOV_TOKEN`.

![codecov secret](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/45xamirvnzkx5f5hr6oq.png)

Next, we add a nifty [Github action](https://github.com/marketplace/actions/codecov) to the end of our unit, e2e and CT jobs. The directory is `coverage-cy` for E2e and CT, `coverage` for Jest. `flags` property is to differentiate between the 3 kinds of coverage in the PR message. The token is used for the communication between the repo and Codecov, in case we are using a private repo. We are showing a relevant section of one of the jobs, and you can take a look at the full yml [here](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/blob/main/.github/workflows/main.yml).

```yml
cypress-e2e-test:
  steps:
  	#...
  - name: Cypress e2e tests 🧪
  	#...
 	- name: ✅ Upload e2e coverage to Codecov
    	uses: codecov/codecov-action@v3
    	with:
      	directory: coverage-cy/
      	flags: cypress-e2e-coverage
      	token: ${{ secrets.CODECOV_TOKEN }}
```

Almost there. We need `codecov.yml` file in the repo root. We are applying one of the [recipes at CodeCov docs](https://docs.codecov.com/docs/common-recipe-list).

```yml
coverage:
  status:
    project:
      default:
        # auto compares coverage to main branch
        target: auto
        # this allows a 2% drop from the previous base commit coverage
        threshold: 2%
  ignore:
    # you can copy the files & folders from nyc to here
    # the format is the same
# makes it so that unit, cy ct and cy e2e reports finish running before the report is shown
codecov:
  notify:
    after_n_builds: 3
```

Finally, let's have a readme badge too. Under Settings > Badges & Graphs copy the markdown to the top of your readme file.

![badge](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4kag354ms4kwbhoskgem.png)

Later on we will get a badge showing the repo's code coverage on the main branch.

![badge](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5pnqi0p0s02dcmibky2l.png)

After pushing the PR, we can view the analytics at the CodeCov web app. This kind of data is not impossible with local testing, but highly inconvenient because we would have to execute the entire unit, component and e2e suites one by one, and then combine the coverage. Then we would have to dig through the html reports to find the source for the lack of coverage. Codecov's sunburst graph makes the lack of coverage at `BookableEdit.js` (bottom red) easily spottable.  Let's take a look.

![analytics](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bzg6g9chcezs4gcoeged.png)

It looks like if we crud a bookable into a pre-existing group, we might hit this code. It will be hard with a unit or component test, but we can replicate the existing e2e test and not randomize the group name, instead use an existing group. As we ponder how best to address the lack of coverage for these private functions, we can observe the line between layers of the test pyramid becoming obsolete; we need some source code coverage and we add the kind of test that will be easiest and most convenient. The type of test, or the layer of the pyramid matter no more. Test is test, code is code, and the decision is only about cost vs confidence.

![BookableEdit.js missing coverage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jkj3pefbwzum29p6bomt.png)

After a few PRs, we can observe the coverage by the time, and a greener looking sunburst graph. We love this kind of insight, giving us a high level and low level view at the same time.

![improved coverage](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8u79r0iadi38e6md5ov8.png)

In the PR, we see a succinct message with the 3 kinds of coverages, the combined coverage, and coverage diff compared to main. Expect a slight deviation from local `nyc` coverages based on ignored files. Mind that the diff is live updating as different kinds of test coverages run and then combine; wait for all tests to finalize before the Codecov report is in its final state. To have the report update only when all the various tests finalize, [notify after n builds](https://docs.codecov.io/docs/notifications#section-preventing-notifications-until-after-n-builds) feature can be used. You can take a look at the final version of the `codecov.yml` file [here](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/blob/main/codecov.yml#L8), with this property added.

![PR message](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gwn943awbjey84vqe7xq.png)

## Conclusion

A de facto standard in production apps in the front end world is targeting 70% unit test coverage. This used to be considered the sweet spot of cost vs confidence. Cypress component tests change that entire paradigm; they are high confidence, low cost, and from a developer's perspective can immediately replace Storybook. Because of the ability to work with a real DOM and interact with it through a fluid API like a real user, it is natural to want to do more testing, which ends up in higher code coverage.

Many apps utilize Cypress for e2e testing, but it is not exactly the norm to gain source code coverage from e2e tests. With Cypress 10, component tests are in the picture and they can directly replace most our work with unit tests which had to rely on the virtual dom. We literally had to unit test in the dark, not seeing what we are testing. It would be a pity not to measure the source code coverage Cypress component tests can provide. Once we have component test code coverage, adding e2e code coverage is a low hanging fruit. Once we have the two Cypress coverages, merging that into any unit test coverage is low effort as well.

With triple combined coverage, we can combine the coverage of our legacy unit tests suite with our Cypress e2e test suite, and begin to add Cypress component tests for new features. Migrating old unit tests to Cypress component tests is not a requirement, because what used to work already provides code coverage. We can migrate unit tests to Cypress component tests optionally, slowly, or not at all. Combining coverage from all kinds of testing, we are less worried about the type of tests or the pyramid; instead we can take 10k feet view and decide on what kind of testing cost makes more sense for the confidence we need, and we measure the results of that decision. This kind of a decision making process puts any team and org in a good spot for quality. We can make  changes, keep increasing the coverage, and try new approaches to our source code without a worry. Today in the front end testing world 95% and above source code coverage is no more a luxury, but an effortless achievement.