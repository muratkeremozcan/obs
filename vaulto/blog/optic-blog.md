# Documenting and Testing Schemas of Serverless Stacks with Optic & Cypress

Serverless testing is unique. Recently I went through [Yan Cui's Testing Serverless Architectures](https://testserverlessapps.com/?utm_source=blog_courses), highly recommended for all cloud engineers, learned a lot and agreed to these conclusions:

- **Increased Use of Managed Services**: With tools like AWS Lambda, there's a notable shift towards leveraging managed services.

- **Focused, Single-Purpose Functions**: Lambda functions typically hone in on a singular task, simplifying individual function complexity. With this structure, the main risk in serverless architectures emerges from integration points—how your Lambda functions communicate and interact with external managed services.

- **Granularity in Deployment**: Serverless promotes deploying in smaller units. While this granularity provides precise control over access, it also introduces complexities.This increased granularity inherently demands more configurations, heightening security measures and careful management. In turn, the potential for misconfigurations is increased—be it within the application or Identity and Access Management (IAM).

Given these distinct challenges and benefits of serverless architectures, our testing strategies must adapt.

- **Unit Testing**: Focused on individual objects or modules. These tests often don't yield a high return on investment unless the business logic is particularly intricate. Their value-to-cost ratio often leans unfavorably when compared to integration tests. They are useful for testing your business logic, particularly your branching logic, but they fall completely flat when testing your cloud infrastructure, making testing this code in a deployed environment essential.

- **Integration Testing**: Tests interaction with external components, like AWS resources. Integration is the same cost, and more value than unit; feed an event into the lambda handler, validate the consequences. Running these tests against deployed AWS resources locally is more practical and beneficial than attempting to emulate AWS in a local environment. Emulations are often brittle and labor-intensive. Temporary stacks are mandatory in this approach, albeit there are teams that can exercise their handlers without local simulation or remote AWS resources; all the better.

- **End-to-End (E2E) Testing**: Holistic testing of the system, including its deployment process. There are things integration tests cannot cover, such as IAM Permissions, our IaC/configuration, how the service is built and deployed. With e2e we get all that, alongside testing the functionality. While E2E tests offer the highest confidence, they're also the most resource-intensive.

Serverless architectures are increasingly prevalent, offering scalability and cost-efficiency. However, they also introduce the above unique testing challenges. Enter schema testing, a method that seems tailor-made for the serverless paradigm. Let's dive in and analyze how it can fit in to our serverless engineering & testing strategy in theory and practice.

- [Schema testing is a good fit with serverless applications/services/APIs](#schema-testing-is-a-good-fit-with-serverless-applicationsservicesapis)
- [Optic stands apart](#optic-stands-apart)
- [Optic vs Postman and Swagger](#optic-vs-postman-and-swagger)
- [How can it all work in a generic serverless application?](#how-can-it-all-work-in-a-generic-serverless-application)
  - [Install Optic to the project, and also globally.](#install-optic-to-the-project-and-also-globally)
  - [Generate your `openapi.yml` file](#generate-your-openapiyml-file)
  - [Create convenience scripts in `package.json`](#create-convenience-scripts-in-packagejson)
  - [One time setup to initialize the Optic (traffic) capture configuration](#one-time-setup-to-initialize-the-optic-traffic-capture-configuration)
  - [Setup Optic cloud](#setup-optic-cloud)
  - [Capture the http traffic using the Optic proxy](#capture-the-http-traffic-using-the-optic-proxy)
- [Questions, Concerns, other comparisons](#questions-concerns-other-comparisons)
  - [Why both e2e and schema test?](#why-both-e2e-and-schema-test)
  - [How about increased CI response time?](#how-about-increased-ci-response-time)
  - [Do we keep having to record the schema with `schema:verify` ?](#do-we-keep-having-to-record-the-schema-with-schemaverify-)
  - [Optic approach vs consumer driven contract testing (Pact)](#optic-approach-vs-consumer-driven-contract-testing-pact)
- [Conclusion](#conclusion)

## Schema testing is a good fit with serverless applications/services/APIs

Schema testing confirms an API's response structure, data types, and more, against a predefined schema, e.g., an OpenAPI specification. This kind of testing is useful for two main purposes; documenting the api and spot testing the implementation against potential breaking changes. It can give faster feedback in comparison to traditional e2e, as a preliminary test before heavy weight e2e tests.

Schema testing stands out as a fitting solution. Here's why:

- **Rapid feedback for integration integrity**: Schema testing swiftly validates Lambda functions against set structures and ensures communication with external services follows expected patterns.
- **Cost effective**: With minimal resource demands, schema tests sit in a cost effective spot, addressing the confidence gap between unit and E2E tests.
- **Guard against unintentional breaking changes**: In the face of potential misconfigurations in serverless architectures, schema testing indirectly confirms correct integrations and guards against unintentional breaking changes in evolving applications.

Equally as important, is documenting our schema / API for the consumers. Creating the API docs and keeping them up to date is usually a chore, but there are ways we can automate the documentation upkeep as well as the testing of the schema.

## Optic stands apart

We recently tried [Optic](https://www.useoptic.com/) and identified standout features that address the unique needs of serverless applications:

1. **Traffic Capture and Real-time Insights**: While optic offers a variety of  [methods to capture traffic](https://www.useoptic.com/docs/capturing-traffic), its ability to observe API traffic during testing through a reverse proxy mechanism streamlines API updates and verifications, akin to Jest snapshot test record and tests.
2. **Seamless Documentation Sync**: Any changes to the API automatically reflect in its documentation, ensuring alignment.
3. **API Lifecycle Management**:
   - **Lifecycle Awareness**: Optic's comprehensive understanding of your API's lifecycle allows it to detect breaking changes and oversee appropriate API versioning.
   - **Forward-Only Governance**: While new API endpoints comply with current standards, legacy endpoints remain intact, giving developers timely and pertinent feedback.
   - **API Version Control**: Much like Git's approach to code, Optic introduces version control for APIs.
4. **Enhanced Feedback and Integration**:
   - **Visual Diff View**: Optic presents intuitive visual depictions of changes in OpenAPI specifications.
   - **Integration with CI Pipelines**: Beyond the usual CI integrations, Optic delivers changelog previews directly within pull requests, elucidating API alterations.
   - **Git Integration**: Tightly integrated with Git, Optic allows API changes comparison across branches, aiding developers in monitoring developments across the entire project lifecycle.

By pairing Optic with e2e tests - we prefer Cypress for that because it is a great fit for [api e2e testing event driven systems](https://dev.to/muratkeremozcan/api-testing-event-driven-systems-7fe) - you can harness Optic's traffic capture to either generate or revise your OpenAPI specification.

## Optic vs Postman and OpenAPI/Swagger

Vs Postman:

- Postman is really focussed on their collections format and do not have good tooling for generating OpenAPI or automatically updating it when the API no longer matches the spec.
- Everything in Postman happens in their UI, whereas Optic is triggered in the local developer flows and CI. This gives developers the right feedback, in the right places.
- Optic has tools for API change review that live right in the CI pipeline, there is no comparison for this on Postman side.
- Optic and Postman both have API documentation viewers - every API tool does. Optic is optimized for internal teams + partner APIs and focused on showing consumers how the APIs they use change over time.

Vs Swagger

- Swagger is primarily an API specification framework. Optic focuses on automated OpenAPI spec updates and change management.
- **Generation**: Swagger tools like Swagger Editor are for manual API definition. Optic auto-captures API traffic to update specs. This is a key difference.
- **Integration**: Swagger often demands manual updates. Optic integrates with developer flows and CI, alerting discrepancies between observed API behavior and the spec.
- **Documentation**: Swagger UI provides interactive docs with request execution. Optic tracks API changes over time, ideal for teams to understand service evolution.
- **Ecosystem**: Swagger boasts tools for editing, visualization, and code generation. Optic specializes in automatic change management.
- **Audience**: Swagger is versatile for broad audiences, while Optic is tailored for environments with frequent API changes, emphasizing automated updates and change tracking.

## How can it all work in a generic serverless application?

Our [sample app](https://github.com/muratkeremozcan/prod-ready-serverless) is from Yan Cui's highly acclaimed [Production-Ready Serverless](https://productionreadyserverless.com/) workshop. Yan guides us through building an event driven serverless app in NodeJs with AWS API Gateway, Lambda, Cognito, EventBridge, DDB, and server-side rendering, which uses AWS Cognito for authentication.

Here is the [link to the repo](https://github.com/muratkeremozcan/prod-ready-serverless) and [Optic PR](https://github.com/muratkeremozcan/prod-ready-serverless/pull/43/files).
Here is a [link to another repo](https://github.com/muratkeremozcan/aws-cdk-in-practice) and its [Optic PR](https://github.com/muratkeremozcan/aws-cdk-in-practice/pull/11/files).

Optic has great documentation, and the below is a cohesive description of the setup with a focus on a serverless app.

### Install Optic to the project, and also globally.

```bash
npm i -D @useoptic/optic
npm i -g @useoptic/optic
```

We need to create an OpenAPI specification. While there are many ways, a quick and dirty way is by using the AWS cli. It needs some tweaks, but we can work with it to start things out.

In the repository we are using [`serverless-export-env`](https://www.serverless.com/plugins/serverless-export-env) which allows us to extract the environment variables of our serverless stack to a `.env` file. The values here mostly change per the deployment.

### Generate your `openapi.yml` file

Create the below script, which will be used to create an OpenApi file using
AWS cli. It uses the [get-export](https://docs.aws.amazon.com/cli/latest/reference/apigateway/get-export.html) command from AWS cli to hit the API gateway and generate the `openapi.yml` file. It assumes you have environment variables `baseUrl` and
`deployment` in your `.env` file, so that we know which api gateway we are concerned with on which
deployment. You can use any var name as long as they match between the .env
file and the script.

```bash
# .env
baseUrl=https://myApiGwId.execute-api.us-east-1.amazonaws.com/dev
deployment=dev # this could be a temp branch, or stage
# create-openapi.sh

# Load .env variables
set -a
source .env
set +a

# Extract the REST API id from the baseUrl
rest_api_id=$(echo $baseUrl | cut -d '/' -f3 | cut -d '.' -f1)

echo "Rest API ID: $rest_api_id"

# Run the aws apigateway get-export command
aws apigateway get-export --rest-api-id $rest_api_id --stage-name $deployment --export-type oas30 --accepts application/yaml ./openapi.yml
```

Give permissions to the file and execute:

```bash
chmod +x create-openapi.sh
./create-openapi.sh
```

> You need the latest AWS CLI version. Here are MAC instructions, here's the [AWS reference](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
>
> ```bash
> curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
> sudo installer -pkg AWSCLIV2.pkg -target /
> 
> ```

Like some other AWS tools, it does not have the best developer experience but we can work around it. The initial `openapi.yml` that gets created with `aws-cli` does not pass checks
at https://apitools.dev/swagger-parser/online/

1. Each path has options > responses, but this needs to be copied also to http-verb > options for each path.
2. Set openapi version to '3.1.0'.
3. Remove the `server` property.

### Create convenience scripts in `package.json`

```json
"optic:lint": "optic lint openapi.yml",
"optic:diff": "optic diff openapi.yml --base main --check'",
"optic:verify": "dotenv -e .env -- bash -c 'echo $baseUrl && optic capture openapi.yml --server-override $baseUrl'",
"optic:update": "dotenv -e .env -- bash -c 'echo $baseUrl && optic capture openapi.yml --server-override $baseUrl --update interactive'"
```

The `optic:update` script will update the OpenAPI specification (the `openapi.yml` file), similar to Jest snapshot update. Optic team recommends to use this locally to "record the API snapshot".

The `optic:verify` script works similar to a Jest snapshot test, check whether the traffic captured in the e2e matches your current openapi specification.

`optic:lint` and `optic:diff` can quickly lint the OpenAPI spec and diff it against main. These scripts are mostly for local use. However, unless the spec has been updated with `optic:update`, the diff will naturally not find any issues.

### One time setup to initialize the Optic (traffic) capture configuration

This creates an`optic.yml` file:

```bash
optic capture init openapi.yml
```

 We need to fine tune it for a serverless stack. In the yml file, enter any placeholder for `server.url`. It has to exist with `https` prefix but does not have to be a valid url.

Remove `server.command` , our server is already deployed and running.

Replace `requests.run.command` wit the e2e test command.

`requests.run.proxy_variable` should be set to your api gateway url, below we are using an environment variable with the name `rest_api_url`.

We are using the e2e test script `cy:run-fast` which turns off video, screenshots and command log. This command could be any kind of e2e/http test for instance in this repository we also have mirroring Jest e2e tests using Axios. So long as they are http tests that hit the API gateway, and kind of test can be used.

```yml
# ./optic.yml

ruleset:
  # Prevent breaking changes
  - breaking-changes
capture:
  openapi.yml:
    server:
      # specified in package.json with --server-override
      url: https://api.example.com # need a placeholder

      ready_endpoint: /
      # The interval to check 'ready_endpoint', in ms.
      # Optional: default: 1000
      ready_interval: 1000
      # The length of time in ms to wait for a successful ready check to occur.
      # Optional: default: 10_000, 10 seconds
      ready_timeout: 10_000
    # At least one of 'requests.run' or 'requests.send' is required below.
    requests:
      # Run a command to generate traffic. Requests should be sent to the Optic proxy, the address of which is injected
      # into 'run.command's env as OPTIC_PROXY or the value of 'run.proxy_variable', if set.
      run:
        # The command that will generate traffic to the Optic proxy. Globbing with '*' is supported.
        # Required if specifying 'requests.run'.
        command: npm run cy:run-fast
        # The name of the environment variable injected into the env of the command that contains the address of the Optic proxy.
        # Optional: default: OPTIC_PROXY
        proxy_variable: rest_api_url
```

### [Setup Optic cloud](https://www.useoptic.com/docs/cloud-get-started)

Optic Cloud has a few key benefits that make it compelling, and a [very generous pricing model](https://www.useoptic.com/pricing):

- PR comments. 
- Centralized api style governance (setup by `optic.yml` file) for all your services, supported with AI.
- [An internal catalogue of all your APIs + changes](https://www.useoptic.com/use-cases/api-change-management) so developers always know how the latest version works. This is much easier to read than `openapi.yml` files in the repos, and the changes between versions are visualized for easier comprehension.
- Dashboards tracking OpenAPI accuracy over time, how well you follow the API standards, and breaking changes + issues over time
- **API Design and Collaboration**: Optic facilitates an interactive feedback mechanism, allowing stakeholders to weigh in on API designs prior to deployment. This proactive approach ensures optimal design choices are made before consumers start relying on them, aligning with the API-first development mantra.
- Support.

Create a token at Optic app. Save this as GitHub secret.
[Enable Optic commenting on pull requests](https://www.useoptic.com/docs/setup-ci#configure-commenting-on-pull-requests) ( `Repo > Settings > Actions > General` and set `Workflow permissions` to `Read and write permissions`)

```bash
optic login
# you will copy over the token from the web prompt

# add the api
optic api add openapi.yml --history-depth=0
```

Once done, you will see your organization at Optic (updated a few seconds ago).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rz59mevh8fgr4rt9evux.png)

### Capture the http traffic using the Optic proxy

Execute the script `optic:update` to capture the traffic and update the `openapi.yml` file. Again, this is similar to recording Jest snapshots.

[From the docs](https://www.useoptic.com/docs/capturing-traffic)

- Your http tests are trafficked through Optic's reverse proxy to your api
  gateway
- Optic observes the traffic, then creates or updates your open api spec.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1ek9yke8nwxdzkr78626.png)

We want to use this script in the CI to verify if the traffic captured in the e2e matches our current OpenAPI specification.

```bash
npm run optic:update
```

The other script `optic:verify` is to test your schema against your openapi spec. This is what we will use in the CI, and it works similar to executing Jest snapshot tests.

```bash
npm run optic:verify
```

We want to add Optic schema verification to our PRs so that the CI executes the schema test when a commit is pushed. There are many varieties of setting up our CI, and we will keep things simple and share the key highlights. For the full yml, take a look at [`PR.yml` file at the repo](https://github.com/muratkeremozcan/prod-ready-serverless/blob/main/.github/workflows/PR.yml).

```yml
# ./.github/workflows/PR.yml

# ...

jobs:
  build-deploy-test:
    runs-on: ubuntu-latest

    # ....

    # Permissions are only needed here if you are using OIDC
    permissions:
      id-token: write
      contents: write
      pull-requests: write # This provides write access to issues and PRs

    steps:
      # installation & deployment...

      # Jest & Cypress tests...

      # Schema verification
      - name: verify the schema with Optic
        run: npm run optic:verify

      # include a preview and changelog in each PR comment
      # sync every OpenAPI spec to Optic Cloud, diff & lint the schema as well
      - uses: opticdev/action@v1
        with:
          # Your Optic Cloud Token
          optic_token: ${{ secrets.OPTIC_TOKEN }}
          # A GitHub token with access to create comments on pull requests
          github_token: ${{ secrets.GITHUB_TOKEN }}
          # If true, standard check failures will cause this action to fail.
          # If false, standard check failures will show in PR comments and
          # in Optic Cloud but will not cause the action to fail
          standards_fail: true
          # If you have more than one spec, separate matches with commas
          # (openapi.yml,other.yml)
          additional_args: --generated --match openapi.yml
          compare_from_pr: cloud:default
          compare_from_push: cloud:default
        env:
          CI: true
```

## Questions, Concerns, other comparisons

> This approach is new and we are trying it out internally at my company [Extend](https://www.extend.com/).
> We will see how serious these concerns actually are and update when we find solutions that work for us.

### Why both e2e and schema test?

Q: If we have e2e tests already, why are we also running schema tests? We need the serverless stack to be deployed for either to be running, because in schema testing the serverless stack we are recording or verifying the traffic with utilizing e2e tests too. What value add do we get from the schema test?

A: In the case of serverless applications, while both e2e and schema testing necessitate the deployment of the serverless stack, and schema tests often leverage e2e for traffic recording and verification, they offer unique perspectives and benefits.

**1. Precision and Specificity**: E2E tests provide a holistic system assessment, a comprehensive overview. In contrast, schema tests zero in on the API's structural integrity, ensuring its responses adhere strictly to the predefined structure. This granular approach might capture subtle inconsistencies broader tests could miss.

**2. Documentation, Drift Detection, and Change Management**: schema tests offer more than mere validation. They play a pivotal role in:

- **Dynamic Documentation Syncing**: Immediate reflection of API changes in its documentation ensures continuous alignment, fostering improved developer communication and minimizing documentation drift risks.
- **API Lifecycle Oversight**: They aid in detecting breaking changes, streamlining versioning, and managing the evolution of the API. This proactive, automated approach uniquely distinguishes schema testing from its e2e counterpart.

---

### How about increased CI response time?

Q: How do we ensure schema testing has less impact on CI response time?

A: In a CI pipeline our setup can look like so:

```yml
install----deploy---------unit & integration
--------------------------e2e
--------------------------schema:verify
```

With that setup we can have a minimal impact on the duration of the pipeline.

Additionally, if we have local testing capabilities - we are working on this at my company [Extend](https://www.extend.com/) - schema testing can happen prior to deploy, or concurrently.

```yml
install----deploy-----------unit & integration
-----------schema:verify----e2e                      
```

```yml
install----schema:verify----deploy----unit & integration
--------------------------------------e2e
```

---

### Do we keep having to record the schema with `schema:verify` ?

Q: In the CI examples, we are executing the `schema:verify` , but when do we execute `schema:update` ? Because, if we do not update the schema, then the tests may give false positives.

A: Usually schema update is a chore; a manual process where we update the `openapi.yml` file. If we have other methods of updating the schema -perhaps we update it via our code, and or types- we do not need to `schema:update` and in turn run the e2e suite locally. 

If we do not have any practical way to update the OpenAPI specification, being able to do so with Optic is a boon. The engineers are responsible to own the schema update when they are making potential changes to the schema, so they would take advantage of automatically updating it by executing `schema:update` and not having to think about it.

Additionally, we can have a cron job in CI that runs nightly and updates the schema. The only gap in this approach would be is if we neglect the local `schema:update` execution, and we are changing the schema *and* releasing that day something. The cron job can reduce the human negligence error, but not entirely eliminate it.

---

### Optic approach vs consumer driven contract testing (Pact)

Q: How does this approach relate to consumer driven contract testing (Pact)? How do they compare to just e2e?

A: First, a comparison of both with e2e:

- With e2e, you deploy and release your service, and you can break consumer expectations, letting them know too late (usually never letting them know and they are surprised)
- With Pact io, you would run a test and realize it is breaking the consumer’s expectation, communicate with the consumer you are going to break their expectations. They are informed prior to the deployment. Alternatively, esspecially if it is an external consumer, you do not make the breaking change.
- With Optic, you realize you are changing your schema, because the schema test fails, so you realize you are probably breaking consumer expectations. A similar communication or decision making process takes place; either you let them know about the breaking change or do not make it.

Now, let's compare API governance with consumer-driven contract testing and the Optic approach:

**Consumer-Driven Contract Testing with Pact**:

- **Prevention Over Correction**: Instead of finding out post-deployment that an API change has disrupted a consumer, you find out during the development phase.
- **Consumer-Centric**: Contracts are based on the consumer's perspective. Providers ensure they meet these contracts.
- **Continuous Feedback**: As API providers and consumers evolve, the contracts can be continually validated and updated.

**Optic**:

- **Observation Over Definition**: Instead of pre-defined contracts, Optic observes actual API behavior and alerts developers to unintended changes.
- **Documentation & Drift Detection**: As API evolves, Optic helps ensure documentation is automatically and accurately updated, reducing the chances of drift.
- **Automated Schema Testing**: You understand changes to your schema, allowing for proactive communication with consumers about potential disruptions.

**Key differences:**

- You have to write the consumer driven contract tests with Pact which is a lot of work compared to just maintaining automated schema test with Optic.
- Pact is not going to tell you anything about API best practices, design standards, etc which is also another really important part of doing governance.

## Conclusion

**Schema testing, Optic, and e2e tests** together create a formidable toolkit for enhancing API development in serverless architectures:

1. **Enhanced Testing Workflow:** Pairing Optic with e2e tests, especially with tools like Cypress, offers a dynamic way to test APIs in event-driven systems. Schema testing is adding another cost-effective & responsive layer to our testing toolbox.
2. **Automated Schema Generation:** Using AWS CLI and OpenAPI specs, API schemas can be initially generated.
3. **Real-time Traffic Capturing with Optic:** The OpenAPI specifications are kept accurate through real-time traffic observation, ensuring a true representation of system interactions.
4. **Effortless API Documentation Updates:** With Optic's traffic capture capabilities, not only is API behavior tracked, but documentation is also automatically updated. This means every time you run your e2e tests, you're also refreshing your API documentation, ensuring it remains relevant and up-to-date.
5. **Simplified Integration & Collaboration:** Embedding Optic commands in `package.json` and using Optic Cloud for team-based API review streamlines collaboration. With CI integration, it enforces an API-first development approach that remains consistent with the defined schema.
6. **Visual Oversight:** Optic Cloud's visual tools provide a clear, intuitive view of API changes, negating the need to delve into raw specification files.

In essence, the synergistic effect of schema testing, Optic, and end-to-end testing tools like Cypress offers a comprehensive solution to API development challenges in serverless environments.
