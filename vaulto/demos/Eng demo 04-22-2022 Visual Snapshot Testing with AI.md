# Visual Snapshot Testing with AI
---

![[Pasted image 20220422080454.png]]
___
## Why AI?
*"Learn to not bother me for [things like this](https://percy.io/782cfcaf/merchants/builds/17436061/changed/978426878?browser=chrome&browser_ids=22%2C23&subcategories=approved&viewLayout=side-by-side&viewMode=new&width=1920&widths=1366%2C1920)"*

![[Pasted image 20220422081140.png]]

---
## Easy to use
* `cy.percySnapshot('mona-lisa')`
* `if (new === old)`   *accept*
* `else`   *reject* | *set new base line*
---
## Easy to manage
[PR workflow look](https://github.com/helloextend/client/pull/3746)

![[Pasted image 20220422083749.png]]

___
## Local demo
* `yarn start:merchants`

* `yarn percy exec -- -- yarn cy:run-merchants-local -- -- --spec 'cypress/integration/ui-e2e/store-customization.spec.ts' --browser chrome --headed`

---
## Failure demo
chop L46,47 `apps/merchants/src/pages/store/customize/customize.tsx` 

---
### References
* [Painlessly setup Cypress & Percy with Github Actions in minutes](https://dev.to/muratkeremozcan/painlessly-setup-cypress-percy-with-github-actions-in-minutes-1aki) Blog

* [visual testing in client](https://github.com/helloextend/client/pull/3714) PR 

* [visual testing in node-core](https://github.com/helloextend/node-core/pull/9042) PR