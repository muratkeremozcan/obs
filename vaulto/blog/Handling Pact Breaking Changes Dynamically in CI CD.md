### Handling Pact Breaking Changes Dynamically in CI/CD

When working with consumer-driven contract testing using Pact, ensuring compatibility between provider and consumer services can be challenging. Providers often introduce breaking changes, or collaborate with the consumers on the changes needed. Pact verification tests against all versions (including `mainBranch` and `deployedOrReleased`) is not always ideal when a breaking change is being introduced.

To address this, we’ve developed a way to dynamically handle breaking changes in Pact tests by leveraging environment variables and GitHub Actions with minimal developer intervention.

You can find the working code and the changes at the PR https://github.com/muratkeremozcan/pact-js-example-provider/pull/54.

#### The Problem: Breaking Changes in Pact Verification

As a best practice, we execute verification tests against multiple consumer versions:

- `matchingBranch`: Verifies against feature branches that match the provider's branch (e.g., coordinated development across teams).
- `mainBranch`: Verifies against the consumer’s `main` branch, which should be stable.
- `deployedOrReleased`: Verifies against consumer versions that are currently deployed or released into production.

When breaking changes are introduced by a Provider, or through a collaboration with the Consumer, these default checks can cause unnecessary failures. A developer would want to focus on verifying only the `matchingBranch`, where compatible changes are expected.

#### The Solution: Managing Breaking Changes with Environment Variables

We introduce an environment variable `PACT_BREAKING_CHANGE` that, when set to `true`, it disables the checks against `mainBranch` and `deployedOrReleased`. This allows developers to focus solely on the `matchingBranch` to verify changes that are compatible with ongoing development.

Here's the logic:

```ts
const options: VerifierOptions = {
  //...
}

const includeMainAndDeployed = process.env.PACT_BREAKING_CHANGE !== 'true'
const consumer = process.env.CONSUMER // to test for all consumers or selectively

options.consumerVersionSelectors = buildConsumerVersionSelectors(consumer, includeMainAndDeployed)

if (!includeMainAndDeployed) {
  console.log('Skipping mainBranch and deployedOrReleased selectors for breaking changes.')
}

verifier = new Verifier(options)
```

#### Breakdown of `buildConsumerVersionSelectors`

This function dynamically generates the Pact consumer version selectors based on whether the environment variable `PACT_BREAKING_CHANGE` is set.

- **Default Behavior**: When `PACT_BREAKING_CHANGE` is not set, the function includes selectors for `mainBranch` and `deployedOrReleased` alongside `matchingBranch`.
- **Breaking Change Behavior**: When `PACT_BREAKING_CHANGE=true`, the code will only verify against the `matchingBranch`, skipping `mainBranch` and `deployedOrReleased`.

Additionally, if a specific consumer is provided via the `PACT_CONSUMER` variable, the selectors will apply only to that consumer. If the consumer is not specified, the function defaults to running tests for **all consumers**.

Here’s how `buildConsumerVersionSelectors` works:

```ts
import type { ConsumerVersionSelector } from '@pact-foundation/pact-core'

export function buildConsumerVersionSelectors(
  consumer: string | undefined,
  includeMainAndDeployed = true
): ConsumerVersionSelector[] {
  const baseSelector: Partial<ConsumerVersionSelector> = consumer ? { consumer } : {}

  const mainAndDeployed = [
    { ...baseSelector, mainBranch: true },
    { ...baseSelector, deployedOrReleased: true }
  ]

  return [
    { ...baseSelector, matchingBranch: true },
    ...(includeMainAndDeployed ? mainAndDeployed : [])
  ]
}
```

This keeps the code DRY by managing both consumer filtering and selector logic in one place.

#### CI/CD Integration

To make this process seamless in CI, we check the PR description for a checkbox or label that indicates a breaking change. This way, the `PACT_BREAKING_CHANGE` environment variable is automatically set based on the committer’s input, without needing to manually adjust the tests.

```yaml
- name: Set PACT_BREAKING_CHANGE based on PR description
  uses: actions/github-script@v6
  with:
    script: |
      const prBody = context.payload.pull_request.body || '';
      if (prBody.includes('[x] Pact breaking change')) {
        core.exportVariable('PACT_BREAKING_CHANGE', 'true');
        console.log('PACT_BREAKING_CHANGE set to true based on PR description.');
      } else {
        console.log('PACT_BREAKING_CHANGE remains false.');
      }
```

In this example, the workflow dynamically checks for a checkbox in the PR description:

```markdown
### Pact Breaking Change

- [x] Pact breaking change (check if this PR introduces a breaking change)
```

If the box is checked, `PACT_BREAKING_CHANGE` is set to `true` and the tests skip the verification against `mainBranch` and `deployedOrReleased`. If it is unchecked, or does not exist, the flag is false.

#### Summary

By introducing this environment-driven mechanism, we now have a flexible way to manage breaking changes in Pact tests. This approach allows us to:

- Maintain confidence in our provider contracts while rolling out changes incrementally.
- Simplify the developer experience by dynamically handling the changes via an env var and a checkbox in the PR description.

Moreover, the addition of the `buildConsumerVersionSelectors` function significantly DRYs up the code by centralizing the logic for determining which consumer versions to verify. By encapsulating this logic, it not only makes the code more maintainable but also provides a cleaner developer experience when managing consumer filtering and selector behavior.

This approach can be applied to any project using Pact and provides a smooth, automated way to handle breaking changes without manual intervention.

