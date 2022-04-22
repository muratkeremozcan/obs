# E2E test Definition of Done (DoD)
-   no flake
    
-   no hard waits/sleeps
    
-   stateless, multiple entities can execute - cron job or semaphore where not possible
    
-   no order dependency; each _it/describe/context_ block can run with .only in isolation
    
-   tests handle their own state and clean up after themselves - deleted or de-activated entities
    
-   tests live near the source code
    
-   shifted left, as possible - begins with local server or sandbox, works throughout deployments
    
-   low/minimal maintenance
    
-   enough testing per feature to give us release confidence
    
-   execution evidence in CI
    
-   some visibility, as in a test report