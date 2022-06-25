[![](https://developers.google.com/drive/images/drive_icon.png)https://drive.google.com/drive/u/0/folders/1o15bA9gPJETiVsLpKGwJYJeMzJIKj9Gr - Connect to preview](https://drive.google.com/drive/u/0/folders/1o15bA9gPJETiVsLpKGwJYJeMzJIKj9Gr)  
  

-   Test Burn-in  
    [Flake Management](https://dashboard.cypress.io/projects/484wzy/analytics/flaky-tests "https://dashboard.cypress.io/projects/484wzy/analytics/flaky-tests") & [Top failures](https://dashboard.cypress.io/projects/484wzy/analytics/top-failures "https://dashboard.cypress.io/projects/484wzy/analytics/top-failures")
    
-   reasons for failures
    
    -   _test flake_ _(generally close to 0%)_
        
    -   _top failures_
        
        -   real Service/App failure
            
        -   environment instability
            
        -   system dependency failure
            
-   old way: [cron jobs](https://crontab.guru/#0_1/2_*_*_6,7 "https://crontab.guru/#0_1/2_*_*_6,7") _(shifting right is costly)_
    
-   [the official feature is coming](https://cypress.io/pricing/?utm_adgroup=132501525560&utm_keyword=cypress%20dashboard%20pricing&utm_source=google&utm_medium=cpc&utm_campaign=15312994475&utm_term=cypress%20dashboard%20pricing&hsa_acc=8898574980&hsa_cam=15312994475&hsa_grp=132501525560&hsa_ad=562694869917&hsa_src=g&hsa_tgt=kwd-1203766167349&hsa_kw=cypress%20dashboard%20pricing&hsa_mt=e&hsa_net=adwords&hsa_ver=3&gclid=Cj0KCQiApL2QBhC8ARIsAGMm-KE_wx6Uof61HVpn5DqhGUas9_FMUwvYOxAWcqfPpjBKOatCfDuW-4kaAhubEALw_wcB "https://cypress.io/pricing/?utm_adgroup=132501525560&utm_keyword=cypress%20dashboard%20pricing&utm_source=google&utm_medium=cpc&utm_campaign=15312994475&utm_term=cypress%20dashboard%20pricing&hsa_acc=8898574980&hsa_cam=15312994475&hsa_grp=132501525560&hsa_ad=562694869917&hsa_src=g&hsa_tgt=kwd-1203766167349&hsa_kw=cypress%20dashboard%20pricing&hsa_mt=e&hsa_net=adwords&hsa_ver=3&gclid=Cj0KCQiApL2QBhC8ARIsAGMm-KE_wx6Uof61HVpn5DqhGUas9_FMUwvYOxAWcqfPpjBKOatCfDuW-4kaAhubEALw_wcB")
    
-   [grep](https://dev.to/muratkeremozcan/the-32-ways-of-selective-testing-with-cypress-a-unified-concise-approach-to-selective-testing-in-ci-and-local-machines-1c19 "https://dev.to/muratkeremozcan/the-32-ways-of-selective-testing-with-cypress-a-unified-concise-approach-to-selective-testing-in-ci-and-local-machines-1c19")
    
    -   local ([copy paste from readme](https://github.com/helloextend/client#grep-cheat-sheet "https://github.com/helloextend/client#grep-cheat-sheet"))
        
    -   CI: [node-core](https://github.com/helloextend/node-core/actions/workflows/auth-repeat-spec.yml "https://github.com/helloextend/node-core/actions/workflows/auth-repeat-spec.yml") [client](https://github.com/helloextend/client/actions/workflows/customers-repeat-spec-local.yml "https://github.com/helloextend/client/actions/workflows/customers-repeat-spec-local.yml")
        
    -   examples
        
        -   [test flake](https://github.com/helloextend/node-core/runs/5203221935?check_suite_focus=true#step:8:531 "https://github.com/helloextend/node-core/runs/5203221935?check_suite_focus=true#step:8:531"), [fixing it](https://github.com/helloextend/node-core/runs/5216504900?check_suite_focus=true#step:8:435 "https://github.com/helloextend/node-core/runs/5216504900?check_suite_focus=true#step:8:435")
            
        -   [real failure](https://github.com/helloextend/cypress-contract/runs/5159848209?check_suite_focus=true#step:8:89 "https://github.com/helloextend/cypress-contract/runs/5159848209?check_suite_focus=true#step:8:89")
            
        -   _"offer tests are not working, something !$%^& with stores"_
            
            -   [test stores in isolation](https://github.com/helloextend/cypress-store/runs/5239063394?check_suite_focus=true#step:8:62 "https://github.com/helloextend/cypress-store/runs/5239063394?check_suite_focus=true#step:8:62")
                
            -   [test offers on sandbox](https://github.com/helloextend/node-core/runs/5239578575?check_suite_focus=true#step:8:731 "https://github.com/helloextend/node-core/runs/5239578575?check_suite_focus=true#step:8:731")
                
            -   [test offers on dev](https://github.com/helloextend/node-core/runs/5239524279?check_suite_focus=true#step:8:726 "https://github.com/helloextend/node-core/runs/5239524279?check_suite_focus=true#step:8:726")
                
-   [GHA reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows "https://docs.github.com/en/actions/using-workflows/reusing-workflows")
    
-   [external demo](https://www.youtube.com/watch?v=m03ru99eBuc "https://www.youtube.com/watch?v=m03ru99eBuc") & repos ([mono](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqa0I3NFJmM3p1b3l1VDZwWFk2RkZweGw4aDM3QXxBQ3Jtc0ttT1RDQzgxZXhiYUQySFIzdjFxeVo1XzB5V1p6aGJpbjFfWG1WRzlLTjFwRm5JMFd4dEhtbUVXVmhGd2pldzRYRGg0MFJhV1pJUlRwZVRvMG5ueWs4TjNiUExiUmdxQWJpX00wbC1lZFVwMDFlZkl5TQ&q=https%3A%2F%2Fgithub.com%2Fmuratkeremozcan%2Flerna-react-ts-cypress "https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqa0I3NFJmM3p1b3l1VDZwWFk2RkZweGw4aDM3QXxBQ3Jtc0ttT1RDQzgxZXhiYUQySFIzdjFxeVo1XzB5V1p6aGJpbjFfWG1WRzlLTjFwRm5JMFd4dEhtbUVXVmhGd2pldzRYRGg0MFJhV1pJUlRwZVRvMG5ueWs4TjNiUExiUmdxQWJpX00wbC1lZFVwMDFlZkl5TQ&q=https%3A%2F%2Fgithub.com%2Fmuratkeremozcan%2Flerna-react-ts-cypress"), [single](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbURUZHJ3VkZ5NnFsa09JVzJoNFNxSFFZckxod3xBQ3Jtc0trcjhfdmhpWW5xUHE0VmxjbFh6cHBkRDlDR1JqeG1saHBaR0FUV2ZiRUxrdEdDczljQklhSjg5SjZEaURSRkpuam1xSWFhanVCWFc5QUlRRWVWSFRGN05Sdmh1SkZiTVJxRlpUdUxMaUVEZjNfcF92TQ&q=https%3A%2F%2Fgithub.com%2Fmuratkeremozcan%2Freact-hooks-in-action-with-cypress "https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbURUZHJ3VkZ5NnFsa09JVzJoNFNxSFFZckxod3xBQ3Jtc0trcjhfdmhpWW5xUHE0VmxjbFh6cHBkRDlDR1JqeG1saHBaR0FUV2ZiRUxrdEdDczljQklhSjg5SjZEaURSRkpuam1xSWFhanVCWFc5QUlRRWVWSFRGN05Sdmh1SkZiTVJxRlpUdUxMaUVEZjNfcF92TQ&q=https%3A%2F%2Fgithub.com%2Fmuratkeremozcan%2Freact-hooks-in-action-with-cypress"))
    

-   Email testing
    
    -   what to test in an email
        
        -   validating email fields; from, to, cc, bcc, subject, attachments.
            
        -   HTML content and links in the email
            
        -   Spam checks
            
    -   statelessness
        
        -   stateless users: [Gmail tricks](https://www.idownloadblog.com/2018/12/19/gmail-email-address-tricks/ "https://www.idownloadblog.com/2018/12/19/gmail-email-address-tricks/") _(ok if we don't care for email content)_
            
        -   problems
            
            -   bouncing emails to cloud service
                
            -   email flagged for spam (too much e2e, or load tests)
                
            -   not able to differentiate between emails being received
                
                -   every spec has to have a unique email -> stateful -> resort to cron jobs or semaphores
                    
            -   unreliable email speeds, which add up in CI costs, and more importantly engineer feedback time.
                
    -   What do we need?
        
        -   [Unique email servers per service and app so that there is a predictable inbox.](https://mailosaur.com/app/login?redirect=%2Fapp%2F "https://mailosaur.com/app/login?redirect=%2Fapp%2F")
            
        -   Being able to create (unlimited) users on the fly and have emails sent to them
            
        -   Receiving the emails fast
            
        -   Being able to verify the content of those emails effortlessly.
            
        -   spam check
            
    -   [external blog & repo](https://dev.to/muratkeremozcan/test-emails-effortlessly-with-cypress-mailosaur-and-cy-spok-56lm "https://dev.to/muratkeremozcan/test-emails-effortlessly-with-cypress-mailosaur-and-cy-spok-56lm")