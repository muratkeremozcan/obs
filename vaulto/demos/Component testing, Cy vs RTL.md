-   [ToH](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts") - RTL vs CyCT examples  
      
    
    -   HeaderBarBrand [rtl](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.test.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.test.tsx") vs [cy](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.cy.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.cy.tsx")
        
        -   Just mount the component to see it. _"Seeing the component" : running the component test vs the whole app_
            
        -   [cy.mount](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/cypress/support/component.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/cypress/support/component.tsx") vs [render](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/test-utils.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/test-utils.tsx") : similar to [App.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/App.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/App.tsx"). _The rest of the api is the same as e2e_
            
        -   1:1 with RTL: same test intent, different api style. [_Cypress has 4 assertion styles_](https://www.youtube.com/watch?v=5w6T5xB_SzI "https://www.youtube.com/watch?v=5w6T5xB_SzI")_, one of them is just like RTL (and it is the worst option)_
            
        -   TDD: explore the app through tests. _You have all the Cypress debugging tools, same developer experience as e2e._  
              
            
    -   NavBar: [rtl](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/NavBar.test.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/NavBar.test.tsx") vs [cy](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/NavBar.cy.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/NavBar.cy.tsx")
        
        -   Run the test and ad-hoc the component. _Inspect elements, console, network, React DevTools…_
            
        -   [example of a bug during development](https://slides.com/muratozcan/cctdd#/9/14 "https://slides.com/muratozcan/cctdd#/9/14")
            
        -   The difference in the APIs is more clear as complexity gets higher  
              
            
    -   Heroes: [rtl](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.test.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.test.tsx") vs [cy](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.cy.tsx "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.cy.tsx")
        
        -   We can mock the same way as Jest in Cy component tests (which uses Sinon) _But we should_ [_avoid testing implementation details_](https://slides.com/muratozcan/cctdd#/9/11 "https://slides.com/muratozcan/cctdd#/9/11") _and use MSW or cy.intercept_
            
        -   [MSW](https://mswjs.io/docs/ "https://mswjs.io/docs/") == [cy.intercept](https://docs.cypress.io/api/commands/intercept "https://docs.cypress.io/api/commands/intercept") _200 scenarios are similar, only the MSW vs cy.intercept is different_
            
        -   Controlling the clock: testing in the dark vs seeing every step _Testing as an afterthought vs the test tool supporting our design_
            
-   Zen
    
    -   Every test has a [story](https://ui.dev.extend.com/?path=/story/overview--page "https://ui.dev.extend.com/?path=/story/overview--page") _Run Zen component tests with_ `yarn cy:open-ct-zen`
        
    -   We get the same [Dashboard & analytics](https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1538916361/Client+Component+Testing+-+11+21+2022# "#")
        
    -   We get the same ecosystem; ex: [visual tests](https://percy.io/782cfcaf/zen "https://percy.io/782cfcaf/zen")  
          
        
    -   [Code coverage](https://muratkerem.gitbook.io/cctdd/ch30-appendix/combined-code-coverage "https://muratkerem.gitbook.io/cctdd/ch30-appendix/combined-code-coverage")
        
        Start zen `yarn cy:run-ct-zen-fast`  
        Check the folder `packages/zen/coverage-cy`
        
        _TDD will boost code coverage naturally,_ _CodeCov helps spot dead code & redundant tests_  
          
        
        [_We can combine the coverage from component tests, with Jest unit test, and e2e tests._](https://muratkerem.gitbook.io/cctdd/ch30-appendix/combined-code-coverage "https://muratkerem.gitbook.io/cctdd/ch30-appendix/combined-code-coverage")
        
        ![](blob:https://helloextend.atlassian.net/67419fe8-754f-4a75-84a4-e73feb74e495#media-blob-url=true&id=17b6e1c0-84be-4d9b-9774-8ca7df3b6ecc&collection=contentId-1538916361&contextId=1538916361&mimeType=image%2Fpng&name=image-20221021-185442.png&size=34535&width=558&height=224&alt=)
        
        ![](blob:https://helloextend.atlassian.net/45c02fce-e504-4487-8636-ce6b1326b2f5#media-blob-url=true&id=3c83f3e9-b8c7-4432-91b7-7a2a3ae4c2eb&collection=contentId-1538916361&contextId=1538916361&mimeType=image%2Fpng&name=image-20221021-185501.png&size=156350&width=2066&height=1090&alt=)
        
    -   What to test where:
        
        -   Always try to test at the lowest level with highest confidence. With CT you can do a lot at a low level.
            
        -   Move up when you cannot confidently test; to the parent, to ui-integration tests, to ui-e2e.
            
        -   Parent-child relationships is best covered with component tests.
            
        -   Unrelated components on the same DOM: best fit for ui-integration tests.
            
        -   Routing and state management (outside the component) might be a better fit for e2e. Your CT suite serves as a regression assurance at that point.
            
        -   Need the backend? Use UI-e2e, but careful with duplicated confidence. _“If A works, can B fail?”_
            
        -   You might begin preferring CT over e2e in new features, might be also moving e2e tests to CT. We've seen a proportion close to 1 e2e : 5 ui-integration : 15 component tests in multiple mid-sized projects.  
              
            

-   _**if you want to start today**_
    
    -   [CCTDD gitBook](https://muratkerem.gitbook.io/cctdd/ "https://muratkerem.gitbook.io/cctdd/")
        
    -   [tour-of-heroes-react-cypress-ts - the final app built in the book](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts "https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts")
        
    -   [react-hooks-in-action-with-cypress - the final app built in React Hooks in Action book](https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress "https://github.com/muratkeremozcan/react-hooks-in-action-with-cypress")
        
    -   [bookshelf - the final app from Kent Dodds' Epic React](https://github.com/muratkeremozcan/bookshelf "https://github.com/muratkeremozcan/bookshelf")
        
    -   [365+ Cypress component test examples](https://github.com/muratkeremozcan/cypress-react-component-test-examples "https://github.com/muratkeremozcan/cypress-react-component-test-examples")