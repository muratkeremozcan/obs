
# GHA Remote reusable workflows
---
### [Engineering Demo 2-13](https://github.com/helloextend/weekly-logs/blob/main/2022/02-Feb-2022.md#week-of-2022-02-13) [video recording](https://drive.google.com/file/d/183ALaAIDuzo9xU6o_UcK1pDPwl_VZNCr/view?usp=drive_web)
#### Reasons for test failures   
 
   -   _test flake_ _(generally close to 0%)_  *[analytics](https://dashboard.cypress.io/projects/t4r241/analytics/flaky-tests?branches=%5B%5D&browsers=%5B%5D&chartRangeMostCommonErrors=%5B%5D&chartRangeSlowestTests=%5B%5D&chartRangeTopFailures=%5B%5D&committers=%5B%5D&cypressVersions=%5B%5D&flaky=%5B%5D&operatingSystems=%5B%5D&runGroups=%5B%5D&specFiles=%5B%5D&status=%5B%7B%22label%22%3A%22Passed%22%2C%22value%22%3A%22PASSED%22%7D%2C%7B%22label%22%3A%22Failed%22%2C%22value%22%3A%22FAILED%22%7D%5D&tags=%5B%7B%22disabled%22%3Afalse%2C%22label%22%3A%22sandbox%22%2C%22labelProperties%22%3A%7B%22color%22%3A%22%23dadade%22%7D%2C%22suggested%22%3Afalse%2C%22value%22%3A%22sandbox%22%7D%5D&timeInterval=WEEK&timeRange=%7B%22startDate%22%3A%222022-03-11%22%2C%22endDate%22%3A%222022-03-18%22%7D&viewBy=TEST_CASE)*
    -   _top failures_ *[analytics](https://dashboard.cypress.io/projects/t4r241/analytics/top-failures?branches=%5B%5D&browsers=%5B%5D&chartRangeMostCommonErrors=%5B%5D&chartRangeSlowestTests=%5B%5D&chartRangeTopFailures=%5B%5D&committers=%5B%5D&cypressVersions=%5B%5D&flaky=%5B%5D&operatingSystems=%5B%5D&runGroups=%5B%5D&specFiles=%5B%5D&status=%5B%7B%22label%22%3A%22Passed%22%2C%22value%22%3A%22PASSED%22%7D%2C%7B%22label%22%3A%22Failed%22%2C%22value%22%3A%22FAILED%22%7D%5D&tags=%5B%7B%22disabled%22%3Afalse%2C%22label%22%3A%22sandbox%22%2C%22labelProperties%22%3A%7B%22color%22%3A%22%23dadade%22%7D%2C%22suggested%22%3Afalse%2C%22value%22%3A%22sandbox%22%7D%5D&timeInterval=WEEK&timeRange=%7B%22startDate%22%3A%222022-03-11%22%2C%22endDate%22%3A%222022-03-18%22%7D&viewBy=TEST_CASE)*
        -   real Service/App failure
        -   environment instability
        -   system dependency failure  

---
### The 3 GitHub Action (GHA) workflows for e2e 
 * **regular e2e**:  *shifted left, runs on feature branches and deployments* [example](https://github.com/helloextend/node-core/actions/workflows/auth-e2e.yml)
 * **test burn-in**: *make your tests unbreakable* [example](https://github.com/helloextend/node-core/actions/workflows/auth-repeat-title.yml)
 * **trigger jobs**: *poor man's CD, or troubleshoot Service X on Sandbox Y* [example](https://github.com/helloextend/node-core/actions/workflows/auth-trigger-e2e-suite.yml)

---
### Save yml duplication between similar entities
* [client reusable workflows (**regular e2e PR**)](https://github.com/helloextend/client/pull/3419): *~150 lines of yml saved per workflow, per app/service*
* [remote reusable worfklow repo](https://github.com/helloextend/gha-reusable-workflows)

---
#### Client (5 apps)
* [client repo - yml overview](https://github.com/helloextend/client/tree/main/.github)
* [client repo - Actions](https://github.com/helloextend/client/actions/workflows/customers-e2e-deployment.yml) 

---
#### Node-core (20 services)
* [node-core repo - file level overview](https://github.com/helloextend/node-core/tree/master/.github/workflows)
* [node-core repo - Actions](https://github.com/helloextend/node-core/actions/workflows/big-commerce-e2e.yml)

---
#### Test plugins (6 plugins)
* [cypress-product Actions](https://github.com/helloextend/cypress-product/actions) vs [cypress-claim Actions](https://github.com/helloextend/cypress-claim/actions)
* [cypress-product yml](https://github.com/helloextend/cypress-product/tree/main/.github/workflows) vs [cypress-claim yml](https://github.com/helloextend/cypress-claim/tree/main/.github/workflows)

---

### How does it work?
![[Pasted image 20220318125231.png]]