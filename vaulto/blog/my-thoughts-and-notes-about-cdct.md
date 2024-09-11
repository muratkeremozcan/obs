# My thoughts and notes about Consumer Driven Contract Testing

I've recently been diving deep into the world of contract testing, guided by the excellent resource that is the [Contract Testing in Action](https://www.manning.com/books/contract-testing-in-action) book by Marie Cruz and Lewis Prescott. As someone who has spent time navigating the challenges of modern software development & testing particularly in environments that rely heavily on microservices and API-driven architectures, this book has provided invaluable insights.

This post encapsulates my notes, interpretations, reflections, and thoughts on how the principles from the book align with my own experiences, and how they can be effectively integrated into a real-world testing and development workflow.

## Contract Testing

Contract testing ensures that two systems (e.g., a web application and an API, or 2 APIs) have a shared understanding of expectations, verified through a "contract", usually a json file that captures the interactions between the systems. It's particularly useful in microservices architectures.

**How Contract Testing Works:**

- The consumer defines the expected interactions and stores them in a contract. During this process the consumer creates and tests against a mock provider, generating a contract. They publish the contract to the PactBroker.

- The provider (e.g., an API) verifies that it can meet these expectations by running tests against the contract.

- Both sides interact through this contract, ensuring compatibility before deploying changes. The contract is what binds the systems together, without needing to be on the same machine or deployment.

**Why Contract Testing Matters:**

- You can test it up front, even when you're doing local development of the client or the service; fast feedback.
- You can deploy changes independently and with confidence.
- Reduces some of the need for comprehensive end-to-end tests.

**How to start**

- Identify consumers and providers.
- Define the contract: establish clear expectations between consumer and provider.
- Write tests: implement basic consumer and provider contract tests.
- Integrate into CI/CD: add the tests to pipelines and continuously improve the process.

**Vocabulary:**

- **Consumer:** The service user that makes requests to another service, driving the contract in CDCT.
- **Provider:** The service that responds to the consumer's requests, verifying the contract against its implementation.
- **Contract:** A JSON object detailing the interactions between the consumer and provider, including request details and expected responses.
- **Contract Broker:** A central location for storing and managing contracts, facilitating communication between consumers and providers.

**Consumer Driven Contract Testing Lifecycle (CDCT):**

- generating a contract from the consumer's test code
- publishing it to a broker
- having the provider pull, verify, and update the contract’s status
- Both sides then check the verification status before deploying changes (can-i-deploy).

**Contract Testing Approaches:**

- **Consumer-Driven Contract Testing (CDCT):** Focuses on the consumer creating the contract, ideal for greenfield projects and organizations committed to contract testing.
- **Provider-Driven Contract Testing (PDCT):** Uses existing OpenAPI specifications of the provider and executes the consumer tests against that, instead of the traditional provider side test execution (of the contract) against the locally served provider service.

**Communication Types Supported:**

- **GraphQL:** Supported through a specific wrapper in Pact, allowing flexible data requests.
- **Event-Driven Systems:** Supported by abstracting the message body from the messaging technology, enabling contract testing in systems using asynchronous messaging.

## Consumer side

The consumer can be any client that makes API calls. Can be an API service, can be a web app (using Axios for example); it does not make a difference.

- **Focus:** Consumer contract tests should focus solely on ensuring that the consumer’s expectations match the provider’s responses, without delving into the provider’s internal functionality.
- **Loose Matchers:** Use loose matchers to avoid high coupling, allowing flexible assertions.
- **Isolated Tests:** stateless, order independent.
- **Using can-i-deploy:** The can-i-deploy tool in Pact helps verify whether changes are safe to deploy, ensuring that the provider has successfully verified the contract before deployment.

Here is how it works:

1. Write the consumer test.

2. Execute the and generate a contract / pact / json file.

   The contract specifies how the provider should respond upon receiving requests from the consumer.

3. Once the contract is created, from then on the Pact `mockProvider` takes over as if we are locally serving the provider API and executing the tests against that.
   That means, there is no need to serve the client api or the provider api at the moment, the consumer tests and `mockProvider` cover that interaction.

4. The consumer test can only fail at the `executeTest` portion, if / when the assertions do not match the specifications. Any changes to the `provider` section makes updates to the contract.

Here's how a test generally looks:

```js
// ...provider setup prior...

it('...', () => {
  provider
    // specifications about how the provider
    // should behave upon receving requests
    // this part is what really configures the contract

  await provider.executeTest(async(mockProvider) => {

    // assertions against the mockProvider/contract
  })
})
```

Run the consumer tests:

```bash
npm run test:consumer
```

The pact gets recorded, the consumer tests (`executeTest`) are verified against the contract.

Now, for the provider to know about it all, we need to publish the contract

Publish the contract to your Pact Broker:

```bash
npm run publish:pact
```

The consumer side uses a mock provider. The specifics of the mock provider are defined in the it blocks.

> What is the difference? TL, DR; a mock is a smarter stub.
>
> **Stub:** Provides a simple, fixed response based on a predefined script. It doesn’t validate how it's used.
>
> **Mock:** More advanced than a stub; it imitates a real object’s behavior and validates interactions, such as method calls and arguments, making it more suitable for scenarios like contract testing.
>
> ```typescript
> // Define the ApiClient interface
> interface ApiClient {
>   fetchUser(userId: number): { id: number; name: string } | null;
> }
>
> // The UserService depends on the ApiClient to fetch user data
> class UserService {
>   private apiClient: ApiClient;
>
>   constructor(apiClient: ApiClient) {
>     this.apiClient = apiClient;
>   }
>
>   getUser(userId: number): { id: number; name: string } | null {
>     return this.apiClient.fetchUser(userId);
>   }
> }
>
> ////// STUB
> const apiClientStub: ApiClient = {
>   fetchUser: (userId: number) => {
>     return { id: userId, name: "John Doe" }; // Predetermined response
>   },
> };
>
> // same in both
> const userService = new UserService(apiClientStub);
> const user = userService.getUser(1);
> console.log(user); // { id: 1, name: 'John Doe' }
>
> /////// MOCK
> const apiClientMock: ApiClient = {
>   fetchUser: jest.fn((userId: number) => {
>     if (userId === 1) {
>       return { id: 1, name: "John Doe" };
>     }
>     return null;
>   }),
> };
>
> // same in both
> const userService = new UserService(apiClientMock);
> const user = userService.getUser(1);
> console.log(user); // { id: 1, name: 'John Doe' }
> // KEY: verify that fetchUser was called with the correct argument
> expect(apiClientMock.fetchUser).toHaveBeenCalledWith(1);
> ```

## Provider side

The main goal is to verify that the provider API can fulfill the contract expectations defined by the consumer(s). This ensures that any changes made to the provider won't break existing consumer integrations.

Here is how it works

1. The consumer already generated the contract and published it.
2. The provider has one test per consumer to ensure all is satisfactory. Most of the file is about setting up the options.
3. We ensure that the provider api is running locally.
4. The consumer tests execute against the provider api, as if they are a regular API client running locally.

Here is how the test generally looks:

```js
const options = {..} // most the work is here (ex: provider states)
const verifier = new Verifier(options)

it('should validate the expectations..', () => {
  return verifier.verifyProvider().then((output) => {
    console.log('Pact Verification Complete!')
    console.log('Result:', output)
  })
})
```

The provider API has to be running locally for the provider tests to be executed. Then we can execute the provider test.

```bash
npm run start:provider

# another tab

npm run test:provider
```

**Provider States**: We can simulate certain states of the api (like an empty or non-empty db) in order to cover different scenarios

- Provider states help maintain the correct data setup before verification.
- State handlers must match the provider states defined in consumer tests.

**Can-I-Deploy Tool**: Before deploying to an environment, we verify if the consumer and provider versions are compatible using the `can-i-deploy` tool. This step ensures that any changes made to the consumer or provider do not break existing integrations across environments.

Verify the provider:

```bash
npm run can:i:deploy:provider
```

Verify the consumer:

```bash
npm run can:i:deploy:consumer
```

## The Pact Broker

Why do we need a pact broker?

1. Sharing contracts: it is what binds the repositories together, without needing a common deployment.
2. Coordination: it can coordinate contracts and releases between branches, environments and teams.
3. CI: the pact broker provides a CLI which offers easy access to provider verification status and webhooks to trigger dependent CI pipelines.

Pact broker flow:

1. The Consumer pushes the consumer contracts to the Pact broker.
2. The provider pulls the contracts from the Pact broker.
3. The provider publishes the verification status of the contract to the Pact broker.
4. The consumer & the provider pull the verification status from the Pact broker to check if they can deploy their service/app.

### Example new release scenario:

Imagine you have a **Movies API** that serves different clients, such as a web application and a mobile app. These clients are at different stages of development and are deployed in different environments (e.g., Staging and Production).

#### Environments:

- **Staging Environment:** Running version **v1.2.0** of the Movies API.
- **Production Environment:** Running version **v1.1.0** of the Movies API.

#### Pact Broker's Role:

- A new version of the API, **v2.0.1**, has been developed to fix some bugs.
- Before this version can be deployed, the Pact Broker checks it against the existing consumer contracts (which specify how clients interact with the API) to ensure it won't break any current implementations.
- The Pact Broker verifies the contract against **v1.2.0** in the Staging environment and **v1.1.0** in the Production environment.
- The checkmarks indicate that both environments successfully verified the new contract, meaning **v2.0.1** can be safely integrated into either environment.

### **SaaS (PactFlow) vs. Self-Hosted Pact Broker**

**SaaS (PactFlow) Solution:**

- **Cost:** SaaS cost, but no setup or maintenance effort.
- **Maintenance:** none.
- **Features:** Offers advanced features like **Can I Deploy**, **API token-based security**, and support for **Provider driven contracts** (e.g., OpenAPI).
- **Ease of Use:** Easy to set up, with a user-friendly interface and seamless third-party integrations.
- **Support:** Dedicated support from the PactFlow team.

**Self-Hosted Pact Broker:**

- **Cost:** Lower running costs, but requires infrastructure setup and residual maintenance.
- **Maintenance:** Requires ongoing management of security patches and updates.
- **Features:** Provides core functionality with basic **username/password security**.
- **Setup Complexity:** More complex setup, especially when using Docker or Kubernetes.
- **Support:** Relies on community support, which may be less consistent.

## PactBroker CI/CD & key features

**Environment Variables:** Why `GITHUB_SHA` and `GITHUB_BRANCH`?

- **`GITHUB_SHA`**: This variable represents the unique commit ID (SHA) in Git. By using the commit ID as the version identifier when publishing the contract or running tests, you can precisely trace which version of your code generated a specific contract. This traceability is crucial in understanding which code changes correspond to which contract versions, allowing teams to pinpoint when and where an issue was introduced.

- **`GITHUB_BRANCH`**: Including the branch name ensures that contracts and deployments are correctly associated with their respective branches, supporting scenarios where different branches represent different environments or features under development. It helps prevent conflicts or mismatches in contracts when multiple teams or features are being developed simultaneously.

  TL,DR; best practice, do it this way.

On the provider side Pact verifies all the relevant versions that are compatible:

```js
// provider spec file

const options = {
  // ...
  providerVersion: process.env.GITHUB_SHA,
  providerVersionBranch: process.env.GITHUB_BRANCH, // represents which contracts the provider should verify against
  consumerVersionSelectors = [
      { mainBranch: true },  // tests against consumer's main branch
      { matchingBranch: true }, // tests against consumer's currently deployed and currently released versions
      { deployedOrReleased: true } // Used for coordinated development between consumer and provider teams using matching feature branch names
    ]
}
```

### The matrix:

The Pact Matrix is a feature within Pactflow (or other Pact brokers) that visualizes the relationships between consumer and provider versions and their verification status across different environments. The matrix shows:

- Which versions of consumers are compatible with which versions of providers.
- The verification results of these interactions across various environments (e.g., dev, stage, prod).

By using `GITHUB_SHA` and `GITHUB_BRANCH` in your CI/CD workflows, you ensure that the matrix accurately reflects the state of your contracts and their verifications. This makes it easier to determine if a particular consumer or provider version is safe to deploy in a specific environment, ultimately enabling seamless integration and deployment processes.

Example matrix:

| **Consumer Version (SHA)** | **Provider Version (SHA)** | **Branch**  | **Environment** | **Verification Status** | **Comments**                                                                                             |
| -------------------------- | -------------------------- | ----------- | --------------- | ----------------------- | -------------------------------------------------------------------------------------------------------- |
| `abc123`                   | `xyz789`                   | `main`      | `production`    | Passed                  | The consumer and provider are both verified and deployed in production.                                  |
| `def456`                   | `xyz789`                   | `main`      | `staging`       | Passed                  | The same provider version is compatible with a newer consumer version in staging.                        |
| `ghi789`                   | `xyz789`                   | `feature-x` | `development`   | Failed                  | The consumer from a feature branch failed verification with the provider in the development environment. |
| `jkl012`                   | `uvw345`                   | `main`      | `production`    | Pending                 | A new provider version is pending verification against the consumer in production.                       |

### **can-i-deploy**

The `can-i-deploy` tool queries the Pact Broker to ensure that the consumer and provider versions are compatible before deployment. This tool provides an extra layer of safety, preventing the deployment of incompatible versions.on.

### **Webhooks:**

Recall the consumer and provider flow.

The key is that, when there are multiple repos, the provider has to run `test:provider` `(#3)` after the consumer runs `publish:pact` `(#2)` but before the consumer can run `can:i:deploy:consumer` `(#4)` . The trigger to run `test:provider` `(#3)` has to happen automatically, webhooks handle this.

```bash
# Consumer
npm run test:consumer # (1)
npm run publish:pact  # (2)
npm run can:i:deploy:consumer # (4)
# only on main
npm run record:consumer:deployment # (5)

# Provider
npm run test:provider # (3) triggered by webhooks
npm run can:i:deploy:provider # (4)
# only on main
npm run record:consumer:deployment # (5)
```

## Provider-driven (bi-directional) contract testing

Consumer-Driven Contract Testing is highly effective when the consumer has a close relationship with the provider, typically within the same organization. However, when dealing with third-party providers, especially those who might not be aware of or prioritize the specific needs of any single consumer, this approach becomes unfeasible.

**Unknown Consumers**: In some cases, a third-party provider may not even know who all of their consumers are. They provide a public API, and various clients may interact with it. In such scenarios, it’s impractical for the provider to tailor their API to specific consumer-driven contracts.

**Provider-Driven Contract Testing** makes more sense in these situations. The provider defines the contract (API specification), and it’s up to the consumers to align with this specification. This shifts the responsibility to the consumer to ensure that they are compatible with the provider’s API, rather than the other way around.

The flow:

1. The provider uploads their OpenAPI specification to PactFlow.
2. The consumer uploads their contract, generated by their consumer-driven contract tests (CDCT).
3. PactFlow cross-validates the contracts to ensure compatibility between the provider's capabilities and the consumer's expectations.
4. Both the provider and the consumer use the `can-i-deploy` tool to verify whether it's safe to deploy their changes.

The key difference in this approach is that instead of the provider running the contract tests / verifying the contract on their end, they upload their OpenAPI specification to PactFlow. PactFlow then takes care of the verification by comparing the consumer's expectations with the provider's actual capabilities. When working with a 3rd party, the teams can upload a copy of their OpenAPI to PactFlow, and verify their contract testing.

**Caveat**: **Risk of False Confidence**: Since the testing is based on the OpenAPI spec of the provider rather than the actually running the consumer tests (the contract) on the provider side, there's a risk that the contract might not fully capture the nuances of the provider's implementation. This could lead to scenarios where a contract is deemed compatible even though the actual service could fail in production. This risk emphasizes the importance of maintaining up-to-date and accurate contracts.

(This gap could be addressed with [generating OpenAPI docs from types](https://dev.to/muratkeremozcan/automating-api-documentation-a-journey-from-typescript-to-openapi-and-schema-governence-with-optic-ge4), or generating OpenAPI spec from e2e (an Optic feature))

### Cypress adapter in a nutshell

**UI Tests**: You have your regular Cypress UI tests where you test the behavior of your application.

**Stubbing the Network**: You use `cy.intercept` to stub network requests. This allows you to mock responses from APIs, ensuring that your UI tests are consistent and isolated from backend fluctuations.

**Generating a Pact Contract**: When you use the `cy.setupPact` command in conjunction with `cy.intercept`, during the test execution, the Cypress Pact adapter generates a Pact contract. This contract captures the expectations set by the stubbed network requests.

**CDCT-Like Use**: The generated contract can be used in a similar fashion to a Consumer-Driven Contract Testing (CDCT) setup. You can publish this contract to a Pact Broker and have your provider verify it, ensuring that the provider meets the consumer's expectations.

## CDCT vs e2e vs integration vs schema testing

### Testing Lambda Handlers (Unit or Integration Tests)

- **Fast Feedback Loop**:
  - **Quick Iterations**: Enables rapid feedback through tests directly on the Lambda handler.
  - **Easier Debugging**: Isolates the Lambda function, simplifying the debugging process.
- **Partial Coverage**:
  - **Integration Points**: Can test Lambda interactions with services like DynamoDB Local, KMS, Kafka, etc., within a Docker environment. (This is the distinction between unit and integration.)
  - **Error Simulation**: Allows for controlled simulations of various edge cases and failure conditions.
- **Drawbacks**:
  - **Less Realistic**: Does not test the entire request-response lifecycle, potentially missing configuration or integration issues outside the Lambda function.
  - **Requires Mocks for Some Scenarios**: Dependency isolation may necessitate mocks, which could lead to false positives.
  - **Other**: Lacks testing for build & configuration, deployments, IAM, or interactions with external services.

### Testing Deployments via HTTP (Temporary Stack, Dev, Stage)

- **Build & Configuration**: Validates that all cloud resources, including CDK/Serverless Framework / infra as code, environment variables, secrets, and services (API Gateway, Lambda, S3), are correctly operational.
- **IAM Permissions**: Ensures that the IAM roles and permissions are correctly configured.
- **Comprehensive Coverage**: Tests the entire system, including authentication mechanisms and integrations with other services.
- **Drawbacks**:
  - Requires deployment, which adds complexity.
  - Results in a slower feedback loop due to the deployment process.

### Testing Local Endpoints via HTTP (Local)

- **Consistency**: Same tests can be run locally as in deployment environments, offering low-cost testing.
- **Ease of Implementation**: Simple to test API interactions before development using tools like REST clients or Postman.
- **Network Mocking**: Easy to configure network mocks with tools like Mockoon, without code changes.
- **No Deployment Drawbacks**: Avoids deployment, providing fast feedback.
- **Caveats**: Lacks testing for build & configuration, deployments, IAM, or interactions with external services.

### CDCT (Contract-Driven Contract Testing)

- **External Service Interaction Coverage**: Addresses a major drawback of local testing by ensuring compatibility with external services.
- **Fast Feedback**: Similar to local testing, CDCT avoids deployment and provides rapid feedback.
- **Caveats**: Does not cover build & configuration, deployments, IAM, or full end-to-end interactions.

### What kind of e2e testing would you replace with contract testing?

- **Replicate what contract tests cover or are overly simple**: If an e2e test simply checks basic API functionality—like whether an API responds with the correct status code or data structure—and this is already thoroughly covered by a contract test, the e2e test may be redundant. This is especially true if the e2e test doesn't involve multi-step interactions, such as getting a response and then performing additional actions based on that response. In such cases, contract tests might provide the necessary coverage, making the e2e test unnecessary.
- **Have low value for effort**: Tests that are complex to maintain but provide little value, such as those that test non-critical paths or scenarios unlikely to change, could be candidates for removal.
- **Streamline Redundant Coverage for build, config, deployment, IAM etc.**: If your contract tests are comprehensive and your existing e2e tests already cover key build & configuration, deployment, infrastructure, and IAM, you consider trimming or removing overlapping e2e tests.
- **Optimize Deployment Strategy**: You can consider skipping temporary stacks or ephemeral environments on PRs and instead focus on deployment testing in the dev environment. This strategy can save costs associated with deployments from scratch. However, be mindful that this may delay the discovery of issues related to build & configuration, deployment, or IAM until later stages. It's a tradeoff between cost efficiency and early detection.
- **Minimize the need for synchronized service versions covered bye 2e**: In traditional testing setups, you might rely on running e2e tests across multiple services in later environments like dev or staging to ensure that changes to Service A don't break Services B, C; deploy A (run e2e for A), but also run e2e for B & C although they didn't deploy. With contract tests, this need is significantly reduced, as they ensure compatibility between services at the contract level.

### How does CDCT fit with schema testing (Optic)?

There is a potential gap in Provider-Driven Contract Testing where the OpenAPI spec provided by the API might not accurately reflect the current implementation of the code. This gap can be mitigated by [generating OpenAPI documentation directly from TypeScript types, or generating the OpenAPI spec from e2e tests (using Optic)](https://dev.to/muratkeremozcan/automating-api-documentation-a-journey-from-typescript-to-openapi-and-schema-governence-with-optic-ge4), ensuring that the specification remains in sync with the actual codebase.

Optic offers a low-cost, quick solution for schema validation by focusing solely on the OpenAPI specification, catching discrepancies early in the development process. On the other hand, Provider-Driven Contract Testing with Pact validates the interactions between consumers and the provider's OpenAPI spec, ensuring that the API's behavior aligns with consumer expectations.

By combining Optic for upfront schema validation and Pact for deeper contract verification, you can achieve a more comprehensive approach to API testing.

### Sample repos

Based on the excellent [Contract Testing in Action](https://www.manning.com/books/contract-testing-in-action) book by Marie Cruz and Lewis Prescott, these are my interpretations of the examples.

They are a work in progress, here is my todo list:

- start-server-and-test

- switch to TS

- use sqlite, instead of in memory data

- zod

- unit tests at provider

- status badges

- setup renovate

- merge gatekeeper

- cy

- optic at provider

- mockoon at consumer

- changes

  - change movie path to movies
  - standerdized responses { status, data, message }
  - make the data a lot more complex to exercise the matchers
  - breaking change examples

https://github.com/muratkeremozcan/pact-js-example-consumer

https://github.com/muratkeremozcan/pact-js-example-provider
