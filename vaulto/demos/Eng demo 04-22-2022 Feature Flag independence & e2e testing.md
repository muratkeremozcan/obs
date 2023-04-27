# Feature Flag independence & e2e testing

---
## 3 efforts towards Cont. Deployment & Delivery
* Automated deployments *- Devops*
* Monorepo -> Polyrepo *- Devex*
* Feature Flag Independence
___
## Why?
To boost feature flag
* Development
* Testing
* Usage
---
## "Usage"?
* [Targeted release](https://docs.launchdarkly.com/guides/best-practices/deployment-strategies/?q=ring+deployments#targeted-releases) - *what we do today*
* [Percentage rollout](https://docs.launchdarkly.com/guides/tutorials/rules-and-targeting#configuring-a-percentage-rollout) - *gradual, uniform users*
* [Ring deployments](https://docs.launchdarkly.com/guides/best-practices/deployment-strategies/?q=ring+deployments#ring-deployment) - *managed risk, groups*
	* [Canary release](https://docs.launchdarkly.com/guides/best-practices/deployment-strategies/?q=ring+deployments#canary-releases) - *evaluate, rep. group*
	* Beta - *close to final, early adopters*
* A/B/n - *do you like this version, or that?*
---
## Development
---
### Don't
```js
// react hook
const flags = useFlags()

// usage

{flags['date-and-week'] && (...)}

// or

{flags[LDFlags.DateAndWeek] && (...)}
```

---
### Prefer
```js
const { 'date-and-week': FF_DATE_AND_WEEK } = useFlags()

// even better

const { [LDFlags.DateAndWeek]: FF_DATE_AND_WEEK } = useFlags()

// usage

{ FF_DATE_AND_WEEK && (...)}
```

---
### Wins
* Know what flags are used in a component
* Know **where** those flags are being used in the component
* Add, maintain, retire flags effortlessly

--- 
## Testing Feature Flags
[`@extend/cypress-ld-ff`](https://github.com/helloextend/cypress-ld-ff)
* Stub the flags
* Control the flags: **stateless, confident, effortless**
 
 #### De-coupled FF testing and usage

---
## Demo
* no flag concerns: `cy.stubFeatureFlags(flagsOn)`
* stubbed flag: `cy.stubFeatureFlags({ [LDFlag.ContractsPage]: false })`
* controlled flag:
```js
setFlagVariation(LDFlag.Shopify20, userId, 0)  

setFlagVariation(LDFlag.Shopify20, userId, 1)

removeUserTarget(LDFlag.Shopify20, userId)
```

---
## References

* [Effective Test Strategies for Front-end Applications using LaunchDarkly Feature Flags and Cypress](https://dev.to/muratkeremozcan/effective-test-strategies-for-testing-front-end-applications-using-launchdarkly-feature-flags-and-cypress-part1-the-setup-jfp) Blog
- [DEVXTEST-944 Merchants feature flag independence (part 1)](https://github.com/helloextend/client/pull/3717) PR
- [DEVXTEST-944 Merchants feature flag e2e testing (part 2)](https://github.com/helloextend/client/pull/3744) PR