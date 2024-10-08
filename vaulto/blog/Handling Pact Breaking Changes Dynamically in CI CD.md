# Handling Pact Breaking Changes Dynamically in CI/CD

When working with consumer-driven contract testing using [Pact](https://docs.pact.io/), ensuring compatibility between provider and consumer services can be challenging, especially when introducing breaking changes. Providers often need to make changes that might not be immediately compatible with consumers, and coordinating these changes can be a hassle. To streamline this process, we can leverage environment variables and GitHub Actions to dynamically handle breaking changes in Pact tests with minimal developer intervention.

In this blog post, I'll walk you through a method to manage breaking changes in Pact verification tests, ensuring a smoother development workflow and reducing the friction often associated with such changes.

You can find the working code and changes in [this PR](https://github.com/muratkeremozcan/pact-js-example-provider/pull/54).

------

## The Problem: Breaking Changes in Pact Verification

By default, when verifying consumer contracts, we might test against multiple consumer versions:

- **`matchingBranch`**: Verifies against feature branches that match the provider's branch, facilitating coordinated development.
- **`mainBranch`**: Verifies against the consumer’s `main` branch, which should be stable.
- **`deployedOrReleased`**: Verifies against consumer versions that are currently deployed or released into production.

However, when a provider introduces a breaking change, these default checks can cause unnecessary failures. Developers want to focus on verifying against the `matchingBranch`, where compatible changes are expected, without being bogged down by failures from the `mainBranch` or `deployedOrReleased` versions.

------

## The Solution: Managing Breaking Changes with Environment Variables

To address this, we introduce an environment variable `PACT_BREAKING_CHANGE`. When set to `true`, it modifies the verification process to focus solely on the `matchingBranch`, skipping the checks against `mainBranch` and `deployedOrReleased`. This allows developers to proceed with breaking changes without the CI pipeline failing due to incompatibilities with consumer versions that haven't yet been updated.

### Implementation Details

#### Verifier Options

We adjust the verifier options to include or exclude certain consumer version selectors based on the `PACT_BREAKING_CHANGE` variable:

```typescript
const options: VerifierOptions = {
  providerBaseUrl: `http://127.0.0.1:${port}`,
  provider: 'MoviesAPI',
  publishVerificationResult: true,
  providerVersion,
  providerVersionBranch: providerBranch,
  pactBrokerUrl: process.env.PACT_BROKER_BASE_URL,
  pactBrokerToken: process.env.PACT_BROKER_TOKEN,
  stateHandlers,
  beforeEach,
  afterEach,
  enablePending: PACT_BREAKING_CHANGE === 'true',
};

// Determine which consumer versions to verify against
options.consumerVersionSelectors = [
  { matchingBranch: true },
  ...(includeMainAndDeployed ? [
    { mainBranch: true },
    { deployedOrReleased: true },
  ] : []),
  ...(includeAllPacts ? [{ all: true }] : []),
];
```

- **`includeMainAndDeployed`**: Set to `false` when `PACT_BREAKING_CHANGE` is `'true'`, excluding `mainBranch` and `deployedOrReleased` from verification.
- **`enablePending`**: Enables pending pacts when introducing breaking changes, allowing the provider to accept pending pacts without failing the build.
- **`includeAllPacts`**: When set, includes all pacts in the verification process, useful for broader testing during breaking changes.

#### Building Consumer Version Selectors

We encapsulate the logic for determining which consumer versions to verify in a helper function:

```typescript
import type { ConsumerVersionSelector } from '@pact-foundation/pact-core';

function buildConsumerVersionSelectors(
  consumer: string | undefined,
  includeMainAndDeployed: boolean,
  includeAllPacts: boolean
): ConsumerVersionSelector[] {
  const baseSelector: Partial<ConsumerVersionSelector> = consumer ? { consumer } : {};

  const selectors: ConsumerVersionSelector[] = [
    { ...baseSelector, matchingBranch: true },
    ...(includeMainAndDeployed ? [
      { ...baseSelector, mainBranch: true },
      { ...baseSelector, deployedOrReleased: true },
    ] : []),
    ...(includeAllPacts ? [{ ...baseSelector, all: true }] : []),
  ];

  return selectors;
}
```

## Integrating with CI/CD

To make this process seamless in CI, we use GitHub Actions to set the `PACT_BREAKING_CHANGE` environment variable based on the PR description. Developers can indicate a breaking change by including a checkbox in the PR description. This automates the process, eliminating the need for manual intervention.

### GitHub Actions Workflow

Here's how we set up the workflow:

```yml
name: Run contract tests

on:
  pull_request:
    types: [opened, synchronize, reopened, edited]
  push:
    branches:
      - main

jobs:
  contract-test:
    runs-on: ubuntu-latest
    env:
      PACT_BROKER_BASE_URL: ${{ secrets.PACT_BROKER_BASE_URL }}
      PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
      GITHUB_SHA: ${{ github.sha }}
      GITHUB_REF_NAME: ${{ github.ref_name }}
      DATABASE_URL: 'file:./dev.db'
      PORT: 3001
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref || github.ref_name }}
          fetch-depth: 0

      - name: Read Node version from .nvmrc
        id: node_version
        run: echo "NODE_VERSION=$(cat .nvmrc)" >> $GITHUB_ENV

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      # Set PACT_BREAKING_CHANGE based on PR description during pull_request events
      - name: Set PACT_BREAKING_CHANGE based on PR description (PR event)
        if: ${{ github.event_name == 'pull_request' }}
        uses: actions/github-script@v7
        with:
          script: |
            const prBody = context.payload.pull_request.body || '';
            if (prBody.includes('[x] Pact breaking change')) {
              core.exportVariable('PACT_BREAKING_CHANGE', 'true');
              console.log('PACT_BREAKING_CHANGE set to true based on PR description checkbox.');
            } else {
              core.exportVariable('PACT_BREAKING_CHANGE', 'false');
              console.log('PACT_BREAKING_CHANGE remains false.');
            }

      # Set PACT_BREAKING_CHANGE based on merged PR description during push events
      - name: Set PACT_BREAKING_CHANGE based on merged PR description (push to main)
        if: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
        uses: actions/github-script@v7
        with:
          script: |
            const commitSha = context.sha;
            const { data: prs } = await github.rest.repos.listPullRequestsAssociatedWithCommit({
              owner: context.repo.owner,
              repo: context.repo.repo,
              commit_sha: commitSha,
            });
            const mergedPr = prs.find(pr => pr.merged_at !== null && pr.merge_commit_sha === commitSha);
            if (mergedPr) {
              const prBody = mergedPr.body || '';
              if (prBody.includes('[x] Pact breaking change')) {
                core.exportVariable('PACT_BREAKING_CHANGE', 'true');
                console.log('PACT_BREAKING_CHANGE set to true based on merged PR description checkbox.');
              } else {
                core.exportVariable('PACT_BREAKING_CHANGE', 'false');
                console.log('PACT_BREAKING_CHANGE remains false.');
              }
            } else {
              core.exportVariable('PACT_BREAKING_CHANGE', 'false');
              console.log('No merged PR found for this commit. PACT_BREAKING_CHANGE remains false.');
            }

      - name: Install dependencies
        run: npm ci

      - name: Run provider contract tests
        run: |
          echo "Running provider contract tests with PACT_BREAKING_CHANGE=$PACT_BREAKING_CHANGE"
          npm run test:provider-ci

      - name: Can I deploy provider?
        if: env.PACT_BREAKING_CHANGE == 'false'
        run: npm run can:i:deploy:provider

      - name: Record provider deployment
        if: github.ref == 'refs/heads/main'
        run: npm run record:provider:deployment --env=dev
```

### PR Description Checkbox

Developers can indicate a breaking change by including the following in their PR description:

```markdown
### Pact Breaking Change

- [x] Pact breaking change (check if this PR introduces a breaking change)
```

When the checkbox is checked, the `PACT_BREAKING_CHANGE` variable is set to `true` in the CI environment.

## Handling Verification Failures Due to Breaking Changes

In scenarios where verification failures occur due to breaking changes, we can adjust our test code to handle these failures gracefully. Here's how we modify the test:

```typescript

const PACT_BREAKING_CHANGE = process.env.PACT_BREAKING_CHANGE || 'false'
const GITHUB_BRANCH = process.env.GITHUB_BRANCH || 'local'

describe('Pact Verification', () => {
  const port = process.env.PORT || '3001'
  const options = buildVerifierOptions({
    provider: 'MoviesAPI',
    consumer: process.env.PACT_CONSUMER, // filter by the consumer, or run for all if no env var is provided
    includeMainAndDeployed: PACT_BREAKING_CHANGE !== 'true', // if it is a breaking change, set the env var
    enablePending: PACT_BREAKING_CHANGE === 'true',
    port,
    stateHandlers,
    beforeEach: () => {
      console.log('I run before each test coming from the consumer...')
      return Promise.resolve()
    },
    afterEach: () => {
      console.log('I run after each test coming from the consumer...')
      return Promise.resolve()
    }
  })
  const verifier = new Verifier(options)

  it('should validate the expectations of movie-consumer', async () => {
    try {
      const output = await verifier.verifyProvider()
      console.log('Pact Verification Complete!')
      console.log('Result:', output)
    } catch (error) {
      console.error('Pact Verification Failed:', error)

      if (PACT_BREAKING_CHANGE === 'true' && GITHUB_BRANCH === 'main') {
        console.log(
          'Ignoring Pact verification failures due to breaking change on main branch.'
        )
      } else {
        throw error // Re-throw the error to fail the test
      }
    }
  })
})
```

- **Error Handling**: We wrap the verification in a try-catch block. If verification fails and `PACT_BREAKING_CHANGE` is `'true'` on the `main` branch, we log a message and prevent the test from failing the CI pipeline.
- **Local Development**: When running tests locally, `PACT_BREAKING_CHANGE` defaults to `'false'`, ensuring that tests fail on verification failures, prompting developers to address issues promptly.

## Workflow Summary

This approach allows us to manage breaking changes effectively by:

- **Automating Environment Variable Setting**: Using GitHub Actions to set `PACT_BREAKING_CHANGE` based on the PR description reduces manual steps.
- **Adjusting Verification Behavior**: Modifying the consumer version selectors dynamically based on `PACT_BREAKING_CHANGE` ensures that we only verify relevant consumer versions during breaking changes.
- **Handling Test Failures Gracefully**: By adjusting the test code to conditionally handle verification failures, we prevent unnecessary CI failures while maintaining test integrity.

## Breaking Change Flows

### Consumer Flow

For consumers adapting to breaking changes:

```bash
# 1. Update the consumer tests
npm run test:consumer  # Execute the updated tests
npm run publish:pact    # Publish the updated pact
npm run can:i:deploy:consumer  # Check if it's safe to deploy
# Only on main
npm run record:consumer:deployment --env=dev  # Record the deployment
```

### Provider Flow

For providers introducing breaking changes:

```bash
# 1. Create a branch with the breaking change
PACT_BREAKING_CHANGE=true npm run test:provider-ci  # Run the provider tests with the breaking change flag
# Note: The can:i:deploy:provider step is skipped because we're introducing a breaking change
# 2. Merge to main
```

## Conclusion

By introducing this environment-driven mechanism, we now have a flexible way to manage breaking changes in Pact tests. This approach allows us to:

- Maintain confidence in our provider contracts while rolling out changes incrementally.
- Simplify the developer experience by dynamically handling the changes via an env var and a checkbox in the PR description.

Moreover, the addition of the `buildConsumerVersionSelectors` function significantly DRYs up the code by centralizing the logic for determining which consumer versions to verify. By encapsulating this logic, it not only makes the code more maintainable but also provides a cleaner developer experience when managing consumer filtering and selector behavior.

This approach can be applied to any project using Pact and provides a smooth, automated way to handle breaking changes without manual intervention.

