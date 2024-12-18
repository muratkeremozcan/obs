
* [Code references](https://docs.launchdarkly.com/home/code/github-actions/?q=github) in LD for easier flag cleanup
* Use TTL effectively with data session for faster e2e
* Prog login with Okta, using data session 
* Vite pilot for Component Testing in Zen & Portal 
* CSS modules

--- 

### How to take advantage of [Code references](https://docs.launchdarkly.com/home/code/github-actions/?q=github) in LaunchDarkly for easier flag cleanup

---

* Links LD web interface to source code, for easier tracking & removal of flags

* [Create a yml and run on push](https://github.com/helloextend/client/blob/main/.github/workflows/ld-code-refs.yml), only needs `LD_Project_Key` & `LD_Auth_token`

* Once done, it shows up in [LD web interface > Integrations](https://app.launchdarkly.com/portal/integrations).

____

* For each flag, 2 sources of truth: [Code references](https://app.launchdarkly.com/portal/dev/features/servicing-integration-suite/code-refs) + IDE

* Filter dead flags: [filter LD flags without any code references](https://app.launchdarkly.com/portal/dev/features?codeReferences=false)

* Identify flags for removal

---

When to remove feature flags:

- It's been months
    
- The flag is On in production
    
- The flag's [Insights tab](https://app.launchdarkly.com/portal/production/features/servicers-list-optimizations/insights) shows less traffic

____

How to remove feature flags:

- We remove it from the code first ([example](https://app.launchdarkly.com/portal/dev/features/servicers-list-optimizations/code-refs))
    
- Then, we have to wait until our flag-removal-PR makes its way to Prod
    
- Finally, we can archive the flag

____

### How to use TTL (time to live) effectively with data session for faster e2e

___

* TTL: no more need to clean up Stores, Products, Contracts, Conversations (thanks to Jesse!)

* `data-session`: create them once & reuse everywhere

* TLL & `data-session` are already in test packages

____

Start using common session names with the `maybe` variant (check out [test-package-consumer repo](https://github.com/helloextend/test-package-consumer)):

![image-20230622133449717](file:///Users/murat/Library/Application%20Support/typora-user-images/image-20230622133449717.png?lastModify=1687465217)

____

(demo)

___

Results:

* [TTL tests on test package consumer](https://github.com/helloextend/test-package-consumer/pull/205)
* 10/44 tests use `data-session`, 15% faster locally, 10% faster in CI (7 parallel), 

____

### Programmatic login with Okta, using data session 

___

* Login once, never repeat login.

* [Preserves login state via `data-session`](https://github.com/helloextend/client/blob/main/apps/portal/cypress/support/commands/okta-prog-login.ts#L83) given email+role combo

---

```js
// as admin, with admin email  
cy.progOktaLogin()   
​  
// customize it (similar to scopedToken from cypress-auth)  
cy.progOktaLogin({   
  role: 'superadmin',   
  email: 'your email'  
  url: '/admin/plans'   
})
```

--- 

(demo)

---


### Vite pilot for Component Testing in Zen & Portal 

for 9x CyCT startup speed

___


* Problem: CyCT initial startup time gets slower as we do more `faker` things.

* For e2e, we took care of it with [esbuild preprocessor](https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1633386618/Improve+Cypress+e2e+test+latency+by+a+factor+of+20+Feb+2023) sped up startup by 20x this year

---

at Portal or Zen  
​  
```bash
yarn cy:open-ct # webpack  
yarn cy:open-ct-vite # vite
```

With Vite we have 9x faster initial start

___

(video demo)

---

* [known limitations](https://helloextend.atlassian.net/browse/DEVXTEST-1849): no mocking with `cy.stub` (FF tests), no `crypto` (Okta tests) 
* continuing collaboration with CyCT team

---


### [Css modules](https://github.com/helloextend/client/pull/6577)

* `Object.assign` for CSS, allows to [compose css](https://github.com/helloextend/client/tree/ccb7fbf8fe215fcc12f777a7fb452aed7efe4426/packages/zen/src/components/css-modules-test)
* Addresses 2 (future) problems with styled components - server side rendering & bundle size

---

(demo)
