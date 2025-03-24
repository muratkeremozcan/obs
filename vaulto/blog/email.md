# Email Testing

> A blog post is lackluster without working code you can play around with in your own environment. All the code samples in this post can be found [here](https://github.com/muratkeremozcan/cypressExamples/tree/master/cypress-mailosaur).

Email testing is [critical for business success](https://www.industrialmarketer.com/why-email-testing-is-critical-for-email-marketing-success/) and [boosts email performance](https://litmus.com/blog/3-reasons-why-email-testing-boosts-email-performance). It is not something we want to forego while testing our web applications because modern email services allow painless automated email testing. The core goal is to enable the last mile of end to end testing, to enable a typical web app to be tested from start to finish.

For example imagine a scenario where a user starts having received an email invite from an organization, through company proprietary services or third party such as LinkedIn invitations. Then, the user verifies email content, accepts the invite, and joins the organization.
Later, the user can leave the organization - or get removed by an administrator - then receives another email notification.
Using an email service, the entirety of this requirement is possible to automate and execute within seconds.

That being stated, email testing is a fundamental enabler for SaaS test architectures by permitting stateless tests that can scale; tests that independently handle their state and can be executed by _n_ number of entities at the same time.

- [Email Testing](#email-testing)
	- [What to test in an email](#what-to-test-in-an-email)
	- [Why you do not want to use the "+" trick](#why-you-do-not-want-to-use-the--trick)
	- [Speed reduces the cost](#speed-reduces-the-cost)
	- [What do we need from an email service?](#what-do-we-need-from-an-email-service)
	- [How to test emails](#how-to-test-emails)
	- [Achieving Stateless tests with unique emails](#achieving-stateless-tests-with-unique-emails)
	- [Setup for this case study](#setup-for-this-case-study)
		- [Environment variables](#environment-variables)
	- [Older, difficult approaches](#older-difficult-approaches)
	- [The overhead is abstracted away with Cypress Mailosaur plugin](#the-overhead-is-abstracted-away-with-cypress-mailosaur-plugin)
	- [Data focused assertions with cy-spok](#data-focused-assertions-with-cy-spok)
	- [Update: Email Previews](#update-email-previews)

## What to test in an email

Typically email testing involves :

- validating email fields; from, to, cc, bcc, subject, attachments.

- HTML content and links in the email.

- Spam checks and maybe visual checks.

## Why you do not want to use the "+" trick

Stateless tests that can scale are a necessity in any modern web application testing. We want tests that independently handle their state and tests that can be executed by _n_ number of entities at the same time.

While testing SaaS applications, which generically have Subscriptions, Users, Organizations (ex: [Slack](https://slack.com/intl/en-sk/help/articles/115004071768-What-is-Slack-#your-team-in-slack), [Cypress Dashboard Service](https://dashboard.cypress.io/organizations), etc.) a lot of the end to end workflows can rely on having **unique users**. Otherwise only one test execution can happen at a time and they clash with other simultaneous test executions. This constraint reduces test automation to cron jobs or semaphores; testing hardware is a classic example.

Some ways to address unique users is by utilizing [Gmail tricks](https://www.idownloadblog.com/2018/12/19/gmail-email-address-tricks/) or [AWS Simple Email Service](https://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-email-simulator.html). It is possible that you do not have to check actual email content (from, to, cc, bcc, subject, attachments etc.) and only want to have unique users, then you are on the right path with stateless tests and this is enough for your current needs.

However, these approaches are generally problematic for email testing:

- Non-existing emails can prompt bouncing emails to your cloud service and that can be a headache.
- Too many emails can flag the email for spam; think load testing or many e2e tests running per day.
- With the + trick, if there is even a way to check the email content, there is no way to differentiate between emails being received. We end up in a situation where every spec has a unique email, which defeats statelessness; we're locked into cron jobs or semaphores.
- Unreliable email speeds, which add up in CI costs, and more importantly engineer feedback time.

If you want to avoid such issues and check real email content in automation, email services provide value.

## Speed reduces the cost

Email services can also provide cost savings in test execution time by receiving emails faster, tests running quicker in the pipeline with less CI resources being consumed and less time waiting for tests to finish. If you are running 1000 pipelines a year, and save 3-4 seconds per pipeline execution, the email service can be already paying for its annual subscription just by providing the extra speed.

## What do we need from an email service?

We need a few things from an email service for test automation:

- Unique email servers per service and app so that there is a predictable inbox.
- Being able to create (unlimited) users on the fly and have emails sent to them.
- Receiving the emails fast.
- Being able to verify the content of those emails effortlessly.

## How to test emails

There are plenty of [email testing solutions](https://www.g2.com/search/products?max=10&query=email+testing) available, and combinations of test frameworks that integrate with them.
For the code snippets and working examples, we will be using [Cypress](https://www.cypress.io/) and [Mailosaur](https://mailosaur.com/), but the ideas should generally apply to any tuple of email services and test automation frameworks.

When using Cypress with Mailosaur, there are 3 test-development approaches:

1. Implement [Mailosaur API]([https://mailosaur.com/docs/api](https://mailosaur.com/docs/api)) using Cypress API testing capabilities using [`cy.request()`](https://docs.cypress.io/api/commands/request.html#Syntax) or [`cy.api()`](https://github.com/bahmutov/cy-api). Utilize plugins and helper utilities to construct test suites. This was the original approach a long time ago. The test spec can be found [here](https://github.com/muratkeremozcan/cypressExamples/blob/master/cypress-mailosaur/cypress/integration/1.with-waituntil-cypress.spec.js).

2. Utilize [Mailosaur's Node package](https://www.npmjs.com/package/mailosaur) and implement them using [`cy.task()`](https://docs.cypress.io/api/commands/task.html#Syntax) which allows running node within Cypress. This is a easier and cleaner approach than (1), and it became available as soon as the npm package was released. The test spec can be found [here](https://github.com/muratkeremozcan/cypressExamples/blob/master/cypress-mailosaur/cypress/integration/2.with-npm-package-and-cy-task.spec.js).

3. Use the [Cypress Mailosaur plugin](https://www.npmjs.com/package/cypress-mailosaur) and abstract away all the complexity! The test spec can be found [here](https://github.com/muratkeremozcan/cypressExamples/blob/master/cypress-mailosaur/cypress/integration/3.best-with-mailosaur-cypress-plugin.spec.js).

> If using the sample repo 3 weeks after the date of this post, note that you will have to start a new Mailosaur trial account and replace environment variables in `cypress.json` for yourself.

## Achieving Stateless tests with unique emails

If in every test execution a new, unique user was used and the emails to this unique user could be verified in isolation, it would be possible to achieve a stateless test. The only side effect would be to the email service inbox, but if the test only checked emails by reference and cleaned up after itself, the email service mailbox would not be impacted.

This is easy to achieve with Mailosaur, here are two approaches to this: [Mailosaur's Node package](https://www.npmjs.com/package/mailosaur) or our own util with `faker.js`.

Note that these emails are unlimited; the term _Users_ at [Mailosaur Pricing Page](https://mailosaur.com/pricing/) refers to the number of employees at your company accessing the Mailosaur web interface.

```javascript
// at cypress/plugins/mailosaur-tasks.js

// generates a random email address
// sample output:   ojh788.<serverId>@mailosaur.io
const createEmail = () => mailosaurClient
  .servers
  .generateEmailAddress(envVars.MAILOSAUR_SERVERID);
);

// our custom function at a helper file or commands file.
// The only difference is the defined prefixed name.
// sample output:  fakerJsName.<serverId>@mailosaur.io
const createMailosaurEmail = randomName =>
  `${randomName}.${Cypress.env('MAILOSAUR_SERVERID')}@mailosaur.io`;
```

## Setup for this case study

The only setup we need is a way for emails to be sent to our server. For this, the sample repo is using [`sendmail` npm package](https://www.npmjs.com/package/sendmail) . Any tool can be used here for the case study. Note that this one needs node 12, therefore an `.nvmrc` file has been included.

### Environment variables

We recommend these values as environment variables in the `cypress.json` file `"env"` property. You can grab them from Mailosaur web application by creating a free trial account using any email. The trial account lasts for two weeks.

> If you are using the referenced repo within three weeks of this post, no need to do anything here.

```json
  "MAILOSAUR_SERVERID": "******",
  "MAILOSAUR_PASSWORD": "******",
  "MAILOSAUR_API_KEY": "*******",
  "MAILOSAUR_API": "https://mailosaur.com/api",
  "MAILOSAUR_SERVERNAME": "user-configurable-server-name"
```

## Older, difficult approaches

We will cover the latest and greatest way with Cypress Mailosaur test plugin in this guide. For the older approaches, the evolution over time, the code has been shared and you can also check out an [older version of this post](https://github.com/NoriSte/ui-testing-best-practices/blob/master/sections/advanced/email-testing.md#modularizing-cytask) for a read through.

## The overhead is abstracted away with [Cypress Mailosaur plugin](https://www.npmjs.com/package/cypress-mailosaur)

Mailosaur team released a Cypress plugin in mid 2020. With it, we do not have to replicate any complex API utities or use cy.task via Mailosaur npm package. There is no need to create cy.task utilities or even hybridize them. With the Cypress Mailosaur plugin, you can just use the custom Cypress commands Mailosaur team created for us.

All we need is to install the package `npm install cypress-mailosaur --save-dev` and add the following line to cypress/support/index.js:
`import 'cypress-mailosaur'`. The new kid on the block is `cy-spok`, and that will make our assertions a bliss. `npm i cy-spok -D` that too, and we only need it imported at the top spec files with `import spok from 'cy-spok'`.

Mailosaur plugin has [a few handy functions](https://github.com/mailosaur/cypress-mailosaur) which help you abstract complex needs. Some of our favorites are bolded.

- mailosaurListServers()
- mailosaurCreateServer({ name })
- mailosaurGetServer(serverId)
- mailosaurUpdateServer(serverId, server)
- mailosaurDeleteServer(serverId)
- mailosaurListMessages(serverId)
- mailosaurCreateMessage(serverId)
- **mailosaurGetMessage(serverId, criteria)**
- **mailosaurGetMessageById(messageId)**
- mailosaurSearchMessages(serverId, criteria, options)
- mailosaurGetMessagesBySubject(serverId, subjectSearchText)
- mailosaurGetMessagesByBody(serverId, bodySearchText)
- **mailosaurGetMessagesBySentTo(serverId, emailAddress)**
- **mailosaurDeleteMessage(messageId)**
- mailosaurDeleteAllMessages(serverId)
- mailosaurDownloadAttachment(attachmentId)
- mailosaurDownloadMessage(messageId)
- **mailosaurGetSpamAnalysis(messageId)**

With the Cypress Mailosaur plugin, we do not have to implement any `cy.task()` utilities, custom helper functions or hybrid helpers.

You can find a working version of this code [here](https://github.com/muratkeremozcan/cypressExamples/blob/master/cypress-mailosaur/cypress/integration/3.best-with-mailosaur-cypress-plugin.spec.js). Here are a few approaches with various commands, and basic Cypress assertions.

```javascript
// an email has been sent in the before hook
it("uses the mailosaur cypress plugin to check the email content", () => {
  cy.mailosaurListMessages(Cypress.env("MAILOSAUR_SERVERID"))
    .its("items")
    .its("length")
    .should("not.eq", 0);

  cy.log("get message");
  cy.mailosaurGetMessage(
    Cypress.env("MAILOSAUR_SERVERID"),
    { sentTo: userEmail }
    // note from Jon at Mailosaur:
    // The get method looks for messages received within the last hour
    // if looking for emails existing before that, you have to add this. Optional otherwise
    // { receivedAfter: new Date('2000-01-01') }
  ).then((emailContent) => {
    cy.wrap(emailContent)
      .its("from")
      .its(0)
      .its("email")
      .should("contain", "test@nodesendmail.com");
    cy.wrap(emailContent).its("to").its(0).its("email").should("eq", userEmail);
    cy.wrap(emailContent)
      .its("subject")
      .should("contain", "MailComposer sendmail");
  });

  cy.log("alternate approach to getting message by sent to");
  cy.mailosaurGetMessagesBySentTo(
    Cypress.env("MAILOSAUR_SERVERID"),
    userEmail
  ).then((emailItem) => {
    // the response is slightly different, but you can modify it to serve the same purpose
    const emailContent = emailItem.items[0];
    cy.wrap(emailContent)
      .its("from")
      .its(0)
      .its("email")
      .should("contain", "test@nodesendmail.com");
    cy.wrap(emailContent).its("to").its(0).its("email").should("eq", userEmail);
    cy.wrap(emailContent)
      .its("subject")
      .should("contain", "MailComposer sendmail");
  });

  cy.mailosaurGetMessagesBySentTo(Cypress.env("MAILOSAUR_SERVERID"), userEmail)
    .its("items")
    .its(0)
    .its("id")
    .then((messageId) => {
      cy.log("does convenient spam analysis");
      cy.mailosaurGetSpamAnalysis(messageId).its("score").should("be.gt", 0);
    });
});
```

## Data focused assertions with [cy-spok](https://github.com/bahmutov/cy-spok)

Let's take a look at the Mailosaur interface. We can add multiple servers here, limited by the subscription plan. We can have a unique server per app or service, and we can prefix random user names by the environment to make diagnostics easier.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uiet7rt98wyx52j3s9xh.png)

We can look at our Messages / Inbox. The email is from the test we executed via node mailer.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b1ksx79bey9zchm4hksu.png)

There isn't too much to look at in our email, just some html and text in it.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/cg7irbde5rzuj9ocv9ip.png)

On the Spam tab, our score is looking pretty bad; the spam analytics must be working correctly.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/67l0ssu61c2dvvv7wmq2.png)

The JSON tab is where the magic starts happening. It is a JSON representation of our email. Look how nicely it matches what we see in DevTools.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gnkrf7d8naf1asaz8zkv.png)
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gy9dq4trwlafkmxybcqh.png)

It would be so good if our test matched that data 1:1. This is where `cy-spok` comes in.

We will let the magic do the talking.

```js
it("uses cy-spok, matches email payload and verifies the data", () => {
  cy.mailosaurGetMessage(Cypress.env("MAILOSAUR_SERVERID"), {
    sentTo: userEmail,
  })
    .should(
      spok({
        from: [
          {
            email: "test@nodesendmail.com",
          },
        ],
        to: [
          {
            email: userEmail,
          },
        ],
        subject: "MailComposer sendmail",
        html: {
          body: (b) => expect(b).to.include("here is some text"),
        },
      })
    )
    .its("id")
    .then((messageId) => {
      cy.mailosaurGetSpamAnalysis(messageId).its("score").should("be.gt", 0);

      // commented out for testing
      // we should use it in CI so that the test is stateless
      // cy.mailosaurDeleteMessage(messageId)
    });
});
```

Email testing used to be an arduous task. Over the few years, with contributions from the community, it became effortless. This blog post and the referenced repository introduced you to email testing, and told a story of how it evolved. Equipped with this knowledge, now you can confidently implement modern, effortless, confident email testing solutions for your teams. Cheers!

## Update: Email Previews

When it comes to low-level automated email testing, Mailosaur makes it all effortless and robust. However, many companies require email testing for marketing, where the content needs to be **visually verified** in a variety of devices, browsers, versions and viewports. Some competitors in this domain are Email on Acid, Litmus, Taxi and Movable Ink. Mailosaur recently added the email preview feature, let's take a look. Assume an email is sent, either manually or during automation, and we do not clear the email box. Notice their new UX and _Email Previews_ on the right.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/va3ulowcn27t7ho0u897.png)

The total number of combinations we can check is over 70. That is some combinatorial explosion and it is great that they are supporting it all. Let's check some of the more probable ones.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/45qtwqljajlzzzyb8799.png)

With that we have 36 previews. We can view them almost immediately.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w1mr8plav9j5uasmqjs9.png)

Let's pick one of them, here's Gmail on Android dark mode. I can verify from my own device that this is spot on.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/x3kpwsrfmnyxpp7u2kpg.png)

Email Previews immediately fulfill the gap for email marketing needs of Mailosaur. **If you are using Mailosaur for automating email testing, and your marketing department needs a dedicated email marketing tool to visually verify the look and feel of the emails, a separate tool is not needed at this time.**

So, what is next? To the engineering mindset, email preview feature immediately brings up an idea; can we visually diff the emails and automate even our marketing tests? James from Mailosaur team let us know **_" at the moment Email Previews are Dashboard-only (i.e. no API functionality [like we have been used to with Mailosaur]) - but we're looking to add 'no code' functionality to allow you to monitor changes in visual appearance"_**. This sounds like the feature set for visual diffing snapshots with AI powered tools like Applitools and Percy and it would be exciting if some day it became a feature set.

Imagine how it could be configured:

- Configure a json, yml, or any language file to pick the visual check combinations.
- Have a command to record the initial run, for example `cy.previewEmail`.
- Record a default email preview of _n_ combinations and compare that default with the new, in subsequent email test executions. We would have to accept the baseline previews once at the Email Preview UI.
- From then on, new snapshots matching the default get auto-accepted.
- Non-matching new snapshots prompt a notification on the web interface, and or fail the automated visual diff email preview; we either have to reject or accept this new baseline. If we reject, it is a defect. If we accept we have a new base line and the cycle continues.

We are looking forward to what Mailosaur Team comes up with next!
