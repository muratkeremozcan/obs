-   [Custom baseUrl in commit message](https://github.com/helloextend/gha-reusable-workflows/pull/13)
    -   problem: we were not able to receive the baseUrl from the CI environment
    -   what we already have
        -   [target a deployment: built-in to monorepos](https://github.com/helloextend/node-core/blob/master/services/audit/package.json#L28)  
            yarn cy:open-foo-sandbox / dev / stage
        -   cypress.json file
        -   [test suite trigger jobs](https://github.com/helloextend/node-core/actions/workflows/incredibot-trigger-e2e-suite.yml)
    -   [what we didn't have: baseUrl in a PR](https://github.com/helloextend/node-core/blob/master/services/audit/cypress/config/sandbox.json). 1 service: 1 sandbox
        -   easy through CLI, if we could get the value from somewhere  
            yarn cy:open --config baseUrl=[https://api-acceptance.helloextend.com](https://api-acceptance.helloextend.com/)  
        -   [we can get it through the commit message](https://github.com/helloextend/gha-reusable-workflows/pull/13/files)  
            git commit -m "[JIRA_TICKET] hello world baseUrl=[https://somewhere.com](https://somewhere.com/)"
            -   [Example run with default configuration,](https://github.com/helloextend/test-package-consumer/runs/6578404742?check_suite_focus=true) 
            -   [Example run with custom baseUrl as stage environment](https://github.com/helloextend/test-package-consumer/runs/6578114626?check_suite_focus=true)


-   Conditional branch protection for the monorepos
    -   problem: cost savings measures -> e2e only trigger with relative changes, BUT [Github does not support this conditional logic](https://github.com/helloextend/node-core/settings/branch_protection_rules/5992355)
    -   Hypothesis driven development, working experimentally
    -   Implementation details
        -   we can use a custom GHA
            -   [If files changed, set a var](https://github.com/helloextend/node-core/blob/master/.github/workflows/stores-e2e.yml#L22) _[filesChanged](https://github.com/helloextend/node-core/blob/master/.github/workflows/stores-e2e.yml#L22)_
            -   [run-tests, only if filesChanged](https://github.com/helloextend/node-core/blob/master/.github/workflows/stores-e2e.yml#L44)
        -   [we can feed the result as an input to the Reusable WF](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/mono-service-e2e.yml#L14)
        -   [we only run e2e if the input is true](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/mono-service-e2e.yml#L88)
        -   [we set success if e2e tests worked](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/mono-service-e2e.yml#L247)
        -   [we have a final check to use in branch protection](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/mono-service-e2e.yml#L260)


-   [LaunchDarkly + Cypress e2e testing on deployed services](https://github.com/helloextend/node-core/pull/10019)
    -   Problem: Feature Flags change the app. How do we test them in various deployments? How do we increase FF usage with new features?
    -   [Multiple apps are piloting on client](https://dev.to/muratkeremozcan/effective-test-strategies-for-testing-front-end-applications-using-launchdarkly-feature-flags-and-cypress-part1-the-setup-jfp), now [node-core can too](https://dev.to/muratkeremozcan/effective-test-strategies-for-deployed-nodejs-services-using-launchdarkly-feature-flags-part1-the-setup-21ji)
    -   Front-end app test strategies
        -   stub the all LaunchDarkly calls
        -   controlled flag
    -   Deployed service test strategies (problem: no network stubbing!)
        -   [conditional execution](https://github.com/helloextend/cypress-ld-ff#conditional-execution-in-a-service-test): check flag status, run accordingly; allows to execute the same suite regardless of flag settings or environment
        -   [controlled flag](https://github.com/helloextend/cypress-ld-ff#flag-control-in-a-service): utilizes targeted users to achieve stateless tests decoupled from flag controls.
    -   [Extend LaunchDarkly Feature Flag utilities for Cypress](https://github.com/helloextend/cypress-ld-ff)
    -   [Contracts PR](https://github.com/helloextend/node-core/pull/10019)