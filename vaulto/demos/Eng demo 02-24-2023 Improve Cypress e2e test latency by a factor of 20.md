#### Improve Cypress e2e test latency by a factor of 20
______
#### ToC

![[Pasted image 20230224090125.png]]

____

#### Problem: 
Each e2e test takes 20 seconds to bundle before execution can start

______

#### Solution effort 1 - leaner module and test plugin imports

-   [faker-js/faker -> @faker-js/faker/locale/en](https://github.com/helloextend/client/pull/5565) at client - still desired at node-core

-   [Cy plugin & config optimization - Portal](https://github.com/helloextend/client/pull/5543), [Merchants](https://github.com/helloextend/client/pull/5651), Customers - still desired everywhere else 

-   [Portal perf optimization part1](https://github.com/helloextend/client/pull/5548)

-   [Portal housekeeping: 1100 loc -> many files](https://github.com/helloextend/client/pull/5612)

-   [Merchants test latency optimization](https://github.com/helloextend/client/pull/5681)

_________

#### Results:

x2 gains; each e2e test takes ~10 seconds to bundle before execution can start

_______

### Solution effort 2 - use an alternative bundler for e2e tests

-   Lot's of discussion & collaboration with Cypress under issue [#25533](https://github.com/cypress-io/cypress/issues/25533)

-   [[DEVXTEST-1616] use esbuild for cy e2e test latency](https://github.com/helloextend/client/pull/5770)
    -   [Same PR for Merchants](https://github.com/helloextend/client/pull/5784)     
    -   [Same PR for Customers](https://github.com/helloextend/client/pull/5789)

___________

#### Results:

Esbuild preprocessor gives us 10x test latency improvement

_______
Local developer experience (videos):  

|       | plugin opt. | esbuild | test latency        |  
| ----- | ------------------- | -------------------- | ------------------------------- |  
| AppA | none                | yes                  | 20sec -> 2 sec, 10x improvement |  
| AppB | yes                 | none                 | 20sec -> 10 sec, 2x improvement |  
| AppC | yes                 | yes                  | 20sec -> 1 sec, 20x improvement |

_________
CI [before](https://github.com/helloextend/client/actions/runs/4207873450) [after](https://github.com/helloextend/client/actions/runs/4235371132), Cy cloud [before](https://cloud.cypress.io/projects/r5mjf5/runs/6702/specs) [after](https://cloud.cypress.io/projects/r5mjf5/runs/6716/specs?utm_source=github)

Conservative estimate: per CI run we are saving at least 20% feedback time and cost in CI minutes.

____
Time & $ ?

 * Conservative estimate: over 100k e2e workflow runs per year ([client workflows](https://github.com/helloextend/client/actions/), [node-core workflows](https://github.com/helloextend/node-core/actions))

 * Over 100 days of engineering time saved per year, 1000 days of CI minutes

_____

### What do I do now?

Choice 1: wait ~6 months and track issue [#25928](https://github.com/cypress-io/cypress/issues/25928)

![image-20230224082406304](file:///Users/murat/Library/Application%20Support/typora-user-images/image-20230224082406304.png?lastModify=1677250611)

_________

Choice 2:

-   Take 6 minutes to read the blog post [Improve Cypress e2e test latency by a factor of 20!!](https://dev.to/muratkeremozcan/improve-cypress-e2e-test-latency-by-a-factor-of-20-34ce)

-   Copy pasta the [PR for Customers](https://github.com/helloextend/client/pull/5789)

	* Estimate is 1 hour per app/service folder, including CI execution

	* Revert / opt out any time by commenting [out 1 line](https://github.com/helloextend/client/pull/5789/files#diff-d1336ebdd6377d5539a38b7ec507f9b1d233f24001c652e9ea267a14b122107fR20) (per deployment)