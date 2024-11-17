

In our integration tests, Pact, a consumer-driven contract testing tool, has been instrumental in verifying interactions between services. However, when integrating Kafka and message queues, we encountered a perplexing issue where tests were cross-executing between different provider-consumer pairs. In this blog post, we'll delve into the problem we faced, why it was critical to resolve, and how we ultimately solved it.

You can find the solution in [this PR](https://github.com/muratkeremozcan/pact-js-example-provider/pull/113).

## The Problem: Cross-Execution of Tests Between Provider-Consumer Pairs

We had two repositories representing two distinct provider-consumer pairs:

[Provider (MoviesAPI)](https://github.com/muratkeremozcan/pact-js-example-provider)

[Consumer (WebConsumer)](https://github.com/muratkeremozcan/pact-js-example-consumer)

Where the 2 repos have 2 unique contracts between them:

1. **MoviesAPI ↔ WebConsumer** (HTTP-based interaction)
    
2. **MoviesAPI-event-producer ↔ WebConsumer-event-consumer** (Kafka message-based interaction)
    

Each pair had its own Pact contracts and verification tests. Locally, everything worked flawlessly, in CI PRs as well.

However, when the consumer triggered a webhook in the provider repository (as part of the CI/CD pipeline), we encountered [a strange issue](https://github.com/muratkeremozcan/pact-js-example-provider/actions/runs/11354888840/job/31583091638):

- The Kafka test (`provider-kafka.pacttest.ts`), intended for the `MoviesAPI-event-producer` and `WebConsumer-event-consumer`, was inadvertently executing tests meant for the HTTP-based pair (`MoviesAPI` and `WebConsumer`).
    
- This cross-execution led to test failures and confusion, especially since the tests were running in the context of the wrong provider-consumer pair.
    

### Root Cause Analysis

The crux of the problem was the handling of the `PACT_PAYLOAD_URL` environment variable:

The `PACT_PAYLOAD_URL` was being set globally, causing both test suites to pick it up, regardless of whether it was relevant to them.

This is the way things have to be, because during a webhook trigger, which is caused by a `repository_dispatch`, we want an HTTP request from the consumer repo to the provider repo instructing GitHub to start the [webhook action](https://github.com/muratkeremozcan/pact-js-example-provider/blob/main/.github/workflows/webhook.yml).

A repository_dispatch in GitHub is basically an HTTP request to your GitHub project instructing GitHub to start any action or webhook. In our example, a repository_dispatch with the event type `contract_requiring_verification_published` will trigger the workflow.

In our local tests and in PRs, PACT_BROKER_BASE_URL is used, and during a web hook trigger it is replaced with PACT_PAYLOAD_URL. This is because we want to verify the newly published contract from the consumer repo PR that caused the trigger.

> Recall the consumer and provider flow.
> 
> The key is that, when there are multiple repos, the provider has to run `test:provider-ci` `(#3)` after the consumer runs `publish:pact` `(#2)` but before the consumer can run `can:i:deploy:consumer` `(#4)` . The trigger to run `test:provider-ci` `(#3)` has to happen automatically, webhooks handle this.
> 
> # Consumer  
> npm run test:consumer # (1)  
> npm run publish:pact  # (2)  
> npm run can:i:deploy:consumer # (4)  
> # only on main  
> npm run record:consumer:deployment --env=dev # (5) change the env param as needed  
> ​  
> # Provider  
> npm run test:provider-ci # (3) triggered by webhooks  
> npm run can:i:deploy:provider # (4)  
> # only on main  
> npm run record:provider:deployment --env=dev # (5) change the env param as needed

## Why Was the Problem Worth Solving

Ensuring that each test suite only executes its intended tests is crucial for several reasons:

1. **Accuracy**: Tests should only verify the contracts relevant to their specific provider-consumer pair.
    
2. **Reliability**: Cross-execution can lead to false negatives, causing CI/CD pipelines to fail unnecessarily.
    
3. **Scalability**: As more provider-consumer pairs are added, the problem would compound, leading to more significant issues down the line.
    

By resolving this issue, we aimed to improve the reliability of our test suites, streamline our CI/CD processes, and set a foundation for scalable contract testing.

## How We Solved It

To address the issue, we undertook a systematic approach:

### 1. Refactoring the URL Handling Logic

We focused on the [`handlePactBrokerUrlAndSelectors`](https://github.com/muratkeremozcan/pact-js-example-provider/pull/113/files#diff-3fe9c15580c38cccec13f1adf95cbb093658c395f41d5fe86f140136ff1cff9eR18) function, which was responsible for configuring the verifier options based on the presence of a `PACT_PAYLOAD_URL`.

#### Original Issues:

- **Complexity**: The function had a high cyclomatic complexity, making it hard to read and maintain.
    
- **Global Variable Usage**: It relied on globally set environment variables, leading to unintended side effects.
    
- **Lack of Conditional Handling**: There was no check to ensure that the `PACT_PAYLOAD_URL` matched the expected provider and consumer before using it.
    

#### Refactored Solution:

We broke down the `handlePactBrokerUrlAndSelectors` function into smaller, focused functions:

- [`parseProviderAndConsumerFromUrl`](https://github.com/muratkeremozcan/pact-js-example-provider/pull/113/files#diff-3fe9c15580c38cccec13f1adf95cbb093658c395f41d5fe86f140136ff1cff9eR96): Extracts the provider and consumer names from the `PACT_PAYLOAD_URL`.
    
- [`processPactPayloadUrl`](https://github.com/muratkeremozcan/pact-js-example-provider/pull/113/files#diff-3fe9c15580c38cccec13f1adf95cbb093658c395f41d5fe86f140136ff1cff9eR53): Determines whether the `PACT_PAYLOAD_URL` should be used based on whether it matches the expected provider and consumer.
    
- [`usePactPayloadUrl`](https://github.com/muratkeremozcan/pact-js-example-provider/pull/113/files#diff-3fe9c15580c38cccec13f1adf95cbb093658c395f41d5fe86f140136ff1cff9eR119) Configures the verifier options to use the `PACT_PAYLOAD_URL` for verification.
    
- [`usePactBrokerUrlAndSelectors`](https://github.com/muratkeremozcan/pact-js-example-provider/pull/113/files#diff-3fe9c15580c38cccec13f1adf95cbb093658c395f41d5fe86f140136ff1cff9eR141): Configures the verifier options to use the Pact Broker URL and appropriate consumer version selectors.
    

By modularizing the code, we reduced complexity and made it easier to understand and maintain.

### 2. Implementing Conditional Logic

We introduced logic to ensure that a test suite only uses the `PACT_PAYLOAD_URL` if it matches its specific provider and consumer:

if (providerMatches && consumerMatches) {  
  usePactPayloadUrl(pactPayloadUrl, options);  
  return true; // Indicate that the Pact payload URL was used  
} else {  
  console.log(  
    `PACT_PAYLOAD_URL does not match the provider (${options.provider}) and consumer (${consumer || 'all'}), ignoring it`  
  );  
}

This check prevents a test suite from inadvertently using a `PACT_PAYLOAD_URL` intended for a different provider-consumer pair.

### 3. Ensuring Proper Fallback

If the `PACT_PAYLOAD_URL` is not provided or doesn't match, the verifier options are configured to use the Pact Broker URL and consumer version selectors:

// If pactPayloadUrl is not provided or doesn't match, use the Pact Broker URL and selectors  
usePactBrokerUrlAndSelectors(  
  pactBrokerUrl,  
  consumer,  
  includeMainAndDeployed,  
  options  
);

This ensures that the test suite can still run and verify contracts appropriately, even in the absence of a matching `PACT_PAYLOAD_URL`.

### 4. Updating Test Files

In our test files (`provider-contract.pacttest.ts` and `provider-kafka.pacttest.ts`), we made sure to specify the correct `provider` and `consumer` when building the verifier options. This information is crucial for the `processPactPayloadUrl` function to correctly determine whether to use the `PACT_PAYLOAD_URL`.

const options = buildVerifierOptions({  
  provider: 'MoviesAPI',  
  consumer: 'WebConsumer',  
  // ... other options  
})  
​  
///  
​  
const options = buildMessageVerifierOptions({  
  provider: 'MoviesAPI-event-producer',  
  consumer: 'WebConsumer-event-consumer',  
})

### 5. Verifying the Solution

Trigger a push from the consumer repo, and we had a [CI failure at the provider web hook job](https://github.com/muratkeremozcan/pact-js-example-provider/actions/runs/11354888840/job/31583091638).

vs

Trigger a push from the consumer repo again, and [no more CI failure](https://github.com/muratkeremozcan/pact-js-example-provider/actions/runs/11368869071/job/31624971740).

## Conclusion

By refactoring our code and introducing conditional logic to handle the `PACT_PAYLOAD_URL` appropriately, we resolved the cross-execution issue in our Pact tests involving Kafka and message queues. This solution not only fixed the immediate problem but also enhanced the maintainability and scalability of our testing framework.

**Key Takeaways:**

- **Modular Code Improves Maintainability**: Breaking down complex functions into smaller, focused ones makes the codebase easier to understand and work with.
    
- **Conditional Logic Prevents Cross-Execution**: Implementing checks to ensure that only the intended tests are executed prevents cross-execution issues.
    
- **Testing Various Scenarios**: Verifying the solution under different conditions ensures robustness and reliability.
    

**Final Thoughts:**

Addressing this issue was crucial for maintaining the integrity of our testing process. As microservices architectures continue to grow in complexity, having reliable and maintainable testing practices becomes ever more important. We hope that sharing our experience will help others facing similar challenges in their testing journeys.