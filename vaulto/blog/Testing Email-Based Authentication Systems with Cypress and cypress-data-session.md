#  Testing Email-Based Authentication Systems with Cypress Mailosaur and `cypress-data-session`

These days our inboxes frequently buzz with verification codes, organization invites, or one-time links, each constituting a variant of authentication security. How can we ensure these email processes are robust? In this post we will cover test strategies for email authentication, using an open source application ([link to the repo](https://github.com/muratkeremozcan/prod-ready-serverless)) as well as real life examples of how there could be variants to the same strategy. We will also touch upon how to reduce redundant consumption of such emails  using [`cypress-data-session`](https://github.com/bahmutov/cypress-data-session). 

TOC:

- [Receiving authentication codes - AWS Cognito example](#receiving-authentication-codes---aws-cognito-example)
- [Gleb's email-authentication-code example at Mercari](#glebs-email-authentication-code-example-at-mercari)
- [User invite at Extend \& passwordless-login example](#user-invite-at-extend--passwordless-login-example)
- [Reducing redundant email consumption with `cypress-data-session`](#reducing-redundant-email-consumption-with-cypress-data-session)
- [Sharing the email between it blocks](#sharing-the-email-between-it-blocks)
- [Using one email per machine](#using-one-email-per-machine)
- [Details about the data-session logic (specific to repo example)](#details-about-the-data-session-logic-specific-to-repo-example)
- [Magic links with `cypress-data-session`](#magic-links-with-cypress-data-session)
- [Wrap-up](#wrap-up)

## Receiving authentication codes - AWS Cognito example

Our [sample app](https://github.com/muratkeremozcan/prod-ready-serverless) is from Yan Cui's highly acclaimed [Production-Ready Serverless](https://productionreadyserverless.com/) workshop. Yan guides us through building an event driven serverless app in NodeJs with AWS API Gateway, Lambda, Cognito, EventBridge, DDB, and server-side rendering, which uses AWS Cognito for authentication. 

This application showcases an exemplary user registration flow, where the user we create receives an email from AWS Cognito containing a one-time code. This code then grants the user access to the application. The following sign-ins only require an email and a password. Below are the visual steps that should be familiar to all users.

![img](https://files.cdn.thinkific.com/file_uploads/179095/images/85e/d53/0b1/mod09-002.png)

![img](https://files.cdn.thinkific.com/file_uploads/179095/images/9fb/92c/4d6/mod09-003.png)

![img](https://files.cdn.thinkific.com/file_uploads/179095/images/0c8/e1c/b10/mod09-004.png)

![img](https://files.cdn.thinkific.com/file_uploads/179095/images/f8b/e8b/7fb/mod09-005.png)

![img](https://files.cdn.thinkific.com/file_uploads/179095/images/1bb/d94/410/mod09-006.png)

Let us take a look at the automated test at [sign-up-new-user](https://github.com/muratkeremozcan/prod-ready-serverless/blob/main/cypress/e2e/front-end/sign-up-new-user.cy.js).

We use a utility to randomize a user and visit the app. [The example](https://github.com/muratkeremozcan/prod-ready-serverless/blob/main/cypress/support/generate-random-user.js#L3) uses Chance library, but a FakerJs version is also provided. The only noteworthy item here is the randomized password. If the randomized password is not compliant, we might get 400 errors.

```js
import {getConfirmationCode} from '../../support/e2e'
import {generateRandomUser} from '../../support/generate-random-user'

describe('sign up a new user', () => {
  it('should register the new user and log in', () => {
    const {firstName, lastName, userName, email, password} = generateRandomUser(
      Cypress.env('MAILOSAUR_SERVERID'),
    )
    
    cy.visit('/')

    // ... the rest of the test
  })
})
```

In the next section we drive the UI to fill in the form and register the user. It is important to check the backend response here, as the UI is not sophisticated to show invalid password or user already existing errors. In the real world, we would both verify the backend response and the UI, and possibly test the error/edge cases by stubbing the network via `cy.intercept` preferably in lower level component tests.

```js
 
  cy.intercept('POST', 'https://cognito-idp*').as('cognito')
  cy.contains('Register').click()
  cy.get('#reg-dialog-form').should('be.visible')
  cy.get('#first-name').type(firstName, {delay: 0})
  cy.get('#last-name').type(lastName, {delay: 0})
  cy.get('#email').type(email, {delay: 0})
  cy.get('#username').type(userName, {delay: 0})
  cy.get('#password').type(password, {delay: 0})
  cy.contains('button', 'Create an account').click()
  cy.wait('@cognito').its('response.statusCode').should('equal', 200)

```

In the next section we check the email with a helper function `getConfirmationCode` which is using [Mailosaur](https://mailosaur.com/)'s Cypress commands (details later), extract the verification code from the email, and continue driving the user until the user is signed in.

```js

getConfirmationCode(email).then(code => {
  cy.get('#verification-code').type(code, {delay: 0})
  cy.contains('button', 'Confirm registration').click()
  cy.wait('@cognito')
  cy.contains('You are now registered!').should('be.visible')
  cy.contains('button', /ok/i).click()

  cy.contains('Sign in').click()
  cy.get('#sign-in-username').type(userName, {delay: 0})
  cy.get('#sign-in-password').type(password, {delay: 0})
  cy.contains('button', 'Sign in').click()
  cy.wait('@cognito')

  cy.contains('Sign out')
})
```

The interesting part here is receiving the email and extracting the confirmation code from the email via regex.
Receiving the email is near-instant and effortless via [Mailosaur](https://mailosaur.com/). We accomplish that with the command from [cypress-mailosaur](https://github.com/mailosaur/cypress-mailosaur) plugin `cy.mailosaurGetMessage(<yourEmailServerID>, {sentTo: <yourEmail>})`. 

Remember that the email is just a string with the confirmation code. We can extract the code using a regular expression.

![img](https://files.cdn.thinkific.com/file_uploads/179095/images/9fb/92c/4d6/mod09-003.png)

```js
const parseConfirmationCode = str => {
  const regex = /Your confirmation code is (\w+)/
  const match = str.match(regex)

  return match ? match[1] : null
}

export const getConfirmationCode = userEmail => {
  return cy
    .mailosaurGetMessage(Cypress.env('MAILOSAUR_SERVERID'), {
      sentTo: userEmail,
    })
    .its('html.body')
    .then(parseConfirmationCode)
}
```

### Addendum: [Mailosaur](https://mailosaur.com/) makes parsing confirmation codes effortless!

We found about this later, but when looking at the `html.codes` property, we see that the Mailosaur already extracts the codes for us. That is so convenient because the regex may be different between authentication providers. The previous code becomes so much easier without having to worry about the regex.

```js
export const getConfirmationCode = userEmail => {
  return cy
    .mailosaurGetMessage(Cypress.env('MAILOSAUR_SERVERID'), {
      sentTo: userEmail,
    })
    .its('html.codes.0.value')
}
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rr33zt9whje59trkdpoj.png)

## Gleb's email-authentication-code example at Mercari 

In the AWS Cognito example we only needed the verification code so that we can enter it in the UI.

In my friend [Gleb Bahmutov's blog](https://glebbahmutov.com/blog/minimize-mailosaur-use/), the case study at Mercari is a similar variant, receiving a confirmation code and performing further testing on the email. We randomize a user, fill the form, get the email, and display the email at our test runner, which is a very neat trick because it can allow us to visualize the email as we are executing the test.

```js
beforeEach(() => {
  const userName = 'Joe Bravo'
  const serverId = Cypress.env('MAILOSAUR_SERVER_ID')
  const randomId = Cypress._.random(1e6)
  const userEmail = `user-${randomId}@${serverId}.mailosaur.net`

  // fill the form...
  
  // get the email
  cy.mailosaurGetMessage(serverId, {
    sentTo: userEmail,
  })
    .then(console.log)
    .its('html.body')
    // store the HTML under an alias
    .as('email)
})

beforeEach(function () {
  cy.document({ log: false }).invoke({ log: false }, 'write', this.email)
})
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yn6nugrt07xlrhvqekfk.png)

## User invite at [Extend](https://www.extend.com/) & passwordless-login example

At our company Extend, in one of our applications we have a user being invited by an admin to an organization. Simply, the user receives an email, then follows the link where they create an account and continue to sign on to the app & the org.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vobq496okbgvf3fgv55v.png)

In the test, the interesting part is extracting the href and visiting it. The browser built-in DOMParser Provides the ability to parse XML or HTML source code.

```ts
/**
 * Extracts the href value from an HTML string.
 *
 * @param {string} htmlString - The HTML string to parse.
 * @returns {string|null} The href value, if it exists, otherwise null.
 */
const extractHref = (htmlString: string): string | null => {
  const parser = new DOMParser() 
  const doc = parser.parseFromString(htmlString, 'text/html')
  const link = doc.querySelector('#reset-password-link')
  return link ? (link as HTMLAnchorElement).href : null
}
```

The htmlString is simply the `html.body` we receive in the email. The function chain yields the link that we visit.

```ts
// as the admin, fill the form with the user's info

// the user receives the email
  cy.mailosaurGetMessage(Cypress.env('MAILOSAUR_SERVERID'), {
    sentTo: email,
  })
    .its('html.body')
    .then(extractHref)
    .should('exist')
    .then(cy.visit)

// the user continues creating the account
```

In a very similar fashion, in the case of a password-less login, we could receive a magic link in the email, extract and visit it. We fathom even the test code would be the same.

```ts
// we randomize a user, they receive a magic link

const extractHref = (htmlString: string): string | null => {
  const parser = new DOMParser() 
  const doc = parser.parseFromString(htmlString, 'text/html')
  const link = doc.querySelector('#reset-password-link')
  return link ? (link as HTMLAnchorElement).href : null
}

// we check the email, extract the link and visit it
cy.mailosaurGetMessage(Cypress.env('MAILOSAUR_SERVERID'), {
  sentTo: email,
})
  .its('html.body')
  .then(extractHref)
  .should('exist')
  .then(cy.visit)

// we are logged in to the app
```

### Addendum: Mailosaur makes parsing links effortless!

Similar to not having to parse the regex, we later found out that we do not have to parse the links either. After all, confirmation codes and links are widespread, and Mailosaur already provides this data in the email response. Such convenience!

```ts
cy.mailosaurGetMessage(Cypress.env('MAILOSAUR_SERVERID'), {
  sentTo: email,
})
  .its('html.links.0.href')
  .should('exist')
  .then(cy.visit)
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vayhol1af0sadmebig43.png)

## Reducing redundant email consumption with [`cypress-data-session`](https://github.com/bahmutov/cypress-data-session)

In the sample repo [prod-ready-serverless](https://github.com/muratkeremozcan/prod-ready-serverless), we are using AWS Cognito which has a [soft limit of 50 emails per day](https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html). Not only there are limits that are not-easy to increase, but also there is a cost to Cognito. Similarly at Extend, we have limits with Okta. In one of the UI apps we have 80 e2e tests and ~500 it blocks, and these can fully execute at a pull request. We may have dozens of commits per day. If every test created a new user via email, even if we had a generous Mailosaur pricing band, email based authentication cannot scale. We would be blocked by our auth provider long before reaching the email limit. Gleb Bahmutov's `cypress-data-session` plugin is the solution. 

## Sharing the email between it blocks

In Gleb's example, we see data session being utilized across the it blocks in a spec, in order to prevent an email being received for each it block. The random user is created in the beginning, the form is filled, the email is received only once. The return value from the `setup` function is the `html.body` which is auto-aliased to the session name "email". The DOM is populated with the email in each test.

```js
beforeEach(() => {
  cy.dataSession({
    name: 'email',
    setup() {
      const userName = 'Joe Bravo'
      const serverId = Cypress.env('MAILOSAUR_SERVER_ID')
      const randomId = Cypress._.random(1e6)
      const userEmail = `user-${randomId}@${serverId}.mailosaur.net`

      // fill the form...

      cy.mailosaurGetMessage(serverId, {
        sentTo: userEmail,
      })
        .its('html.body')
    },
    shareAcrossSpecs: true, // the email is reused between the tests
  })
})

beforeEach(function () {
  cy.document({ log: false }).invoke({ log: false }, 'write', this.email)
})

it('shows the code by itself', () => ...)
it('has the confirmation code link', () => ...)
it('has the working code', () => ...)
```

![Reusing the same email in all tests](https://glebbahmutov.com/blog/images/minimize-mailosaur-use/m3.png)

In the `prod-ready-serverless` application -we will share the details of the data-session configuration in the [`registerAndSignIn`](https://github.com/muratkeremozcan/prod-ready-serverless/blob/main/cypress/support/e2e.js#L720) command later- we can contemplate about how a test would look if we would use one email per file. The first test in the suite would go through the complete flow of filling the form, receiving the one time code, using it to sign in. The 2nd test would only sign in. The spec would consume 1 email, similar to the previous example.

```js
// ./cypress/e2e/front-end/place-order.cy.js
const {generateRandomUser} = require('../../support/generate-random-user')

describe('place an order', () => {
  // randomize a user
  const {fullName, userName, email, password} = generateRandomUser(
    Cypress.env('MAILOSAUR_SERVERID'),
  )
  
  beforeEach(() => {
    cy.visit('/')

    cy.registerAndSignIn({
      fullName,
      userName,
      email,
      password,
    })
  })

  // the first test registers, receives the email and signs in
  it('should place an order', () => {
    cy.on('window:alert', cy.stub().as('alert'))
    cy.intercept('POST', '**/orders').as('placeOrder')

    cy.get('#restaurantsUl > :nth-child(1)').click()
    cy.wait('@placeOrder').its('response.statusCode').should('eq', 200)
    cy.get('@alert').should('be.calledOnce')
  })
  
	// the 2nd test only signs in
  it('should do something else', () => {
    cy.log('we are logged in with the same user')
  })
})

```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lul7r7cgqs0eppbusza4.png)

## Using one email per machine

50 emails with Cognito is not a very safe limit. In our repo we want to further reduce the email consumption, and share the random user, therefore the email, between all the tests that execute on that machine. 

The way to accomplish this is to create the random user at the [`cypress.config`](https://github.com/muratkeremozcan/prod-ready-serverless/blob/main/cypress.config.js) file, which executes once per `cy:run` or `cy:open` . We can assign the values as environment variables, and use them as arguments when calling the `registerAndSignIn` command, ensuring the same values are used not only between the it blocks, but the spec files as well. 

```js
const {fullName, userName, email, password} =
  generateRandomUser(MAILOSAUR_SERVERID)

module.exports = defineConfig({
  // ...
  env: {
    fullName,
    userName,
    email,
    password,
  },
  e2e: {
    // ...
  },
})
```

We get the same effect at the single test file from before.

```js
// ./cypress/e2e/front-end/place-order.cy.js

describe('place an order', () => {
  beforeEach(() => {
    cy.visit('/')

    cy.registerAndSignIn({
      fullName: Cypress.env('fullName'),
      userName: Cypress.env('userName'),
      email: Cypress.env('email'),
      password: Cypress.env('password'),
    })
  })

  it('should place an order', () => {..})

  it('should do something else', () => {..})
})

```

The key distinction is that other spec files utilize the same user and email as well.
```js
// cypress/e2e/front-end/sign-up-test-user.cy.js
describe('sign in with test user', () => {
  it('should register & log in with the test user', () => {
    cy.visit('/')

    cy.registerAndSignIn({
      fullName: Cypress.env('fullName'),
      userName: Cypress.env('userName'),
      email: Cypress.env('email'),
      password: Cypress.env('password'),
    })
  })
})
```

The idea of sharing the test user between different spec files applies to magic-link/passwordless login scenarios as well. Mirroring the above, we would randomize the user & assign the values as environment variables upon launching Cypress, then in the specs we would use the environment variables for the command arguments.

## Details about the data-session logic (specific to repo example)

One thing is for certain; with email based authentication, we cannot utilize the built-in  [`cy.session`](https://docs.cypress.io/api/commands/session#__docusaurus_skipToContent_fallback) because the email portion of the flow requires more control over the logic compared to a UI login not being repeated. `cypress-data-session` can be applied to login scenarios that `cy.session` would satisfy, but not the other way around. For a comparison of cypress-data-session vs cy.session, take a look at [this repo](https://github.com/muratkeremozcan/appsyncmasterclass-frontend/blob/main/cypress/support/e2e.js#L68) and [this video](https://www.youtube.com/watch?v=NT-Zjj0fQMQ).

The way to configure the data-session logic will be different in any app. In Gleb's example, the data-session was focused on the email, receiving one email, rendering it in the DOM and performing assertions on it. In the [`prod-ready-serverless`](https://github.com/muratkeremozcan/prod-ready-serverless/blob/main/cypress.config.js) example we have to satisfy 2 concerns:

- If it's a new user, we have to go through the registration flow (fill form, receive one time code, login with the code)
- If the user already exists, only sign in

Let us start breaking apart the functions and build up to the final `registerAndSignIn` command.

We have a function that simply drives the UI to fill the form for a new user.

```js
// first part of registration
const fillRegistrationForm = ({fullName, userName, email, password}) => {}
```

We have a function to get the confirmation code, using it one time at the UI, yielding out the confirmation code.

```js
// second part of registration
const confirmRegistration = email =>
  getConfirmationCode(email).then(confirmationCode => {
    cy.intercept('POST', 'https://cognito-idp*').as('cognito')
    cy.get('#verification-code').type(confirmationCode, {delay: 0})
    cy.contains('button', 'Confirm registration').click()
    cy.wait('@cognito')
    cy.contains('You are now registered!').should('be.visible')
    cy.contains('button', /ok/i).click()
    return cy.wrap(confirmationCode)
  })
}
```

We have a function that combines the above two parts for the full registration.

```js
// the registration flow; the above 2 parts
const register = ({fullName, userName, email, password}) => {
  fillRegistrationForm({fullName, userName, email, password})
  return confirmRegistration(email)
})
```

Another function that drives the UI, for sign in. 

```js
// nothing interesting, just driving the UI
const signIn = ({userName, password}) => {}
```

The final command makes clear why we had to separate `register` into two. 

Initially we want the full flow of filling out the form,  getting the confirmation and signing in; `init()` does the form fill and receives the code,  and `recreate()` signs in.

In subsequent tests, we only want to check if the email & verification code exist, and  sign in;  `setup()` checks the email & verification code, `recreate()` signs in.

```js

const registerAndSignIn = ({fullName, userName, email, password}) =>
  cy.dataSession({
    // unique name of the data session will be the email address. 
    // With any new email address, the data session will be recreated
    name: email, 
    // initially, we do the full registration flow
    init: () => register({fullName, userName, email, password}),
    // subsequent tests start from setup, just checking the email
    setup: () => confirmRegistration(email),
    // recreate runs always, either after init or setup
    recreate: () => signIn({userName, password}),
    shareAcrossSpecs: true,
  })
Cypress.Commands.add('registerAndSignIn', registerAndSignIn)
```

## Magic links with `cypress-data-session`

How could the above example look for a magic link scenario, where we do not want to keep receiving an email? 

After the setup step where we `visitEmailLink`, we need some logged-in state to be preserved for a of subsequent tests. This state could be local storage, session storage, and or cookies; whatever we need to be logged in after clicking the magic link.

For instance, in [this example](https://github.com/muratkeremozcan/appsyncmasterclass-frontend/blob/main/cypress/support/e2e.js#L65) our setup step logs in an returns the localstorage. That return value is fed-in as the argument to recreate step, and the recreate step rebuilds the localstorage.

```js
// preserving local storage example from 
// https://github.com/muratkeremozcan/appsyncmasterclass-frontend/blob/main/cypress/support/e2e.js#L65

Cypress.Commands.add('progLogin', (username, password) => {
  cy.then(() => Auth.signIn(username, password)).then(cognitoUser => {
    const idToken = cognitoUser.signInUserSession.idToken.jwtToken
    const accessToken = cognitoUser.signInUserSession.accessToken.jwtToken
    const makeKey = name =>
      `CognitoIdentityServiceProvider.${cognitoUser.pool.clientId}.${cognitoUser.username}.${name}`
    cy.setLocalStorage(makeKey('accessToken'), accessToken)
    cy.setLocalStorage(makeKey('idToken'), idToken)
    cy.setLocalStorage(
      `CognitoIdentityServiceProvider.${cognitoUser.pool.clientId}.LastAuthUser`,
      cognitoUser.username,
    )
  })
  cy.saveLocalStorage()
  // IMPORTANT: preserving the localstorage state
  // this could be session storage, or cookies 
  return cy.visit('/home').then(() => JSON.parse(JSON.stringify(localStorage)))
})

Cypress.Commands.add('dataSessionLogin', (email, password) => {
  return cy.dataSession({
    name: email,
    setup: () => cy.progLogin(email, password),
    validate: validateLocalStorage,
    // the return of the setup step is yielded as an arg to recreate
    // and we rebuild the localstorage
    recreate: ls => {
      for (const key in ls) {
        localStorage[key] = ls[key]
      }
      cy.visit('/home')
      return cy.contains('Home', {timeout: 10000})
    },
    cacheAcrossSpecs: true,
  })
})
```

Here's how the magic link scenario could work:

```js
const visitEmailLink = (email) => {
  // we randomize a user, they receive a magic link
  const extractHref = (htmlString: string): string | null => {
    const parser = new DOMParser() 
    const doc = parser.parseFromString(htmlString, 'text/html')
    const link = doc.querySelector('#reset-password-link')
    return link ? (link as HTMLAnchorElement).href : null
  }

  // we check the email, extract the link and visit it
  cy.mailosaurGetMessage(Cypress.env('MAILOSAUR_SERVERID'), {
    sentTo: email,
  })
    .its('html.body')
    .then(extractHref)
    .should('exist')
    .then(cy.visit)
  
  // IMPORTANT: preserve some state here, and return that
  // could be localstorage, sessionstorage, and/or cookies
  return cy.then(() => JSON.parse(JSON.stringify(localStorage)))
}

cy.dataSession({
  // we can again make the unique email the session
  name: email,
  // something that runs the first time,
  // causes an email to be sent
  init: () => {..},
  // exract the magic link and visit it
  // in this function, make sure to preserve some state
  // localstorage, sessionstorage, and/or cookies
  setup: () => visitEmailLink(email),
  // the return value of the setup gets fed to recreate as an arg
  recreate: ls => {
    // rebuild the preserved state before visiting
    for (const key in ls) {
      localStorage[key] = ls[key]
    }  
  cy.visit('/')
  }
})
```

## Wrap-up

We examined the use of Cypress, a powerful testing tool, and [Mailosaur](https://mailosaur.com/), a service designed specifically for testing emails, as a potent combination for testing email based authentication.

We  delved into methods to extract required data from emails, either by using Regular Expressions or directly pulling href values from the email's HTML content. These techniques provide the flexibility to extract almost any information that might be needed from an email for testing purposes.

Lastly, we looked at how to use `cypress-data-session` to minimize redundant email consumption. This plugin not only makes tests more efficient by eliminating unnecessary email retrievals but also enables sharing of email content within the same test, or even across tests running on the same machine.

Testing is never an easy task, but with these tools and strategies, even complex systems like email-based authentication can be effectively and efficiently validated.

