

Start the service `localhost:3000`

Run e2e tests against it

At root: `yarn start:open-incredibot`

At service folder:  `yarn start:open`

Implications:

* No need to deploy to a sandbox to test the changes
* We can get code coverage from e2e tests, and combine it with unit tests

In CI, the same happens

-   [runs against localhost](https://cloud.cypress.io/projects/n6iz4z/runs/4788/test-results/3a35b85a-3a37-4188-8d0e-861cd925fdc9/screenshots)
-   [good dashboard results](https://cloud.cypress.io/projects/n6iz4z/runs/4788/specs)

Implications

* No need for sandboxes in PRs
* No shared mutable state
* No way to test first & deploy later(!)

-   [Docs](https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1552023683/How+to+Run+Lambdas+Functions+Locally)
*   [node-core PR](https://github.com/helloextend/node-core/pull/13073) , [GHA](https://github.com/helloextend/node-core/pull/13073/)