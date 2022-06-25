[![](https://developers.google.com/drive/images/drive_icon.png)https://drive.google.com/file/d/1grRnjoqi8C62Mr0EilOh6M5__BsHz1IR/view?usp=sharing - Connect to preview](https://drive.google.com/file/d/1grRnjoqi8C62Mr0EilOh6M5__BsHz1IR/view?usp=sharing)

-   the before
    
    _webDriver superTest & jest_
    
    -   ui testing
        
        -   manual test setup using Postman
            
        -   local ui test execution (NO CI!)
            
    -   api testing
        
        -   difficult failure diagnosis
            
        -   flake
            
    -   some of our [DoD](https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1353711882/E2E+test+Definition+of+Done+DoD "https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1353711882/E2E+test+Definition+of+Done+DoD") is possible, but the above are not
        
-   test plugins
    
    _reduce code & effort duplication between teams_
    
    -   [cypress-auth](https://github.com/helloextend/cypress-auth "https://github.com/helloextend/cypress-auth")
        
    -   [cypress-store](https://github.com/helloextend/cypress-store "https://github.com/helloextend/cypress-store")
        
    -   [cypress-product](https://github.com/helloextend/cypress-product "https://github.com/helloextend/cypress-product")
        
    -   [cypress-contract](https://github.com/helloextend/cypress-contract "https://github.com/helloextend/cypress-contract")
        
    -   [cypress-claim](https://github.com/helloextend/cypress-claim "https://github.com/helloextend/cypress-claim")
        
    -   [cypress-lead](https://github.com/helloextend/cypress-lead "https://github.com/helloextend/cypress-lead")
        
    -   [test-package-consumer](https://github.com/helloextend/test-package-consumer "https://github.com/helloextend/test-package-consumer")
        
    -   show how to do it: [how to create internal test plugins](https://dev.to/muratkeremozcan/how-to-create-an-internal-test-plugins-for-your-team-in-ts-implement-custom-commands-and-use-other-cypress-plugins-in-them-5lp "https://dev.to/muratkeremozcan/how-to-create-an-internal-test-plugins-for-your-team-in-ts-implement-custom-commands-and-use-other-cypress-plugins-in-them-5lp")
        
    -   teaches the domain
        
-   after
    
    _Cypress_
    
    -   ui
        
        -   api setup with plugins + ui e2e
            
        -   [test methodology](https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1353711882/E2E+test+Definition+of+Done+DoD "https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1353711882/E2E+test+Definition+of+Done+DoD") & test architecture possibilities _(ex:_ [_ui integration tests_](https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1341325600/E2E+Integration+Test+Strategy+Q1+2022 "https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1341325600/E2E+Integration+Test+Strategy+Q1+2022")_,_ [_component testing_](https://youtu.be/koEEYxtWUMs "https://youtu.be/koEEYxtWUMs")_)_
            
    -   api
        
        _applies to UI as well_
        
        -   0 flake possible
            
        -   next level DeVex & TDD
            
        -   reliable, fast, cost effective, fault-finding _check out why Cypress for_ [_API e2e testing event driven systems_](https://dev.to/muratkeremozcan/api-testing-event-driven-systems-7fe "https://dev.to/muratkeremozcan/api-testing-event-driven-systems-7fe")
            
-   [e2e onboarding](https://helloextend.atlassian.net/wiki/spaces/ENG/pages/1354400102 "/wiki/spaces/ENG/pages/1354400102") (live demo)
    
-   [external example](https://dev.to/muratkeremozcan/crud-api-testing-a-deployed-service-with-cypress-using-cy-api-spok-cypress-data-session-cypress-each-4mlg "https://dev.to/muratkeremozcan/crud-api-testing-a-deployed-service-with-cypress-using-cy-api-spok-cypress-data-session-cypress-each-4mlg")
    
    -   The 4 horseman of Cypocalypse
        
        -   [cy-api](https://github.com/bahmutov/cy-api "https://github.com/bahmutov/cy-api")
            
        -   [cy-spok](https://github.com/bahmutov/cy-spok "https://github.com/bahmutov/cy-spok")
            
        -   [cypress-data-session](https://github.com/bahmutov/cypress-data-session "https://github.com/bahmutov/cypress-data-session")
            
        -   [cypress-each](https://github.com/bahmutov/cypress-each "https://github.com/bahmutov/cypress-each")