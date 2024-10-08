### **Building Custom Request Filters for PactJs Verifications in Express and Non-Express Environments**

#### Introduction:

When working with PactJs contract testing, [request filters](https://docs.pact.io/implementation_guides/javascript/docs/provider#verification-options) are essential for modifying HTTP requests during the verification process between a consumer and a provider. Request filters allow you to add headers, modify request bodies, or handle authentication tokens before requests are sent to the provider. However, implementing these filters can be challenging when working across non-Express setups.

> Pact docs recommend to only use `requestFilter` feature for things that cannot be persisted in the pact file. Auth tokens are a common use case. 

This blog post will walk through how to create a custom request filter that adds an Authorization header, which works both in Express environments (where middleware functions handle requests) and non-Express environments such as lambdas. Additionally, we'll explore why the solution is designed as a higher-order function and how it accommodates Pact's express-like type requirements.

Here is a link to the [PR](https://github.com/muratkeremozcan/pact-js-example-provider/pull/97) with the specific changes & the [source code](https://github.com/muratkeremozcan/pact-js-example-provider).

#### The Problem:

In certain contract testing scenarios with Pact, you may need a mechanism to modify HTTP requests, such as injecting an Authorization token into headers before the requests are sent to the provider for verification. When testing in non-Express environments, the issue lies with Pact types requiring an Express-like shape for request handling.

Express middleware typically requires three arguments: `req`, `res`, and `next`. In non-Express environments, only the request object might be available, and the absence of the `next` function (used to pass control to the next middleware) can break the flow of the request-handling logic. Thus, the solution must accommodate both cases while ensuring the logic remains flexible, especially for custom token generation.

#### The Solution: A Higher-Order Function

To solve this issue, the request filter is implemented as a **higher-order function**. This allows flexibility in how token generation logic is injected into the request handler. Additionally, it conforms to the Pact verifier's expectations of handling the `req`, `res`, and `next` arguments. This way, the filter can work seamlessly with both Express and non-Express environments.

Let's break down the code:

```typescript
// generic HttpRequest structure to accommodate both Express and non-Express environments
type HttpRequest = {
  headers: Record<string, string | string[] | undefined>
  body?: unknown
}

type NextFunction = () => void | undefined

// allows customization of token generation logic
type RequestFilterOptions = {
  tokenGenerator?: () => string
}
```

Here, `HttpRequest` is a generic structure that can represent both Express requests and non-Express requests. The `RequestFilterOptions` allows for customizable token generation by providing an optional `tokenGenerator` function.

```typescript
const handleExpressEnv = (
  req: HttpRequest,
  next: NextFunction
): HttpRequest | undefined => {
  // If this is an Express environment, call next()
  if (next && typeof next === 'function') {
    next()
  } else {
    // In a non-Express environment, return the modified request
    return req
  }
}
```

The `handleExpressEnv` function is the key to managing both Express and non-Express environments. It checks if `next` exists and, if so, it assumes the environment is Express and passes control to the next middleware. Otherwise, it simply returns the modified request for non-Express environments. The `else` clause can be modified to suite your needs.

```typescript
const createRequestFilter =
  (options?: RequestFilterOptions): ProxyOptions['requestFilter'] =>
  (req, _, next) => {
    const defaultTokenGenerator = () => new Date().toISOString()
    const tokenGenerator = options?.tokenGenerator || defaultTokenGenerator

    // add an authorization header if not present
    if (!req.headers['Authorization']) {
      req.headers['Authorization'] = `Bearer ${tokenGenerator()}`
    }

    return handleExpressEnv(req, next)
  }
```

The `createRequestFilter` is a higher-order function because it returns a function that will be used to filter requests. It allows for the optional injection of a custom token generator. Inside the function, if the `Authorization` header is missing, a token is generated and added to the request headers. After modifying the headers, it hands off the request to `handleExpressEnv` for environment-appropriate handling.

```typescript
// if you have a token generator, pass it as an option
// createRequestFilter({ tokenGenerator: myCustomTokenGenerator })
export const requestFilter = createRequestFilter()

export const noOpRequestFilter: ProxyOptions['requestFilter'] = (
  req,
  _,
  next
) => handleExpressEnv(req, next)
```

Here, we define two exports:

- `requestFilter` is the default filter that adds the `Authorization` header.
- `noOpRequestFilter` is a no-operation filter that doesn’t modify the request but still handles the environment appropriately.

These two exports are used to build verifier options for Pact tests. The `noOpRequestFilter` can be used as a default value for the  [`buildVerifierOptions` function](https://github.com/muratkeremozcan/pact-js-example-provider/blob/main/src/test-helpers/pact-utils.ts#L70). `requestFilter` can be used [directly in our tests](https://github.com/muratkeremozcan/pact-js-example-provider/blob/main/src/provider-contract.pacttest.ts#L25) to modify the http request

#### Key Takeaways:

- **Higher-Order Functions**: By using a higher-order function, we provide flexibility for future customization, like custom token generation. This design pattern is crucial in ensuring reusable and customizable logic.
- **Environment Agnosticism**: The combination of `handleExpressEnv` and higher-order functions allows the filter to work in both Express and non-Express environments. This makes the code more robust and versatile across different contexts.
- **Pact's express-like type requirements**: The filter satisfies Pact's need to handle three arguments (`req`, `res`, and `next`), even if the environment doesn't use Express, ensuring compatibility during the contract testing process.

#### Conclusion:

In summary, this custom request filter solves the challenge of modifying HTTP requests in both Express and non-Express environments while allowing for future customization. Using a higher-order function ensures that we can inject different token generation strategies, providing flexibility and maintaining compatibility with Pact’s express-like type requirements.