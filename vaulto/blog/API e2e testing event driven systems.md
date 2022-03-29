Lately I have been asked about API testing tools and the approaches to API testing event driven systems.

### What is API Testing?

Let us remember the definitions from [Test Methodologies](https://dev.to/muratkeremozcan/mostly-incomplete-list-of-test-methodologies-52no) post:

> [API testing](https://en.wikipedia.org/wiki/API_testing) : in API testing a real server is the request target; this can also be referred to as a contract test sometimes. It can also be that the tests span over multiple apis, whereby this can be referred to as API e2e testing. The goal is to test services or the integration of them at a higher level than isolated modules.
>
>[API component/module testing](https://github.com/goldbergyoni/javascript-testing-best-practices#-%EF%B8%8F-17-test-many-input-combinations-using-property-based-testing): the distinction from API testing is that all that is external to the component is mocked and the module is tested in isolation. In the NodeJS world, Nock (mock external http) and Supertest (abstract the server) can be used to achieve this kind of testing.
>
>[Consumer driven Contract testing](https://docs.pact.io/): a type of contract testing that ensures that a provider is compatible with the expectations that the consumer has of it. It is the primary way to shift API testing left, and test the integration of apis/services prior to deploying them to a common environment. Pact is a well known framework on this matter.

We are considering API testing (the first kind) tools in an event driven system.

### What is an event driven system, and why is it hard to test?

In layman terms, the event-driven system is asynchronous and non-blocking: drop the message, move on, and (maybe) pick up a confirmation message later. This can lead to frustration and confusion when API testing and changing the state of the system. Imagine this scenario:

1. Send a POST request to service A, we get a 200 - so far so good.

  > Assume that we expect a change at service B because of this

2. We send a GET to service B to verify this change, we get a 200, but there is no indication of our change, yet(!)

3. We wait for **an uncertain amount of time**, we GET again, we get a 200, and we can confirm that the change has been made.

We may do a GET at step 3, and still not be able to confirm that the change has been made! 

### How long should we wait?

The solution to the asynchronous updates seems handy: sleeping/waiting for a few seconds. If it keeps failing in CI, increase the timer. This can eventually get the test working because it gives the service the time to update and before we check the next event to be tested.

**This is an anti-pattern in testing**. We should [Await, not Sleep](https://github.com/NoriSte/ui-testing-best-practices/blob/master/sections/generic-best-practices/await-dont-sleep.md), but this is not possible with the majority of the API testing tools.

### Postman/Newman, and any tools like it (Supertest/Chai http being used as an API client) fall short

In a previous job, the system under test had a microservice we call the adapter. Mainly, it translates building automation protocol to the cloud. The dev for this microservice started out the tests in Postman back in the day, and diligently maintained them. We setup Newman to run in CI - Newman is a tool that lets us CLI test postman collections. In my opinion
for manual/local testing use cases these are fine. I prefer restUtil (vs-code extension) for instance. But, for CI, especially event driven systems, the needs are different.

I have used Supertest & Chai http for [API component/module testing](https://github.com/goldbergyoni/javascript-testing-best-practices#-%EF%B8%8F-17-test-many-input-combinations-using-property-based-testing) (definition 2 above), where all that is external to the component is mocked and the module is tested in isolation. Great tools in any environment for this use case, but not suited for API e2e testing event driven systems because they are no different than Postman in that use case.

### So what can we use for API e2e testing event driven systems?

There is a huge shortcoming with most the API test tools I know of; they do not have a built-in retry utility, therefore are synchronous; you have to hard-wait so the back-ends settle after your CRUD ops.

For example, running Newman in the pipeline, to accommodate slow pipeline conditions and deployments, I have a command:

```shell
newman run <collection> -e <postman-environment-file> --delay-request <ms-delay-btwn-requests> --env-var <environment-variable=value> "READBACK_DELAY=<in ms>"
```

`--delay-request` is the time to wait between Postman/Newman tests, `READBACK_DELAY` is the time to wait to check on a response after an operation. we have to keep increasing these to accommodate CI; this is the only way to test event driven systems with Postman/Newman and it is an anti-pattern.

At this time in the industry CI is the meta, Await, don't sleep is the motto. Postman or tools like it cannot compete in an event-driven space.

### What can this look like in a better world?

Cypress is not only a great tool for UI e2e testing handling [network requests in a web app](https://docs.cypress.io/guides/guides/network-requests#Testing-Strategies), but it is also a great api testing framework thanks to [cy.request](https://docs.cypress.io/api/commands/request#Syntax).

```javascript
  it('your API test', () => {
    // Assume Arrange & Act already happened in this test
    // (1. Send a POST request to service A, we get a 200)

    // Assert: assume that we expect a change at service B
    // (2. Now we are sending a GET to service B to verify the change)

    cy.request({
      method: 'GET',
      url: `https://api.service-b`,
      headers: { 'Authorization': `${bearerToken}` },
      // under the hood Cypress retries the initial request
      retryOnStatusCodeFailure: true 
      // Cypress also retries for transient network errors
      retryOnNetworkFailure: true 
      // only fail if it takes 10 seconds, if shorter then keep retrying
    }, { timeout: 10000 }) 
      .its('length') // the data we are expecting at service B
      .should('be.greaterThan', 0); // will retry the assertion for 10 seconds
  });
```

The above pattern by itself takes care of majority of the API testing problems in an event driven system. There are also advanced techniques with plugins like [cypress-recurse](https://www.npmjs.com/package/cypress-recurse) and [cypress-wait-until](https://www.npmjs.com/package/cypress-wait-until) that can be used to handle more difficult scenarios.

```javascript
// assume cy.getToken() is a function that yields the auth token

/** the API request to GET and yield all the items at Service B */
const getServiceBItems = () =>
  cy.getToken().then((bearerToken) =>
    cy
      .request({
        method: 'GET',
        url: 'https://api.service-b',
        headers: {
          Authorization: `${bearerToken}`
        },
        retryOnNetworkFailure: true,
        retryOnStatusCodeFailure: true,
        timeout: 10000
      })
      .its('body')
  );

/** gets service B items, filters the list for the item we want */
const itemNeeded = (itemId) =>
  getServiceBItems().then(
    (itemList) =>
      itemList
        .filter((serviceDB) => serviceDB.item.id === `${itemId}`)
        .map((arr) => arr.item[0] 
  );

it('your very complex API test', () => {
  // Assume Arrange & Act already happened in this test
  // (1. Send a POST request to service A, we get a 200)

  // Assert: assume that we expect a change at service B because of this
  // (2. Now we are sending a GET to service B to verify this change)

  // hypothetically there is more logic needed in this test
  // so we are using cypress-recurse
  recurse( 
    // a pure function that yields a value
    () => itemNeeded(itemXYZ),
    // the predicate that we keep retrying for
    (item) => item === itemXYZ,  
    { // until optional configurations
      log: true,  // logs details in the Cypress runner
      limit: 100, // max number of iterations before failing
      timeout: 30000, // time limit in ms     
      delay: 3000 // delay before next iteration
    }
  );
});
 
```

Let me know your thoughts about API testing tools, perhaps there are new ones I haven't yet heard about, and they allow such techniques. 