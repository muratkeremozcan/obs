Recently I was asked about cross browser / device testing solutions such as Browserstack, Sauce Labs or Lambdatest.

Having used one of these tools in the past, and having researched the others, I have formed an opinion on these solutions and the grid approach to cross browser testing.


### There are only 2 main code bases for browsers, and the versions are not a critical factor anymore

Browsers are converging into 2 main choices; Chromium-based and Firefox. All Chromium based browsers behave near identically in testing. The versions are kept up to date automatically and tested rigorously. Our value can be in testing the canary/beta versions of these browsers to gain confidence that things will be fine when browser updates happen.

### The OS does not matter on desktop, mobile browsers boil down to viewport sizes as opposed to device types

There was a time these did matter, especially in desktop development and a while ago in mobile development. In web development, this parameter is not critical, and if it becomes so there are effortless built-in solutions such as [Github Actions build matrix](https://docs.github.com/en/actions/learn-github-actions/managing-complex-workflows#using-a-build-matrix).


### So what is the meta solution today? How do we get away from the grid approach but still have a confidence in our products?

Given that the main parameters of concern today are the browser type and the viewport, the below 3 strategies can be harmonized to gain the maximum value with minimal effort.

1. [Combinatorial Testing](https://github.com/NoriSte/ui-testing-best-practices/blob/master/sections/advanced/combinatorial-testing.md) can be applied to CI, considering browsers, deployments and a subsets of tests. Check out [this slide deck](https://cypress.slides.com/cypress-io/siemens-case-study#/16) (scroll down with down arrow), it showcases how we applied theoretical math to reduce 72 CI test config combinations to 5.

2. [Combined  (unit+ e2e) code coverage](https://dev.to/muratkeremozcan/combined-unit-e2e-code-coverage-case-study-on-a-real-life-system-using-angular-jest-cypress-gitlab-35nk) can be utilized to decide on a lean subset of tests that covers the main workflows and at the same time hits a good percentage of the source code.

3. Visual AI testing can be used to not only cover browser & viewport concerns but also ensure that there are no visual and/or CSS regressions in the app. [In this example of a run](https://cypress.slides.com/cypress-io/siemens-case-study#/6/2) with Percy.io, 1 test covers 2 browsers x  3 viewports.