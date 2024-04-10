 
### rebase / merge master in #monorepo
`git pull --rebase origin main` and force push with `git push --force` 
also try
`git fetch`
`git rebase -i origin/master`


### install in #monorepo to subfolders
`yarn install -W`  installs using yarn workspaces

### re-run #GHA
`git commit --allow-empty -n -m "re-run checks" && git push`

### Jira commit cheat sheet

branch name:  `chore/DEVXTEST-666-foo-bar`
commit message: `chore: [DEVXTEST-666] foo bar`


\### scaffold #Cypress with cly
Say you want to scaffold a new Cypress test:  
`npm i -D cypress`  
`npx @bahmutov/cly init` (may need to add -p here to install on the fly)  
Boom, you get a cypress.json file and the Cypress folder with a single example spec. 
Want to scaffold TypeScript specs?  
`npx @bahmutov/cly init --typescript`  
Want to scaffold a bare minimum project?  
`npx @bahmutov/cly init --bare (or -b)`

## calculate number of commits 
`git log --since="2023-01-01" -p apps/portal > commits.txt` 
or 
`git log --since="2023-01-01" -p apps/portal | grep "commit " | wc -l`

## package release troubles
You need super_admin
You can double check the file `~/.aws/config`
`ec aws creds init`
get super_admin
then you can release

## automatically fix type import 
https://stackoverflow.com/questions/71080256/is-there-a-way-to-automatically-fix-import-type-errors-on-typescript-when-usin
add to rules:
```javascript
'@typescript-eslint/consistent-type-imports': 'error',
```
`yarn lint --fix`

## ec login notes
https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1431011740/AWS+Authentication


![[Pasted image 20221205132331.png]]


[how to stub react-redux useReducer hook - using cypress](https://stackoverflow.com/questions/66151947/how-to-stub-react-redux-usereducer-hook-using-cypress)

### flake element detached from the DOM error
don’t:  
```js 
cy.get('#cart input') // query command
  .eq(2) // query command
  .clear() // action command
  .type('20') // action command
```

do:  

```js
// merge the cy.get + cy.eq into a single query
const selector = '#cart input:nth(2)'

cy.get(selector).clear()

// query the input again to make sure has been cleared
cy.get(selector).should('have.value', '')

// type the new value and check
cy.get(selector).type('20')

cy.get(selector).should('have.value', '20')

```

try this out in the instances where cy.get().eq() is used. It might make things better


there is another solution , using aliases or helper functions.  
When it isn’t a 1:1 conversion from  `foo().eq()`  to `foo:nth()`you can try these outlet’s say you have a dom like this, which causes a detached element

```html
<div id="chain-example">
  <ul id="items">
    <li>Oranges</li>
    <li>Bananas</li>
  </ul>
</div>
<script>
  setTimeout(function () {
    // notice that we re-render the entire list
    // and insert the item at the first position
    document.getElementById('chain-example').innerHTML = `
      <ul id="items">
        <li>Grapes</li>
        <li>Apples</li>
      </ul>
    `
  }, 2000)
</script>
```

don’t:

```js
// DOES NOT WORK because it only retries the last command plus assertion
cy.get('#chain-example')
  .find('#items')
  .find('li')
  .should('have.length', 2)
  .first()  // RETRIES HERE 
  .should('have.text', 'Grapes'). 
```

We need retry the entire chain of commands from the root `cy.get` command.One way to achieve this by storing the querying chain as an alias.  
Cypress retries the entire chain if the aliased element becomes detached

```js
// notice how we create an alias to the element we want to check
// the entire chain of commands uses 4 different commands

cy.find('#items')
  .find('li')
  .should('have.length', 2)
  .first()
  .as('firstItem')

// when using the alias, if the element is detached from DOM
// the entire chain is recomputed (but non-querying commands are skipped)
cy.get('@firstItem').should('have.text', 'Grapes')
```

Here’s what I like better  

```js
const firstItem = () => cy.find('#items')
  .find('li')
  .should('have.length', 2)
  .first()

firstItem().should('have.text', 'Grapes')

```



these might be the solutions if you’re seeing element detached from the dom error

-   CodeCov demo for Rami
    -   how to view code coverage today ([Actions](https://github.com/helloextend/node-core/runs/7544057063?check_suite_focus=true))
    -   can we track coverage over time? (diff)
    -   how do we ensure coverage does not regress ([thresholds](https://github.com/helloextend/node-core/blob/master/services/contracts/jest.config.js))
    -   how do we ensure it keeps going up?
    -   how does codecov do it?
        -   [PR](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress/pull/193)
        -   [web ui](https://app.codecov.io/gh)


## unit vs e2e
one advice I have for you is one that I give to many people coming from unit testing and trying out e2e testing


Always look for opportunities to tweak what test is already existing as opposed to writing partially duplicated tests for new specs. The reason Cucumber / Gherkin is not great is this duplication; if every feature was mapped to a spec, there would be much duplication between the specs. What matters from a test perspective is the beginning state of a test; if reaching that state is common, then it is an opportunity for a test enhancement vs partial test duplication. At which point, the only caveat becomes the test duration for parallelization concerns.


for example, these 3 is really 1 test
![[Pasted image 20220727224332.png]]
this one does everything above

![[Pasted image 20220727224354.png]]

the next block is the same idea, they do exactly the same thing with less code and effort
![[Pasted image 20220727224438.png]]


![[Pasted image 20220727224506.png]]





### Regex
Learn regex  
[regexone.com](http://regexone.com/)  
[regexlearn.com](http://regexlearn.com/)

Test regex  
[regex101.com](http://regex101.com/)  
[regexr.com](http://regexr.com/)

Help writing regex  
[regex.help](http://regex.help/)

Copy/paste regex  
[projects.lukehaas.me/regexhub/](http://projects.lukehaas.me/regexhub/)  
[ihateregex.io](http://ihateregex.io/)  
[uibakery.io/regex-library](http://uibakery.io/regex-library)


## React event types
-   event  `onChange`: `React.ChangeEvent<HTMLInputElement> 
-   submit event `onSubmit`: `React.FormEvent`
-   click event  `onClick`: `React.MouseEvent<HTMLButtonElement>`


## Return object from a Cypress command
![[Pasted image 20220909074734.png]]


## Flip method based on argument
Instead of all the duplication, the whole thing can be customized with an argument added to the original `upsertOrder` function.

Give it a default 'post' or 'put', and return the http request accordingly.

Here is an example:
```js
/**
 * Upserts an order with the given token, store ID, line items, API version, and order overrides.
 * @param {object} options - An object containing options for the request.
 * @param {string} options.token - The authentication token.
 * @param {string} options.storeId - The ID of the store.
 * @param {Array<object>} options.lineItems - An array of line item objects containing product information.
 * @param {string} [options.httpMethod='PUT'] - The HTTP method to use, either 'put' or 'post'.
 * @param {string} [options.apiVersion='latest'] - Optional API version to use.
 * @param {object} [options.orderOverrides={}] - Optional overrides to apply to the generated order object.
 * @returns {object} The response object from the API.*/
export function upsertOrder({
  token,
  storeId,
  lineItems,
  httpMethod = 'PUT',
  apiVersion = 'latest',
  ordersOverrides = {},
}) {
  const params = {
    headers: makeHeaders(token, apiVersion, {'X-Idempotency-Key': uuidv4()}),
  }
  const payload = generateOrder(storeId, lineItems, ordersOverrides)

  // KEY
  const httpMethods = {
    PUT: http.put,
    POST: http.post,
  }

  if (!httpMethods.hasOwnProperty(httpMethod)) {
    throw new Error(
      `Invalid HTTP method: ${httpMethod}. Allowed methods: put, post.`,
    )
  }

  return httpMethods[httpMethod](`${baseUrl}/orders`, payload, params)
}
```

We can further reduce the duplication by making the test (the original) `order.js` just include an extra line.

    const orderResPut = upsertOrder({token, storeId, lineItems, httpMethod: 'PUT'})
    checkResponse(orderResPut)

    const orderResPost = upsertOrder({token, storeId, lineItems, httpMethod: 'POST'})
    checkResponse(orderResPost)

That's around 8 lines extra for the feature, and 2 lines for the test. Pretty much nothing else has to change compared to `main`; just the readme needs a tweak.



## Github wants SSH now, can't push https 
Set ssh for the repo:
`git remote set-url origin git@github.com:muratkeremozcan/aws-cdk-in-practice.git


## deployments as of Sep 2023
Yup right now the following behaviors are as followed:  

1. Developer creates a PR and merges it to `main`
2. Deployment occurs to `dev` automatically
3. Right now we **manually** kick off a deployment to `stage` and prepare for `prod`
4. These [tests](https://github.com/helloextend/authentication-service/blob/main/src/infrastructure/config/custom-actions.ts) are automatically kicked off after `stage` deploy has landed successfully
5. Test reports are posted within deployment channels [#cy-deployment-security](https://extend-workspace.slack.com/archives/C05GQUHTYR5)
6. After all of the tests looks good with a **manual** verification, we can **manually** approve the production gates and deploy.
   
   
### sinon vs cypress-map + spok   
Do you like the abstraction with Sinon or the concretion with cypress-map and cy-spok?
```js
cy.get('@someEvent')
  .should('be.calledWith', 
    'property', 
    {
      type: 'checkbox',
      values: ['bar']
    }
   )


import 'cypress-map'
import spok from 'cy-spok'

cy.get('@someEvent')
  .invoke('getCalls')
  .map('args')
  .should(
    spok([
      [
        'property',
        {
          type: 'checkbox',
          values: ['bar'],
        }
      ]
    ])
  )
```


### Stubbing the event handler

A technique for stubbing or spying on the behavior of a function prop within a React component during a Cypress test. Specifically, it concerns the `onChange` event of the `Select` component. Here's a breakdown and the significance of this pattern:

```javascript
const activeOption = 'a to z'
const selectOption = cy.stub().as('selectOption')
const select = (e) => selectOption(e.target.value)
```

1. **Initialization of `activeOption`**: 
   - `const activeOption = 'a to z'`
   This is just setting up a default active option for the `Select` component for the test.

2. **Stubbing with Cypress**: 
   - `const selectOption = cy.stub().as('selectOption')`
   Here, we're creating a stub using Cypress and giving it an alias (`selectOption`). This stub will serve as a "fake" function that we can use to check if it's been called, how many times it's been called, and what arguments it was called with.

3. **Stubbed Event Handler**:
   - `const select = (e) => selectOption(e.target.value)`
   This is an event handler that will be passed to the `Select` component for its `onChange` prop. Instead of doing any real action, it simply calls the stubbed function (`selectOption`) with the value of the selected option. This is crucial because it allows us to observe and assert interactions with the component.

Use it when  **checking Prop Callbacks**: If a component expects a callback as a prop (like `onChange` in this case), this technique allows you to ensure that the callback is indeed being called with the right arguments.

## process.env with Zod
![[Screenshot 2024-02-23 at 8.27.49 AM.png]]

## Retryable before hook in Cypress
```js
/**
 * A `before()` alternative that gets run when a failing test is retried.
 *
 * By default cypress `before()` isn't run when a test below it fails
 * and is retried. Because we use `before()` as a place to setup state
 * before running assertions inside `it()` this means we can't make use
 * of cypress retry functionality to make our suites more reliable.
 *
 * https://github.com/cypress-io/cypress/issues/19458
 * https://stackoverflow.com/questions/71285827/cypress-e2e-before-hook-not-working-on-retries
 */
export const retryableBefore = (fn) => {
  let shouldRun = true;

  // we use beforeEach as cypress will run this on retry attempt
  // we just abort early if we detected that it's already run
  beforeEach(() => {
    if (!shouldRun) return;
    shouldRun = false;
    fn();
  });

  // When a test fails we flip the `shouldRun` flag back to true
  // so when cypress retries and runs the `beforeEach()` before
  // the test that failed, we'll run the `fn()` logic once more.
  Cypress.on('test:after:run', (result) => {
    if (result.state === 'failed') {
      if (result.currentRetry < result.retries) {
        shouldRun = true;
      }
    }
  });
};

describe('my suite', () => {
  retryableBefore(() => {
    // reset database and seed with test data …

    cy.visit('/some/page');
  });

  it('my test 2', () => {
    …
  });

  it('test 2', () => {
    …
  });
  
  describe('my suite', () => {
    retryableBefore(() => {
      // do something in ui
    });

    it('my test 3', () => {
      …
    });

    it('test 4', () => {
      …
    });
  });
});

```

## Time zone in Cypress component or CT tests, local vs CI

Local vs CI can have a difference, because the CI runs in a different time zone.
For this, the time zone needs to be standardized.
Use an env var:
`TZ='Etc/GMT' npm run cypress open`

If that doesn't work, you could try something like this:
```js
module.exports = (on, config) => {
  on('before:browser:launch', (browser = {}, launchOptions) => {
      launchOptions.args.push('--lang=en-US')
      launchOptions.args.push('--timezone=Etc/GMT')
      return launchOptions;
    })
}
```

## Setting a standard time zone with Cypress

 `TZ='Etc/GMT' npm run cypress open` 
 
 if that doesn't work, you could try something like this:

```js
module.exports = (on, config) => {
  on('before:browser:launch', (browser = {}, launchOptions) => {
      launchOptions.args.push('--lang=en-US')
      launchOptions.args.push('--timezone=Etc/GMT')
      return launchOptions;
    })
}
```

If that doesn't work, you could try standardizing the time zone in the app, some libs have that option.

## Cypress drag and drop

```js
const dragAndDrop = (dragLocator, dropLocator) => {
  cy.get(dragLocator)
    .realMouseDown({ button: 'left', position: 'center' })
    .realMouseMove(0, 10, { position: 'center' })
    .wait(200);
  cy.get(dropLocator)
    .realMouseMove(0, 0, { position: 'center' })
    .realMouseUp();
};
```