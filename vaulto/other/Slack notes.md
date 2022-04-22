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

[how to stub react-redux useReducer hook - using cypress](https://stackoverflow.com/questions/66151947/how-to-stub-react-redux-usereducer-hook-using-cypress)
