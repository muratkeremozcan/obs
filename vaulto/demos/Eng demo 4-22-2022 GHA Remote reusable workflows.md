
# GHA Remote reusable workflows
---

* *Reduce CI setup duplication and abstract it away to a central repository*
* *Prevent trial & error grind*
* *Configure new CI effortlessly*

---
### The 3 GitHub Action (GHA) workflows for e2e 
 * **regular e2e**:  *shifted left, runs on feature branches and deployments* [example](https://github.com/helloextend/node-core/actions/workflows/auth-e2e.yml)
 * **test burn-in**: *make your tests unbreakable* [example](https://github.com/helloextend/node-core/actions/workflows/auth-repeat-title.yml)
 * **trigger jobs**: *poor man's CD, troubleshoot Service X on Sandbox Y, burn-in a full test suite for test-quality check * [example](https://github.com/helloextend/node-core/actions/workflows/auth-trigger-e2e-suite.yml)

---
### Save yml duplication between similar entities
* [client reusable workflows (**regular e2e PR**)](https://github.com/helloextend/client/pull/3419): *~150 lines of yml saved per workflow, per app/service*
* [remote reusable worfklow repo](https://github.com/helloextend/gha-reusable-workflows)

---
#### Node-core (20 services)
* [node-core repo - file level overview](https://github.com/helloextend/node-core/tree/master/.github/workflows)
* [node-core repo - Actions](https://github.com/helloextend/node-core/actions/workflows/big-commerce-e2e.yml)

---
#### Client (5 apps)
* [client repo - yml overview](https://github.com/helloextend/client/tree/main/.github)
* [client repo - Actions](https://github.com/helloextend/client/actions/workflows/customers-e2e-deployment.yml) 

---
#### Test plugins (6 plugins)
* [cypress-product Actions](https://github.com/helloextend/cypress-product/actions) vs [cypress-claim Actions](https://github.com/helloextend/cypress-claim/actions)
* [cypress-product yml](https://github.com/helloextend/cypress-product/tree/main/.github/workflows) vs [cypress-claim yml](https://github.com/helloextend/cypress-claim/tree/main/.github/workflows)

---

### How does it work?
![[Pasted image 20220318125231.png]]

---
### Reference
[cypress-grep, single repo, monorepo, re-usable workflows](https://www.youtube.com/watch?v=m03ru99eBuc&t=926s)
