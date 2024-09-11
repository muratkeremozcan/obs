In the previous blog post [Documenting and Testing Schemas of Serverless Stacks with Optic & Cypress](https://dev.to/muratkeremozcan/schema-testing-serverless-stacks-with-optic-cypress-26f5), we focused on the benefits of schema testing and governance. Briefly, some of the core problems addressed were:

1. **Effortless API Documentation**: Automating the API documentation creation and update, ensuring that our API documentation evolves in tandem with our schema, courtesy of Optic's forward-governing capabilities.
2. **Detecting Cross-Service Integration Issues Up Front**: Having the ability to test and detect cross-service integration issues locally; long before we have to deploy services to a common environment and test them with e2e.
3. **Making the Process Painless**: Re-using existing HTTP/e2e suites (or their subsets) to update and/or verify our OpenAPI schema; no additional work for team-level service owners.

In the previous approach, we relied on Optic's key feature to [capture the HTTP traffic using the Optic proxy](https://dev.to/muratkeremozcan/schema-testing-serverless-stacks-with-optic-cypress-26f5#capture-the-http-traffic-using-the-optic-proxy). Henceforth in the text, we will refer to that as **HTTP-capturing Approach**:

1. Capture HTTP traffic during e2e tests.
2. Generate or update OpenAPI spec based on captured traffic.
3. Perform schema governance with [Optic](https://www.useoptic.com/).

In this post, we want to evaluate an alternative approach of generating our OpenAPI documentation from our TypeScript types. Henceforth in the text, we will refer to that as **TypeScript-based Approach**:

1. Organize our TypeScript request and response types into a certain folder.
2. Use these TypeScript request & response types in our lambda code.
3. Generate JSON schemas with the [`ts-json-schema-generator`](https://github.com/vega/ts-json-schema-generator) package.
4. Generate the OpenAPI spec with the [`openapi-types`](https://github.com/octokit/openapi-types.ts) package.
5. Perform schema governance with [Optic](https://www.useoptic.com/).

For our case study, we utilize the same comprehensive repository that includes a TypeScript-based backend and frontend, AWS Lambdas, and temporary stacks managed with AWS CDK. These components are tested in PRs through backend and frontend e2e tests targeting temporary deployments, as well as consistent deployments in development, staging, and production environments.

Here is the [link to the repo](https://github.com/muratkeremozcan/aws-cdk-in-practice) and initial [Optic PR](https://github.com/muratkeremozcan/aws-cdk-in-practice/pull/11/files).
Here is the [new PR](https://github.com/muratkeremozcan/aws-cdk-in-practice/pull/18) with OpenAPI generation from types.

> Note: This case study results in two different OpenAPI spec files, each representing a distinct approach. The traditional method involving HTTP tests and Optic proxy generates an `openapi.yml` file, whereas the new type-based method produces an `openapi.json` file. Keep in mind that in practical applications, the choice of file format (YAML or JSON) depends on your specific needs. This case study also includes two sets of Optic diff and lint scripts, which is unusual for most real-world applications but was necessary here to clearly differentiate between the two approaches within a single repository."

- [1. Organize our TypeScript request and response types into a certain folder](#1-organize-our-typescript-request-and-response-types-into-a-certain-folder)
  - [Create a TS file per handler to house the request and response types](#create-a-ts-file-per-handler-to-house-the-request-and-response-types)
- [2. Use these TypeScript request \& response types in our lambda code](#2-use-these-typescript-request--response-types-in-our-lambda-code)
- [3. Generate JSON schemas with the `ts-json-schema-generator` package](#3-generate-json-schemas-with-the-ts-json-schema-generator-package)
- [4. Generate the OpenAPI spec with the `openapi-types` package.](#4-generate-the-openapi-spec-with-the-openapi-types-package)
  - [Create the `openapi.ts` file](#create-the-openapits-file)
  - [Create a script to execute the `openapi.ts` file and generate the OpenAPI spec.](#create-a-script-to-execute-the-openapits-file-and-generate-the-openapi-spec)
- [5. Perform schema governance with Optic](#5-perform-schema-governance-with-optic)
- [Comparison of the TS approach vs http-capture approach](#comparison-of-the-ts-approach-vs-http-capture-approach)
  - [TypeScript-based Approach with Optic: Use Case Scenario:](#typescript-based-approach-with-optic-use-case-scenario)
  - [HTTP-based Approach with Optic: Use Case Scenario](#http-based-approach-with-optic-use-case-scenario)
- [Conclusion](#conclusion)
- [Addendum: Using the Types at the Lambdas as the Source of Truth (Recommended)](#addendum-using-the-types-at-the-lambdas-as-the-source-of-truth-recommended)

### 1. Organize our TypeScript request and response types into a certain folder

> ```bash
> #(We are here)
> Using types -> JSON schemas -> OpenAPI spec -> schema diffing with Optic
> ```

In the source code, let's examine our [lambda handlers](https://github.com/muratkeremozcan/aws-cdk-in-practice/tree/main/infrastructure/Lambda). Currently, they return HTTP responses but do not utilize specific request or response types.

```ts
// ./infrastructure/Lambda/delete/lambda/index.ts

return httpResponse(
  200,
  JSON.stringify({ message: "Todo deleted successfully." })
);
```

```ts
// ./infrastructure/Lambda/get/lambda/index.ts

const { Items }: DynamoDB.ScanOutput = await dynamoDB
  .scan({ TableName: tableName })
  .promise();

return httpResponse(200, JSON.stringify({ todos: Items }));
```

```ts
// ./infrastructure/Lambda/post/lambda/index.ts

const todo: Todo = {
  id: uuidv4(),
  todo_completed,
  todo_description,
  todo_name,
};

await dynamoDB.put({ TableName: tableName, Item: todo }).promise();

return httpResponse(200, JSON.stringify({ todo }));
```

```ts
// ./infrastructure/Lambda/put/lambda/index.ts

const updatedTodo: Todo = {
  id,
  todo_name,
  todo_description,
  todo_completed,
};

return httpResponse(200, JSON.stringify({ todo: updatedTodo }));
```

If we define request and response types centrally and use them across our lambda functions, we can leverage them to automatically generate an OpenAPI specification. This method ensures that our OpenAPI documentation remains synchronized with our codebase, reflecting any changes in our types seamlessly. The cost is the requirement of a one-time setup investment. This approach not only streamlines our documentation workflow but also enhances the accuracy and reliability of our API specifications.

The distinction from our previous approach, which involved capturing HTTP tests via the Optic proxy to verify or update our OpenAPI schema, lies in our newfound reliance on TypeScript types. In the earlier method, we depended on capturing real HTTP traffic to reflect our API's behavior. Now, we pivot to a more proactive approach, using our TypeScript request and response types as the primary source of truth.

#### Create a TS file per handler to house the request and response types

This folder can be anywhere in our repository. For the example's sake, let's assume it is under `./infrastructure/api-specs`. We will also make a possible real world assumption that there may be multiple versions of the API, and we are working with v1.

```bash
├── api-specs
  └── v1
      ├── deleteTodo.ts
      ├── getTodos.ts
      ├── postTodo.ts
      ├── putTodo.ts
```

Define the response body and if needed the request body per handler. Here we get a clue from our already existing lambda code.

```ts
// ./infrastructure/api-specs/v1/deleteTodo.ts

export type ResponseBody = {
  message: string;
};
```

Import any types used in these responses from anywhere. Example: type Todo.

```ts
// ./infrastructure/api-specs/v1/getTodos.ts

// import any types used in these responses from anywhere
import type { Todo } from "customTypes/index";

export type ResponseBody = {
  todos: Todo[];
};
```

If we have a possible request body, define it here. Although it may or may not be used in the lambda function, we want to define these here so that our OpenAPI doc is complete for the benefit of our API's consumers.

```ts
// ./infrastructure/api-specs/v1/postTodo.ts

import type { Todo } from "customTypes/index";

export type ResponseBody = {
  todo: Todo;
};

export type RequestBody = {
  todo: Todo;
};
```

```ts
// ./infrastructure/api-specs/v1/putTodo.ts

import type { Todo } from "customTypes/index";

export type ResponseBody = {
  todo: Todo;
};

export type RequestBody = {
  todo: Todo;
};
```

### 2. Use these TypeScript request & response types in our lambda code

> ```bash
> #(We are here still)
> Using types -> JSON schemas -> OpenAPI spec -> schema diffing with Optic
> ```

Now that we have the request and response types in a central location, use them in our lambda code. Here are the key changes in the repo example. You can find the full code in the repo as well as the PR.

Import the type from the central location, and use it in the handler. All we are doing differently here is making an assignment to a const with a type and using it in the http response.

```ts
// ./infrastructure/Lambda/delete/lambda/index.ts

import { ResponseBody } from "api-specs/v1/deleteTodo";

// the assignment
const response: ResponseBody = {
  message: "Todo deleted successfully.",
};

// before:
// JSON.stringify({message: 'Todo deleted successfully.'}),
// after:
return httpResponse(200, JSON.stringify(response));
```

Disclaimer; we may need to do some additional work to make TypeScript happy, but this will also be a one-time effort. We foresee that most the time the code change will be minimal, and they will be future proof because of the type protection.

> Check out the Addendum section for an alternative approach where the types in the lambdas are the source of the truth, which can mediate such issues.

```ts
// ./infrastructure/Lambda/get/lambda/index.ts

const { Items }: DynamoDB.ScanOutput = await dynamoDB
  .scan({ TableName: tableName })
  .promise();

const response: ResponseBody = { todos: Items };

// before:
// return httpResponse(200, JSON.stringify({todos: Items}))
// after:
return httpResponse(200, JSON.stringify(response));
```

```ts
// ./infrastructure/Lambda/post/lambda/index.ts

import type { ResponseBody } from "api-specs/v1/postTodo";

// this part is the same
const todo: Todo = {
  id: uuidv4(),
  todo_completed,
  todo_description,
  todo_name,
};
await dynamoDB.put({ TableName: tableName, Item: todo }).promise();

// Use the Response Type in the Lambda Handler
const response: ResponseBody = { todo };

// before:
// return httpResponse(200, JSON.stringify({todo}))
// after:
return httpResponse(200, JSON.stringify(response));
```

```ts
// ./infrastructure/Lambda/put/lambda/index.ts

const updatedTodo: Todo = {
  id,
  todo_name,
  todo_description,
  todo_completed,
};

// Use the Response Type in the Lambda Handler
const response: ResponseBody = { todo: updatedTodo };

// before:
// return httpResponse(200, JSON.stringify({todo: updatedTodo}))
// after:
return httpResponse(200, JSON.stringify(response));
```

> Note: In the following sections, the files can be named and placed anywhere of your preference. The scripts may need adjusting. It is only important that if you plan to use them ubiquitously, every service repo follows the set pattern.

### 3. Generate JSON schemas with the [`ts-json-schema-generator`](https://github.com/vega/ts-json-schema-generator) package

> ```bash
> ############# (We are here)
> Using types -> JSON schemas -> OpenAPI spec -> schema diffing with Optic
> ```

The library is used to generate json schemas, which in turn will get used in the final open api spec. We have to use an elaborate script here, but it should not require modification and should be easy to reuse. Consider creating a package for these scripts for repeated usage.

> You can find the final code in the repo and the PR.

```ts
// ./infrastructure/api-specs/generate-json-schemas.ts

import * as tsj from "ts-json-schema-generator";
import * as fs from "fs";
import * as path from "path";

// Function to recursively find all .ts files in subdirectories and exclude 'openapi.ts'
function findTsSchemaFiles(
  dir: string,
  fileList: string[] = [],
  isRoot = true
): string[] {
  fs.readdirSync(dir, { withFileTypes: true }).forEach((dirent) => {
    const fullPath = path.join(dir, dirent.name);
    if (dirent.isDirectory()) {
      // Process subdirectories; skip processing the api-specs folder root
      if (!isRoot) {
        fileList = findTsSchemaFiles(fullPath, fileList, false);
      }
    } else if (
      dirent.isFile() &&
      dirent.name.endsWith(".ts") &&
      dirent.name !== "openapi.ts"
    ) {
      // Add only .ts files that are not named 'openapi.ts', and only if it's not in the root directory
      if (!isRoot) {
        fileList.push(fullPath);
      }
    }
  });

  // If it's the root directory, proceed to its subdirectories
  if (isRoot) {
    fs.readdirSync(dir, { withFileTypes: true }).forEach((dirent) => {
      if (dirent.isDirectory()) {
        fileList = findTsSchemaFiles(
          path.join(dir, dirent.name),
          fileList,
          false
        );
      }
    });
  }

  return fileList;
}

// Function to generate JSON schema from a TypeScript file
function generateSchema(tsFilePath: string): void {
  const schemaFilePath = tsFilePath.replace(".ts", ".schema.json");
  const config = {
    path: tsFilePath,
    tsconfig: path.join(__dirname, "../tsconfig.json"),
    noTypeCheck: true,
    // generate schema for all types;
    // RequestBody, ResponseBody and all the imported types they need
    type: "*",
    // avoid creating shared $ref definitions (which is not valid in OpenAPI)
    // this e results in JSON schema files that directly embed the type definitions,
    // instead of referring to them via $ref
    expose: "none" as const,
  };

  try {
    const schema = tsj.createGenerator(config).createSchema(config.type);
    fs.writeFileSync(schemaFilePath, JSON.stringify(schema, null, 2));
    console.log(`Generated JSON schema for ${tsFilePath}`);
  } catch (error) {
    console.error(`Error generating JSON schema for ${tsFilePath}:`, error);
  }
}

// Main execution
const openApiFiles = findTsSchemaFiles(__dirname);
openApiFiles.forEach(generateSchema);
```

```bash
├── api-specs
  ├── generate-json-schemas.ts
  └── v1
      ├── deleteTodo.ts
      ├── getTodos.ts
      ├── postTodo.ts
      ├── putTodo.ts
```

### 4. Generate the OpenAPI spec with the [`openapi-types`](https://github.com/octokit/openapi-types.ts) package

> ```bash
> ############################# (We are here)
> Using types -> JSON schemas -> OpenAPI spec -> schema diffing with Optic
> ```

We need to utilize this library and make a one time investment to create an `openapi.ts` file. This is the file that will generate the OpenAPI spec. Note that if we add new endpoints to our api, we will need to add it to this file as well. If we have different versions of the API, we might need multiple files, but since these are all in code, using helper modules is a possibility albeit at the cost of abstraction.

#### Create the `openapi.ts` file

The general pattern in the file:

- Import the JSON schemas, created in the previous step: `import getTodosV1 from './getTodos.schema.json'`

- In the components section, identify these schemas:

  ```ts
  components: {
    schemas: {
      getTodosV1: getTodosV1.definitions as OpenAPIV3_1.SchemaObject,
    },
  ```

- Reference the components

  ```ts
  content: {
    'application/json': {
      schema: {
        $ref: '#/components/schemas/getTodosV1',
      },
    },
  },
  ```

Aside from having to add new endpoints to our API, this file does not need any maintenance. If we have new endpoints though, we might need some copy pasting for them.

```ts
// ./infrastructure/api-specs/v1/openapi.ts

import type { OpenAPIV3_1 } from "openapi-types";
import fs from "fs";
import path from "path";
import getTodosV1 from "./getTodos.schema.json";
import deleteTodoV1 from "./deleteTodo.schema.json";
import postTodoV1 from "./postTodo.schema.json";
import putTodoV1 from "./putTodo.schema.json";

export const openapi: OpenAPIV3_1.Document = {
  openapi: "3.0.1",
  info: {
    title: "aws cdk in practice specification",
    version: "1.0.0",
  },
  paths: {
    "/": {
      get: {
        responses: {
          200: {
            description: "Success",
            content: {
              "application/json": {
                schema: {
                  $ref: "#/components/schemas/getTodosV1",
                },
              },
            },
          },
        },
      },
      post: {
        requestBody: {
          required: true,
          content: {
            "application/json": {
              schema: {
                $ref: "#/components/schemas/postTodoV1",
              },
            },
          },
        },
        responses: {
          200: {
            description: "Success",
            content: {
              "application/json": {
                schema: {
                  $ref: "#/components/schemas/postTodoV1",
                },
              },
            },
          },
        },
      },
      put: {
        requestBody: {
          required: true,
          content: {
            "application/json": {
              schema: {
                $ref: "#/components/schemas/putTodoV1",
              },
            },
          },
        },
        responses: {
          200: {
            description: "Success",
            content: {
              "application/json": {
                schema: {
                  $ref: "#/components/schemas/putTodoV1",
                },
              },
            },
          },
        },
      },
    },
    "/{id}": {
      delete: {
        parameters: [
          {
            name: "id",
            in: "path",
            required: true,
            schema: {
              type: "string",
            },
          },
        ],
        responses: {
          200: {
            description: "Success",
            content: {
              "application/json": {
                schema: {
                  $ref: "#/components/schemas/deleteTodoV1",
                },
              },
            },
          },
        },
      },
    },
  },
  components: {
    schemas: {
      getTodosV1: getTodosV1.definitions as OpenAPIV3_1.SchemaObject,

      deleteTodoV1: deleteTodoV1.definitions as OpenAPIV3_1.SchemaObject,

      postTodoV1: postTodoV1.definitions as OpenAPIV3_1.SchemaObject,

      putTodoV1: putTodoV1.definitions as OpenAPIV3_1.SchemaObject,
    },
  },
};

const filePath = path.join(__dirname, "openapi.json");
fs.writeFileSync(filePath, JSON.stringify(openapi, null, 2));
```

```
├── api-specs
  ├── generate-json-schemas.ts
  └── v1
      ├── openapi.ts
      ├── deleteTodo.ts
      ├── getTodos.ts
      ├── postTodo.ts
      ├── putTodo.ts
```

#### Create a script to execute the `openapi.ts` file and generate the OpenAPI spec.

Now that we have the `openapi.ts` file, We need a script to find the `openapi.ts` file(s) and execute them, in order to generate the OpenAPI spec. There may be multiple version folders, and the script will accommodate that.

```ts
// ./infrastructure/api-specs/generate-openapi-docs.ts

import fs from "fs";
import path from "path";

// Function to recursively find all openapi.ts files
function findOpenApiFiles(dir: string, fileList: string[] = []): string[] {
  fs.readdirSync(dir, { withFileTypes: true }).forEach((dirent) => {
    const filePath = path.join(dir, dirent.name);
    if (dirent.isDirectory()) {
      fileList = findOpenApiFiles(filePath, fileList);
    } else if (dirent.isFile() && dirent.name === "openapi.ts") {
      fileList.push(filePath);
    }
  });
  return fileList;
}

// Find all openapi.ts files in src/api-specs
const openApiFiles = findOpenApiFiles(__dirname);
console.log(openApiFiles);

// Import and execute each openapi.ts file to generate openapi.json
openApiFiles.forEach((file) => {
  import(path.resolve(file))
    .then(() => console.log(`Generated OpenAPI document for ${file}`))
    .catch((err) =>
      console.error(`Error generating OpenAPI document for ${file}:`, err)
    );
});
```

We are including a bonus script here to reset/delete all the generated json, for demo usage.

```ts
// ./infrastructure/api-specs/delete-json-files.ts

import fs from "fs";
import path from "path";

// Function to recursively delete all .json files
function deleteJsonFiles(dir: string): void {
  fs.readdirSync(dir, { withFileTypes: true }).forEach((dirent) => {
    const fullPath = path.join(dir, dirent.name);
    if (dirent.isDirectory()) {
      // Recursively delete .json files in subdirectories
      deleteJsonFiles(fullPath);
    } else if (dirent.isFile() && dirent.name.endsWith(".json")) {
      // Delete the file if it's a .json file
      fs.unlinkSync(fullPath);
      console.log(`Deleted file: ${fullPath}`);
    }
  });
}

const apiSpecsDir = path.join(__dirname);
deleteJsonFiles(apiSpecsDir);
```

```bash
├── api-specs
  ### these script files can be a part of a package
  ├── generate-json-schemas.ts
  ├── generate-openapi-docs.ts
  ├── delete-json-files.ts
  └── v1
      ├── openapi.ts
      ├── deleteTodo.ts
      ├── getTodos.ts
      ├── postTodo.ts
      ├── putTodo.ts
```

> The OpenAPI file generated is of json type, for the example purposes and not to intermix it with the http-capture approach we have referenced before which uses the yml file type.

### 5. Perform schema governance with [Optic](https://www.useoptic.com/)

Optic helps in detecting schema changes in our OpenAPI specification (`openapi.json or yml`). It categorizes these changes as either breaking or non-breaking; which is key for us to identify them. This is something we do not get with our own testing; we would just update the types and/or the test and would not really know if they would break future service integrations unless we are consistently very careful and knowledgeable.

We can use Optic locally, but CI is where it shines, making it obvious for us to detect such changes, or breakages in a schema, not only have analytics and history (Optic Cloud) but also have a neat representation of our OpenAPI spec in the form of online documentation.

Now that we have the OpenAPI spec, we can create a few utility scripts. You can reference the repo or the PR for the script detail, and modify them to your needs in the real world. We have differentiated the -json vs -yml scripts for reasons previously mentioned with the two approaches.

```bash
yarn update:api-docs # generates JSON schemas and OpenAPI docs
yarn optic:lint-json # lints our OpenAPI spec for validity
yarn optic:diff-json # detects breaking schema changes vs main with Optic
```

```json
"reset:schemas": "npx ts-node ./api-specs/delete-json-files.ts",
"build:schemas": "npx ts-node ./api-specs/generate-json-schemas.ts",
"build:open-api": "npx ts-node ./api-specs/generate-openapi-docs.ts",
"update:api-docs": "yarn reset:schemas && yarn build:schemas && yarn build:open-api"
"optic:diff-json": "optic diff ./api-specs/v1/openapi.json --base main --check'",
"optic:lint-json": "optic lint ./api-specs/v1/openapi.json",
```

Note that in this repository we are already running Optic Cloud with the http-capture approach, in CI. We cannot have 2 OpenAPI specifications; one for TS approach and one for http-capture approach, and perform schema governance. However, we will propose a simple CI config, for the TS approach. Mind that in the repo you will find the http-capture approach working in the CI.

```yml
name: Optic-cloud-features
on:
  pull_request:
    types: [opened, reopened, edited, synchronize]

concurrency:
  group: ${{ github.ref }} && ${{ github.workflow }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  optic-cloud:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: write
      pull-requests: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4.1.1

      - name: Use node
        uses: actions/setup-node@v3.8.2
        with:
          node-version-file: .nvmrc
          cache: yarn

      - name: Install dependencies
        run: yarn install --frozen-lockfile

      - name: Update api docs
        run: yarn update:api-docs

      # include a preview and changelog in each PR comment
      # sync every OpenAPI spec to Optic Cloud, diff & lint the schema as well
      - name: Run Optic
        uses: opticdev/action@v1
        with:
          # Your Optic Cloud Token
          optic_token: ${{ secrets.OPTIC_TOKEN }}
          # A GitHub token with access to create comments on pull requests
          github_token: ${{ secrets.GITHUB_TOKEN }}
          # If true, standard check failures will cause this action to fail.
          # If false, standard check failures will show in PR comments and
          # in Optic Cloud but will not cause the action to fail
          standards_fail: true
          additional_args: --match **/**/openapi.json
          compare_from_pr: main
          compare_from_push: main
        env:
          CI: true
```

### Comparison of the TS approach vs http-capture approach

#### TypeScript-based Approach with Optic: Use Case Scenario:

1. **Identifying Type Changes**: A breaking change is introduced to our TypeScript types, which could potentially alter the structure or behavior of our API.
2. **Auto-Generating OpenAPI Documentation**: Following these type changes, our OpenAPI documentation is automatically regenerated using the `update:api-docs` command. This process ensures that the OpenAPI specification reflects the latest state of our TypeScript types.
3. **Detecting Breaking Changes with Optic**: The command `optic:diff-json` is used to detect any breaking changes in the updated OpenAPI specification. This step is crucial for identifying discrepancies that could affect API consumers.
4. **Decision Making and Communication**:
   - **Rollback or Update**: Based on the nature of the detected changes, we decide whether to roll back the changes to our types or proceed with updating the API documentation to reflect these changes.
   - **Consumer Notification and Version Update**: For significant changes, especially those that are breaking, we notify service consumers of the potential impact. Additionally, we update the version of our OpenAPI specification. This step is essential to communicate the change effectively and ensure that Optic's checks recognize the update as a deliberate and managed change.

This approach emphasizes the role of TypeScript types in driving the API documentation process. By automatically generating the OpenAPI specification from TypeScript types, we ensure that our documentation is always in sync with our codebase. Moreover, it highlights the use of Optic as a tool for governing API schema changes, ensuring that updates are tracked, managed, and communicated effectively.

#### HTTP-based Approach with Optic: Use Case Scenario

Optic can also validate the accuracy of the OpenAPI spec by capturing traffic from E2E tests and comparing it against the OpenAPI spec (`openapi.json` or yml file).

```bash
# in ./infrastructure folder
yarn optic:verify # Captures E2E test traffic, detects breaking schema changes
```

Similar to Optic validate, Optic update allows for interactive updates to the OpenAPI spec. It's similar to `optic:verify` but includes prompts for additional observed changes during E2E test capture. 

```bash
# in ./infrastructure folder
yarn optic:update
```

We generally would run `optic:verify` in CI to vet our http tests against our OpenAPI specification. We would run `optic:update` when we know there are changes in our code, not necessarily types, but could be anything. We would theoretically update the code, perhaps update the http tests, run `optic:verify` and record a new OpenAPI spec. This usage is very similar to Jest snapshot testing, where the snapshot is the OpenAPI spec.

![diagram](https://www.useoptic.com/img/proxy-diagram.png)

Use case scenario:

1. **Identifying Potential Breaking Changes**: Any black-box breaking change is made in our service code. This change isn't necessarily related to type definitions but could impact the behavior of the API.

2. **Verification with Optic**: We use `optic:verify` to identify these changes.
   - **Real E2E Tests**: This involves executing real end-to-end tests against a local server or a deployment with our code changes.
   - **Comparison Against OpenAPI Spec**: During this process, Optic verifies the traffic captured during these tests against our existing OpenAPI specification.

3. **Updating OpenAPI Documentation**: If the actual behavior (captured traffic) does not match the current OpenAPI documentation, we are suggested by Optic to update the OpenAPI docs, so we utilize `optic:update` to bring the documentation in line with reality. This ensures that `optic:verify` can now successfully pass, providing us with an accurate coverage report of our OpenAPI documentation.

4. **Detecting Breaking Changes**: The `optic:diff` command is used to identify any breaking changes compared to the main/master branch. This step is crucial for understanding the impact of recent changes on the overall API.

5. **Making Informed Decisions**: Depending on the nature of the detected changes, we make a decision:
   - **Discarding or Updating**: We either discard the recent changes or update our API documentation using `optic:update`.
   - **Communication and Versioning**: For significant or breaking changes, we communicate these changes to our service consumers. Additionally, we update the version of our OpenAPI specification to reflect these changes, ensuring that Optic's checks pass.

This approach effectively leverages Optic's capabilities to manage and maintain accurate and up-to-date API documentation, especially in the context of continuous integration and delivery. It highlights the importance of aligning actual service behavior with documented API contracts, ensuring consistency and reliability for API consumers.

### Conclusion

In conclusion, this exploration into generating OpenAPI documentation from TypeScript types presents a viable alternative to the HTTP-capturing method. While both approaches have their merits, the TypeScript-based method offers a cost effective and type-safe strategy, ensuring that API documentation remains closely aligned with our codebase. It is particularly beneficial for teams seeking to automate their documentation process but keep things simple and fast, albeit it does require the one time investment, and updating of the `opeanapi.ts` file when there are new api endpoints.

On the other hand, using real http tests to qualify our OpenAPI documentation gives us the ability to not only verify but also modify our spec. Reality is often different to the wishful perception of it, and the HTTP-capturing method is immune to that. The coverage report of our http tests versus our OpenAPI docs is a killer feature as well, giving us the proof of coverage that matters; "_Are our tests covering what we publish that we feature?_". 

> This is a much better alternative to source code coverage, akin to ui-interaction-coverage in Cypress Cloud (for UI apps).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o5ftgzb41ccdqqknvivw.png)

In either approach we get the schema governing features of Optic. We get optic-diff locally or in CI, terminal output for the free version, and human readable web version with PR comments using Optic Cloud.

As a reminder, here are the Optic Cloud benefits in brief

- optic diff-all (runs multiple specs at once instead of having to pick each OpenAPI spec)
- PR comments
- [Catalogue of your API changes over time, analytics ](https://app.useoptic.com/organizations/61d22cd6-d47c-478f-885d-677f8a89449f/apis)
- centralized styles guides, with AI
- support

Whether we use HTTP-capturing or TypeScript-based approach, we still get the same schema governance with Optic.

![alt](https://www.useoptic.com/changelog2.jpg)

![alt](https://www.useoptic.com/_next/image?url=%2F_next%2Fstatic%2Fmedia%2Fcompared-to-github.77af37a6.png&w=3840&q=75)

Looking ahead, further enhancements could include exploring ways to integrate these methods into more complex, multi-service architectures, starting with creating an internal package for scripts that will be repeated in each service repo.

Your feedback and experiences with these approaches are invaluable—feel free to share your thoughts and insights.

### Addendum: Using the Types at the Lambdas as the Source of Truth (Recommended)

Previously, we created type definitions in the openapi-spec folder and then used them in the lambda. This approach ensured that the same type was being used in both the lambda and the OpenAPI spec, thereby creating an alignment.

```
types in getTodos -> lambda && OpenAPI spec
```

However, a better approach might be to define a type in the lambda and then use it directly in the openAPI spec, with the getTodos type file serving as an intermediary.

```
lambda -> types in getTodos -> OpenAPI spec
```

In this manner, the lambda becomes the source of truth.

```ts
// ./infrastructure/Lambda/delete/index.ts

// (1) define & export a type for the response body,
export type DeleteResponseBody = {
  message: string;
};

export const handler = async (event: DeleteEvent) => {
  // ...

  // (2) and use it in the Lambda Handler
  const response: DeleteResponseBody = {
    message: 'Todo deleted successfully.',
  };

  return httpResponse(200, JSON.stringify(response));
};
```

Reuse the exported `DeleteResponseBody` in API docs.

```ts
// ./infrastructure/api-specs/v1/getTodos.ts

// (3) re-use the exported type in api docs
import type {DeleteResponseBody} from '../../Lambda/delete/lambda/index';

export type ResponseBody = DeleteResponseBody;
```

For the POST endpoint:

```ts
// ./infrastructure/Lambda/post/index.ts

export type PostBody = {
  todo: Todo;
};

export const handler = async (event: PostEvent) => {
  // ...

  const response: PostBody = { todo };

  return httpResponse(200, JSON.stringify(response));
};
```

```ts
// ./infrastructure/api-specs/v1/postTodo.ts

import type {PostBody} from '../../Lambda/post/lambda/index';

export type ResponseBody = PostBody;
export type RequestBody = PostBody;
```

For the PUT endpoint:

```ts
// ./infrastructure/Lambda/put/index.ts

export type PutBody = {
  todo: Todo;
};

export const handler = async (event: PutEvent) => {
  // ...

  const response: PutBody = { todo: updatedTodo };

  return httpResponse(200, JSON.stringify(response));
};
```

```ts
// ./infrastructure/api-specs/v1/putTodo.ts

import type {PutBody} from '../../Lambda/put/lambda/index';

export type ResponseBody = PutBody;
export type RequestBody = PutBody;
```

The GET endpoint is particularly noteworthy as it differentiates this approach from the previous one. In the past, we struggled to make `Todo[]` compliant with the more generic DynamoDB expectation. Now, we can create a type that works for both our lambda and generates more precise API documentation.

```ts
// ./infrastructure/Lambda/get/index.ts

export type GetResponseBody = {
  todos: Partial<Todo>[] | undefined;
};

export const handler = async () => {
  const {Items} = await dynamoDB.scan({TableName: tableName}).promise();

  const response: GetResponseBody = {todos: Items};

  return httpResponse(200, JSON.stringify(response));
};
```

```ts
// ./infrastructure/api-specs/v1/getTodos.ts

import type {GetResponseBody} from '../../Lambda/get/lambda/index';

export type ResponseBody = GetResponseBody;
```

Here is the [PR](https://github.com/muratkeremozcan/aws-cdk-in-practice/pull/19) that captures the above changes.

