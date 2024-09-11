# Strengthening Pact Contract Testing with TypeScript and Data Abstraction

- [Strengthening Pact Contract Testing with TypeScript and Data Abstraction](#strengthening-pact-contract-testing-with-typescript-and-data-abstraction)
  - [What are State Handlers?](#what-are-state-handlers)
  - [The Problem: Type Mismatch in Provider State Handling](#the-problem-type-mismatch-in-provider-state-handling)
  - [Introducing `createProviderState`: A Type-Safe Abstraction](#introducing-createproviderstate-a-type-safe-abstraction)
    - [What is `JsonMap`?](#what-is-jsonmap)
    - [So, how do we start fixing things?](#so-how-do-we-start-fixing-things)
    - [The abstraction we want: `createProviderState`](#the-abstraction-we-want-createproviderstate)
    - [Example: Consumer Side Tests](#example-consumer-side-tests)
  - [Aligning Provider State Handlers with Strong Typing](#aligning-provider-state-handlers-with-strong-typing)
  - [**Conclusion**](#conclusion)

Working on a pair of sample repositories for consumer-driven contract testing with Pact, I realized that a major potential breaking point when setting up state handlers is when the state parameters are being passed from the consumer to the provider.

Let the code talk; here are the sample repos and PRs the blog post is about:

https://github.com/muratkeremozcan/pact-js-example-consumer/pull/14/files

https://github.com/muratkeremozcan/pact-js-example-provider/pull/7/files

### What are State Handlers?

In Pact testing, **state handlers** are functions on the provider side that prepare the provider's system under test to be in a specific state before verifying the interactions described in the Pact. These states are described by the consumer during the contract test. For example, a state might describe a scenario where "a movie with a specific ID exists in the database." The consumer will call this state, and the provider's state handler will set up the necessary conditions to meet this requirement.

## The Problem: Type Mismatch in Provider State Handling

As it is, on the consumer side we are allowed to pass in anything. The JS and TS versions of the test are the same.

```ts
// consumer-contract.pacttest.ts
it('should return a specific movie', async () => {
  const testId = 100
  const EXPECTED_BODY = { id: testId, name: 'My movie', year: 1999 }
  const params = { id: testId }

  provider
    .given('Has a movie with a specific ID', params)
    // ...
```

```ts
// another consumer side test
it('should not add a movie that already exists', async () => {
  const { name, year } = {
    name: 'My existing movie',
    year: 2001
  }
  const params = { name, year }

  provider
    .given('An existing movie exists', params)
    // ...
```

The `given` method accepts a type of `(providerState: string, parameters?: JsonMap)`. As users, we care about the state and parameters we are passing in for the provider to use. The provider will have to setup the stateHandlers, and run the test suite / pact from the consumer against its own locally served server. There comes the main potential breaking point; what if the state/params passed in are not correct? This can become a significant issue, especially when dealing with complex data structures - in the PRs I had much trouble working with objects with 1-2 properties!

Here’s the working provider-side state handler in JavaScript. It worked fine, but TypeScript revealed issues:

```ts
// provider-contract.pacttest.ts

// @ts-nocheck - the only way this would work with TS

const stateHandlers = {
  "Has a movie with a specific ID": (params) => {
    movies.getFirstMovie().id = params.id;
    return Promise.resolve({
      description: `Movie with ID ${params.id} added!`,
    });
  },
  "An existing movie exists": (params) => {
    movies.addMovie(params);
    return Promise.resolve({
      description: `Movie with ID ${params.id} added!`,
    });
  },
};

// and then it's used at options
const options: VerifierOptions = {
  // ...
  stateHandlers,
};
```

## Introducing `createProviderState`: A Type-Safe Abstraction

The core of the problem is how we pass the parameters from the consumer. There is no straightforward way to make the provider types happy without changing that. The challenge lies in the fact that Pact accepts `JsonMap` during its communication between the consumer and provider.

### What is `JsonMap`?

`JsonMap` is a type used by Pact that ensures the parameters passed between the consumer and provider are in a format compatible with JSON. The keys must be strings, and the values must be JSON-serializable. Failing to convert parameters properly can lead to issues, particularly with complex nested structures. Without proper conversion, you may encounter runtime errors or subtle bugs where the provider's state doesn't match what the consumer expects. This becomes especially problematic when dealing with objects containing nested objects, arrays, or dates, as these might not be automatically converted into the format expected by Pact, leading to unexpected behavior or test failures.

### So, how do we start fixing things?

Well, if pact expects JsonMap, we have to pass in JsonMap. That means we have to convert all key values to strings.

```ts
it('should return a specific movie', async () => {
  const testId = 100
  const EXPECTED_BODY = { id: testId, name: 'My movie', year: 1999 }
  const params = { id: String(testId) } // PLEASE NO

  provider
    .given('Has a movie with a specific ID', params)
    // ...
})

it('should not add a movie that already exists', async () => {
  const { name, year } = {
    name: 'My existing movie',
    year: 2001
  }
  const params = { name, String(year) } // PLEASE NO

  provider
    .given('An existing movie exists', params)
    // ...
})
```

That is a hassle to remember, and can get very error prone and painful when the data structure is complex. Why don't we have some nice utility, where we pass in what we need as the user, and Pact takes care of whatever it needs for its communication without us needing to know about it.

### The abstraction we want: `createProviderState`

To address this, we developed the `createProviderState` function. This utility abstracts the complexity of passing data between the consumer and provider by converting parameters into a `JsonMap` that complies with Pact's expectations. It ensures that all data passed to the provider is properly formatted, handling everything from `null` values to complex objects.

```ts
import type { JsonMap } from "@pact-foundation/pact/src/common/jsonTypes";

const toJsonMap = (obj: Record<string, unknown>): JsonMap =>
  Object.fromEntries(
    Object.entries(obj).map(([key, value]) => {
      if (value === null || value === undefined) {
        return [key, "null"];
      } else if (
        typeof value === "object" &&
        !(value instanceof Date) &&
        !Array.isArray(value)
      ) {
        return [key, JSON.stringify(value)];
      } else if (typeof value === "number" || typeof value === "boolean") {
        return [key, value];
      } else if (value instanceof Date) {
        return [key, value.toISOString()];
      } else {
        return [key, String(value)];
      }
    })
  );

type ProviderStateInput = {
  name: string;
  params: Record<string, unknown>;
};

export const createProviderState = ({
  name,
  params,
}: ProviderStateInput): [string, JsonMap] => [name, toJsonMap(params)];
```

With `createProviderState`, we not only abstract the necessary conversion but also ensure a clean API that communicates exactly what the right approach is.

### Example: Consumer Side Tests

Here are the consumer-side tests before and after using `createProviderState`:

```ts
// consumer-contract.pacttest.ts

// before
it("should return a specific movie", async () => {
  const testId = 100;
  const EXPECTED_BODY = { id: testId, name: "My movie", year: 1999 };
  const params = { id: String(testId) }; // PLEASE NO

  provider.given("Has a movie with a specific ID", params);
  // ...
});
```

```ts
// after
it("should return a specific movie", async () => {
  const testId = 100;
  const EXPECTED_BODY = { id: testId, name: "My movie", year: 1999 };

  const [stateName, stateParams] = createProviderState({
    name: "Has a movie with a specific ID",
    params: { id: testId },
  });

  provider.given(stateName, stateParams);
  // ..
});
```

```ts
// another consumer side test

// before
it('should not add a movie that already exists', async () => {
  const { name, year } = {
    name: 'My existing movie',
    year: 2001
  }
  const params = { name, String(year) } // PLEASE NO

  provider
    .given('An existing movie exists', params)
    // ...
})
```

```ts
// after
it("should not add a movie that already exists", async () => {
  const movie: Movie = {
    name: "My existing movie",
    year: 2001,
  };

  const [stateName, stateParams] = createProviderState({
    name: "An existing movie exists",
    params: movie,
  });

  provider.given(stateName, stateParams);
  // ...
});
```

It seems like a minor change, but without `createProviderState`, you would have to manually convert parameters into a format compliant with Pact's `JsonMap`, leading to confusing and error-prone code, especially with complex data structures.

## Aligning Provider State Handlers with Strong Typing

On the provider side, we need to ensure that the types of incoming parameters match what we expect while still making sure the tests work. One of the hard parts of consumer driven contract testing is diagnosing why the provider side test fails. When the types aren't working correctly, it's much harder to diagnose why the provider-side test fails. By ensuring strong typing, we can isolate issues more effectively simply because the surface area of the failure is smaller.

We start by using the existing types we already have, like `MovieType`. In a larger organization, these types could be defined in shared libraries, ensuring consistency across different projects and teams. By using a common set of types, everyone can agree on and rely on, we reduce duplication and potential errors. The params being passed in from the consumer are then just a subset of those known and trusted types, promoting reusability and consistency. This approach also facilitates collaboration and code maintenance, as developers across the organization can confidently work with the same data structures without having to redefine them.

```ts
// provider-contract.pacttest.ts
// import from wherever
type MovieType = {
  id: number;
  name: string;
  year: number;
};

// the types we will use in the state handler
// matching the parameters coming from the consumer
type ExistingMovieParams = Omit<MovieType, "id">;
type HasMovieWithSpecificIDParams = Omit<MovieType, "name" | "year">;
```

The below is what the state handler looks like before and after.

We have to match the name of the state versus the consumer side.

We have to use `AnyJson` as the `params` type, because that is what Pact communicates with and expects for the types.

We cast params into what we actually need (`HasMovieWithSpecificIDParams`, `ExistingMovieParams`) and destructure it.

```ts
// provider-contract.pacttest.ts

// before
// @ts-nocheck - the only way this would work with TS
const stateHandlers = {
  "Has a movie with a specific ID": (params) => {
    movies.getFirstMovie().id = params.id;
    return Promise.resolve({
      description: `Movie with ID ${params.id} added!`,
    });
  },
  "An existing movie exists": (params) => {
    movies.addMovie(params);
    return Promise.resolve({
      description: `Movie with ID ${params.id} added!`,
    });
  },
};
```

```ts
// after
type HasMovieWithSpecificIDParams = Omit<MovieType, "name" | "year">;
type ExistingMovieParams = Omit<MovieType, "id">;

const stateHandlers: StateHandlers & MessageStateHandlers = {
  "Has a movie with a specific ID": (params: AnyJson) => {
    const { id } = params as HasMovieWithSpecificIDParams;
    const movie = movies.getFirstMovie();

    if (!movie) {
      return Promise.reject(new Error("No movie found to update"));
    }

    movie.id = id;
    return Promise.resolve({
      description: `Movie with ID ${id} added!`,
    });
  },
  "An existing movie exists": (params: AnyJson) => {
    const { name, year } = params as ExistingMovieParams;
    const movie = { name, year };

    movies.addMovie(movie);
    return Promise.resolve({
      description: `Movie with name ${movie.name} added!`,
    });
  },
};
```

This approach allows us to enforce type safety while ensuring the tests are robust and maintainable. By aligning the provider-side state handlers with the types expected from the consumer, we minimize the risk of errors and promote consistency across the codebase.

Another benefit of this approach is that the types defined for state handlers can be shared across the provider repo, or even the organization. After all, the exact state is shared, no matter where the test is being run. This promotes consistency and reduces duplication of effort, as developers can rely on a common set of types and patterns when writing their tests.

## **Conclusion**

By introducing `createProviderState` on the consumer side, we improved the developer experience with a desired abstraction, and on the provider side, we ensured type alignment with Pact's communication and our actual domain types. This approach not only makes our tests more reliable but also easier to understand and maintain. The clear separation of concerns and robust type handling ensures that both consumer and provider speak the same language, reducing the likelihood of miscommunication and difficult-to-diagnose test failures. Moreover, this aligns with broader TypeScript best practices, where type safety and clarity are paramount. By encapsulating complexity and ensuring that data structures conform to expected types across the entire testing process, we promote a more predictable and maintainable codebase. This abstraction helps teams focus on their business logic while letting the utility handle the complexities of data transformation.
