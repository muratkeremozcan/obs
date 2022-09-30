In the previous post we covered triple combined coverage in a React app written in JS . Alas, Typescript can be tricky with combined code coverage. We continue the series with a Typescript example using the React TS app featured in the book [**CCTDD: Cypress Component Test Driven Design**](https://app.gitbook.com/s/jK1ARkRsP5OQ6ygMk6el/). The application built in the book is in TS, includes Cypress e2e, CT tests, as well as React Testing Library mirrors of them. The repo [tour-of-heroes-react-cypress-ts](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts) has the final version of the repo with triple combined coverage setup. The state of the repo prior to code coverage is in the branch `before-code-coverage` and there is a sample [PR](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/pull/82/files) for reproducing this guide.

## Setup Cypress Component & E2e coverage

### Add the packages

With TS the packages are slightly different:

yarn add -D @babel/plugin-transform-modules-commonjs @babel/preset-env @babel/preset-react @babel/preset-typescript @bahmutov/cypress-code-coverage @cypress/instrument-cra  babel-loader istanbul istanbul-lib-coverage nyc

### Instrument the app for E2e

This script is the same as the react version

"start": "react-scripts -r @cypress/instrument-cra start"

### Configure `nyc` for local coverage evaluation

Add a`.nycrc` file for config. We are setting coverage report directory as `coverage-cy` to isolate it from Jest. `all` property instruments even the files not touched by tests. `excludeAfterRemap` is set to true, per the [Cypress code coverage package docs](https://github.com/cypress-io/code-coverage#exclude-files-and-folders), to not let any excluded files through. Here's a quick [reference to nyc docs](https://github.com/istanbuljs/nyc#common-configuration-options).

{  
  "excludeAfterRemap": true,  
  "report-dir": "coverage-cy",  
  "reporter": ["text", "json", "html"],  
  "extension": [".ts", ".tsx", "js", "jsx"],  
  "include": ["src/**/*.tsx", "src/**/*.ts", "src/**/*.jsx", "src/**/*.js"],  
  "exclude": [  
    "src/setupTests.ts",  
    "src/**/*.test.tsx",  
    "src/**/*.cy.ts",  
    "src/**/*.cy.tsx",  
    "src/**/*.d.ts",  
    "src/reportWebVitals.ts"  
  ]  
}

### Configure `cypress.config.js` for code coverage, instrument the app for component testing

There are 3 key differences on the TS version of the configuration. In the beginning of the file we have to import `@cypress/instrument-cra`. We need to include `@babel/preset-typescript` in the module presets, and the test property has to be TS instead of JS.

import '@cypress/instrument-cra'  
import {defineConfig} from 'cypress'  
const codeCoverageTask = require('@bahmutov/cypress-code-coverage/plugin')  
​  
module.exports = defineConfig({  
  projectId: '7mypio',  
  experimentalSingleTabRunMode: true,  
  retries: {  
    runMode: 2,  
    openMode: 0,  
  },  
  env: {  
    API_URL: 'http://localhost:4000/api',  
  },  
  e2e: {  
    specPattern: 'cypress/e2e/**/*.cy.{js,jsx,ts,tsx}',  
    baseUrl: 'http://localhost:3000',  
    setupNodeEvents(on, config) {  
      return Object.assign({}, config, codeCoverageTask(on, config))  
    },  
  },  
​  
  component: {  
    setupNodeEvents(on, config) {  
      return Object.assign({}, config, codeCoverageTask(on, config))  
    },  
    specPattern: 'src/**/*.cy.{js,jsx,ts,tsx}',  
    devServer: {  
      framework: 'create-react-app',  
      bundler: 'webpack',  
      // here are the additional settings from Gleb's instructions  
      webpackConfig: {  
        // workaround to react-scripts 5 issue https://github.com/cypress-io/cypress/issues/22762  
        devServer: {  
          port: 3001,  
        },  
        mode: 'development',  
        devtool: false,  
        module: {  
          rules: [  
            // application and Cypress files are bundled like React components  
            // and instrumented using the babel-plugin-istanbul  
            {  
              test: /\.ts$/,  
              exclude: /node_modules/,  
              use: {  
                loader: 'babel-loader',  
                options: {  
                  presets: [  
                    '@babel/preset-env',  
                    '@babel/preset-react',  
                    '@babel/preset-typescript',  
                  ],  
                  plugins: [  
                    'istanbul',  
                    ['@babel/plugin-transform-modules-commonjs', {loose: true}],  
                  ],  
                },  
              },  
            },  
          ],  
        },  
      },  
    },  
  },  
})  
​  
/* eslint-disable @typescript-eslint/no-unused-vars */

> This is our preferred component test config because `@babel/plugin-transform-modules-commonjs` lets us customize webpack config, so that we can make all imports accessible from any file including specs. This allows to spy/stub a wider range of code in the component tests.

### Configure both `cypress/support/e2e.ts` and `cypress/support/component.ts`

Indifferent to a JS app.

import "@bahmutov/cypress-code-coverage/support";

### Add coverage convenience scripts to package.json

Same convenience scripts as in the JS version

"cov:combined": "yarn copy:reports && yarn combine:reports && yarn finalize:combined-report",  
"copy:reports": "(mkdir reports || true) && cp coverage-cy/coverage-final.json reports/from-cypress.json && cp coverage/coverage-final.json reports/from-jest.json",  
"combine:reports": "(mkdir .nyc_output || true) && yarn nyc merge reports && mv coverage.json .nyc_output/out.json",  
"finalize:combined-report": "yarn nyc report --reporter html --reporter text --reporter json-summary --report-dir combined-coverage",  
"cov:reset": "rm -rf .nyc_output && rm -rf reports && rm -rf coverage && rm -rf coverage-cy && rm -rf combined-coverage",

### In the TS version there are no differences in local combined coverage, CodeCov service, Github Actions setup

We can replicate the same steps from the JS variant of the guide. Remember to add the CodeCov secret to the repository. Here is the `codecov.yml` file:

codecov:  
  notify:  
    after_n_builds: 3  
coverage:  
  status:  
    project:  
      default:  
        target: auto  
        # this allows a 1% drop from the previous base commit coverage  
        threshold: 2%  
  # makes it so that unit, cy ct and cy e2e reports finish running before the report is shown in a PAR  
  # https://docs.codecov.com/docs/notifications#preventing-notifications-until-after-n-builds  
  ignore:  
    - 'src/setupTests.ts'  
    - 'src/**/*.test.tsx'  
    - 'src/**/*.cy.ts'  
    - 'src/**/*.cy.tsx'  
    - 'src/**/*.d.ts'  
    - './src/models'  
    - 'src/reportWebVitals.ts'

### `jest` to ignore `cy.ts*` files

We need to tell Jest not to include coverage from `cy.ts*` files. This can be done in the `package.json` script. Remember that we also need to modify the start script to instrument e2e tests.

"start": "react-scripts -r @cypress/instrument-cra start",  
"test:coverage": "yarn test --watchAll=false --collectCoverageFrom=src/**/*.ts* --collectCoverageFrom=!src/**/*.*.ts* --coverage",