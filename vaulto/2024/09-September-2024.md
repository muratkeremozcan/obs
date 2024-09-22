# July 2024

## Week of 09-02-2025

- [Strengthening Pact Contract Testing with TypeScript and Data Abstraction](https://dev.to/muratkeremozcan/-strengthening-pact-contract-testing-with-typescript-and-data-abstraction-16hc) @blog
- [resolved the trigger e2e problem with pnpm at Portal](https://github.com/helloextend/gha-reusable-workflows/pull/793) @devops
- [Resolved it better](https://github.com/helloextend/portal-web-app/pull/186) @devops
- [a pattern to fine tune cy task when handling multiple arguments](https://www.youtube.com/watch?v=R_3lRSB3Zd8) @blog
- [Made vite config more like Mike's fast config but it's still slow](https://github.com/helloextend/portal-web-app/pull/184) @testing

## Week of 09-09-2025

* [Mike made vite config fast](https://github.com/helloextend/portal-web-app/pull/187#pullrequestreview-2290465046)) @testing
* [enabled claims ui-e2e tests](https://github.com/helloextend/portal-web-app/pull/194) @testing
* [enabled authV3 on Portal]([https://github.com/helloextend/portal-web-app/pull/198](https://github.com/helloextend/portal-web-app/pull/198)) @testing
* [no more getByClass at Portal](https://github.com/helloextend/portal-web-app/pull/209) @testing
* [address circular dependencies for vite](https://github.com/helloextend/portal-web-app/pull/212) @testing
* [portal lint rule for circular deps and type export/import as types](https://github.com/helloextend/portal-web-app/pull/213) @testing
* [book: consumer driven contract testing](https://www.manning.com/books/contract-testing-in-action) @learning
* [go through pact docs](https://docs.pact.io/implementation_guides/javascript/docs/messages) @learning
* [a pattern to fine tune cy task when handling multiple arguments](https://www.youtube.com/watch?v=R_3lRSB3Zd8) @learning
* [Schema validation using cypress-ajv-schema-validator vs Optic](https://www.youtube.com/watch?v=ysCADOh9aJU) @blog
* cypress-recurse and spok combo
  
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gqehrdz3te1sbv1okh14.png)


## Week of 09-16-2025
* [pact videos](https://www.youtube.com/playlist?list=PLwy9Bnco-IpfZ72VQ7hce8GicVZs7nm0i) @learning
* [pact slides and workshop](https://docs.pactflow.io/docs/workshops/introduction) @learning
* [Advanced contract testing with Pact](https://docs.pactflow.io/docs/workshops/advanced) @learning
* [pact CI/CD workshop `enablePending`  `includeWipPactsSince`](https://docs.pactflow.io/docs/workshops/ci-cd/set-up-ci/configure-consumer-and-provider-pipelines) @learning
* [Handling Pact Breaking Changes Dynamically in CI/CD](https://dev.to/muratkeremozcan/handling-pact-breaking-changes-dynamically-in-cicd-3dl4) @blog
* [buildVerifierOptions utility at the provider](https://github.com/muratkeremozcan/pact-js-example-provider/pull/57) @learning
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jsbj6w5uifmmimxj3mcq.png)
* [Portal - used start:preview in CI, switched start to vite](https://github.com/helloextend/portal-web-app/pull/215) @testing
* [Portal - switched build script to vite](https://github.com/helloextend/portal-web-app/pull/217) @testing
* [BEST - removed optic:verify]([https://github.com/helloextend/backend-service-template/pull/1486](https://github.com/helloextend/backend-service-template/pull/1486)) @testing
* [reverts the portal revert w/o changing the build scripts, sets vite.config.ts define better]https://github.com/helloextend/portal-web-app/pull/223
```json
define: {

'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV || 'development'),

'process.env.VITE': JSON.stringify('true'),

'process.env.NODE_DEBUG': JSON.stringify(process.env.NODE_DEBUG || ''),
```


## Week of 09-23-2025