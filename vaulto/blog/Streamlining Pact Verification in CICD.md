# Streamlining Pact Verification in CI/CD with GitHub Actions: Extracting Consumer Branch Names and Dynamic Provider Testing

Pact contract testing facilitates integration by allowing consumers and providers to agree on a contract. However, automating this process in a Continuous Integration/Continuous Deployment (CI/CD) pipeline can present challenges, especially when dealing with multiple branches and dynamic environments.

In this blog post, I'll walk you through the problem I faced while automating Pact verification using GitHub Actions, how I identified the root cause, and the step-by-step reasoning behind the solution.

You can find the webhook yml implementation at my [sample repository](https://github.com/muratkeremozcan/pact-js-example-provider/blob/main/.github/workflows/webhook.yml). With a few modifications of the parameters, it should be usable anywhere. 

TL, DR; use the linked webhook.yml to setup webhooks with GHA.

## **The Problem: Dynamic Branch Verification in CI/CD**

When a consumer publishes a new Pact to the Pact Broker, I wanted a GitHub Actions workflow to automatically trigger provider tests against the **corresponding consumer** branch. This ensures that any changes made in the consumer are immediately validated against the provider's implementation. Upon closer inspection, I realized that may not always be the case, due to the plethora of conveniences and helper utilities created in the previous blog posts. 

We may run into a situation where a branch is prematurely tested against only main, while not being covered against the matching branch, if there is one. An edge case that can go unnoticed!

The challenge was to:

1. **Extract the consumer branch name** from the Pact Broker.
2. **Check out the corresponding branch** in the provider repository.
3. **Run the provider tests** against the Pact from the specific consumer branch.

Initial attempts resulted in failures because the consumer branch name was not being correctly extracted, leading to the provider tests running against the wrong branch or defaulting to `main`.

## **Identifying the Root Cause**

Upon inspecting the logs, I noticed that the `CONSUMER_BRANCH` variable was being set to `null`, even though the Pact Broker's JSON response contained the branch information.

```json
CONSUMER_VERSION_JSON: {
  "number": "0f871ff",
  "createdAt": "2024-10-29T20:40:33+00:00",
  "_embedded": {
    "branchVersions": [
      {
        "name": "test/better-webhook",
        "latest": true,
        "_links": {
          "self": {
            "title": "Branch version",
            "name": "test/better-webhook",
            "href": "***"
          }
        }
      }
    ],
    "tags": []
  },
  "_links": { ... }
}
Consumer branch extracted: null
```

The issue was with the `jq` command used to parse the JSON. The command was trying to extract a `branch` field that didn't exist at the top level of the JSON structure.

## **The Solution: Correcting the JSON Path with jq**

To extract the branch name correctly, I needed to adjust the `jq` command to point to the correct location in the JSON:

```bash
CONSUMER_BRANCH=$(echo "$CONSUMER_VERSION_JSON" | jq -r '._embedded.branchVersions[0].name')
```

### **Why Use jq?**

`jq` is a lightweight and flexible command-line JSON processor. It allows us to parse and extract data from JSON responses easily, which is essential when dealing with API responses in shell scripts.

### **Step-by-Step Reasoning**

1. **Fetch the Pact JSON from the Pact Broker:**

   ```bash
   PACT_JSON=$(curl -s -H "Authorization: Bearer $PACT_BROKER_TOKEN" "$PACT_PAYLOAD_URL")
   ```

2. **Extract the Consumer Version URL:**

   ```bash
   CONSUMER_VERSION_URL=$(echo "$PACT_JSON" | jq -r '._links."pb:consumer-version".href')
   ```

3. **Fetch the Consumer Version JSON:**

   ```bash
   CONSUMER_VERSION_JSON=$(curl -s -H "Authorization: Bearer $PACT_BROKER_TOKEN" "$CONSUMER_VERSION_URL")
   ```

4. **Extract the Consumer Branch Name:**

   ```bash
   CONSUMER_BRANCH=$(echo "$CONSUMER_VERSION_JSON" | jq -r '._embedded.branchVersions[0].name')
   ```

5. **Handle Cases Where the Branch Might Not Exist:**

   ```bash
   if [ "$CONSUMER_BRANCH" = "null" ] || [ -z "$CONSUMER_BRANCH" ]; then
     echo "No consumer branch found. Defaulting to 'main'."
     CONSUMER_BRANCH="main"
   fi
   ```

By correctly pointing to `._embedded.branchVersions[0].name`, we can reliably extract the consumer branch name.

## **Optimizing the GitHub Actions Workflow**

With the consumer branch name correctly extracted, I also optimized the GitHub Actions workflow to simplify the Git operations.

### **Original Approach: Manual Git Configuration**

Initially, I manually configured Git within the workflow:

```yml
- name: Configure Git
  run: |
    git config --global init.defaultBranch main
    git init
    git remote add origin https://github.com/your-org/your-provider-repo.git
```

This approach required additional steps to manage the repository and branches, which could be error-prone and redundant.

### **Improved Approach: Leveraging actions/checkout**

I removed the manual Git configuration and utilized the `actions/checkout` action's capabilities to handle repository checkout and branch management more efficiently.

#### **Updated Workflow Snippet**

```yml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
    repository: ${{ env.GITHUB_REPO_OWNER }}/${{ env.GITHUB_REPO_NAME }}

- name: Checkout matching branch or default to main
  shell: bash
  run: |
    git fetch --all
    if git ls-remote --exit-code --heads origin "${CONSUMER_BRANCH}"; then
      echo "Branch '${CONSUMER_BRANCH}' exists in the remote repository."
      git checkout "${CONSUMER_BRANCH}"
      echo "GITHUB_BRANCH=${CONSUMER_BRANCH}" >> $GITHUB_ENV
    else
      echo "Branch '${CONSUMER_BRANCH}' does not exist in the remote repository. Using 'main' branch."
      git checkout main
      echo "GITHUB_BRANCH=main" >> $GITHUB_ENV
    fi
```

### **Reasons for the Change**

- **Simplification:** By removing manual Git steps, the workflow becomes cleaner and less prone to errors.
- **Efficiency:** `actions/checkout` handles authentication and repository setup, reducing the need for custom configuration.
- **Flexibility:** Fetching all branches allows us to switch to the appropriate branch based on the consumer's branch.

## **Outcome**

After implementing these changes, the workflow successfully:

- Extracts the correct consumer branch name.
- Checks out the corresponding branch in the provider repository.
- Runs the provider tests against the correct Pact version.

The [logs](https://github.com/muratkeremozcan/pact-js-example-provider/actions/runs/11582557082/job/32245833402) confirm that the branch was correctly identified and used:

```bash
Consumer branch extracted: test/better-webhook2

Branch 'test/better-webhook2' exists in provider repository.
```

## **Conclusion**

Automating Pact verification in a CI/CD pipeline requires careful handling of dynamic variables and repository management. By correctly parsing JSON responses with `jq` and optimizing the GitHub Actions workflow, I streamlined the process of verifying Pacts against the correct provider branch.