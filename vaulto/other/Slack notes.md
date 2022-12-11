# Slack notes
### rebase / merge master in #monorepo
`git pull --rebase origin master` and force push with `git push --force` 
also try
`git fetch`
`git rebase -i origin/master`


### install in #monorepo to subfolders
`yarn install -D`  installs using yarn workspaces

### re-run #GHA
`git commit --allow-empty -n -m "re-run checks" && git push`

### scaffold #Cypress with cly
Say you want to scaffold a new Cypress test:  
`npm i -D cypress`  
`npx @bahmutov/cly init` (may need to add -p here to install on the fly)  
Boom, you get a cypress.json file and the Cypress folder with a single example spec. 
Want to scaffold TypeScript specs?  
`npx @bahmutov/cly init --typescript`  
Want to scaffold a bare minimum project?  
`npx @bahmutov/cly init --bare (or -b)`

## package release troubles
You need super_admin
You can double check the file `~/.aws/config`
`ec aws creds init`
get super_admin
then you can release

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