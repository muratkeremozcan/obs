## Nested reusable workflows
---
## GitHub MS edition naming 
GitHub CI system:  Github Actions

CI workflow file:  Github Actions

Reusable CI steps:  Github Actions

---
## My Preferred naming
GitHub CI system:  Actions CI

CI workflow file:  Workflow file

Reusable CI steps: Github Actions

**Reusable CI workflows: Reusable workflows**

---
## **Nested** Reusable Workflows

[Recently available](https://docs.github.com/en/actions/using-workflows/reusing-workflows#passing-secrets-to-nested-workflows)

Addresses the only con of RWF; **Nesting**

---

## Reusable Workflows 

![[Screen Shot 2022-11-18 at 10.35.28 AM.png]]

---
## **Nested** Reusable Workflows

![[Screen Shot 2022-11-18 at 11.01.48 AM.png]]

---

## Before & After

-   [mono-service-e2e yml](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/mono-service-e2e.yml#L75)
-   [mono-trigger-e2e-suite yml](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/mono-trigger-e2e-suite.yml#L87)
-   [mono-repeat-title yml](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/mono-repeat-title.yml#L73)

-   [poly-install-test yml](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/poly-service-install-test.yml#L165)
-   [poly-trigger-e2e yml](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/poly-service-trigger-e2e-suite.yml#L73)
-   [poly-burn-in-2e yml](https://github.com/helloextend/gha-reusable-workflows/blob/main/.github/workflows/poly-service-test-burn-in.yml#L62)

---
## What's next

All naming & yml is being
refined *(Christopher)*
Going into the poly service template *(Christopher)*
CI works out of the box 
But, also customizable & versioned
Polyrepo-service-playground (*WIP*)