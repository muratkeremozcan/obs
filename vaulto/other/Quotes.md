# Quotes
## Dev
-   Testing, Engineering and the Scientific Method are all bound together. Engineering is about assuming stuff will go wrong and learn from that (then write tests) and predict the ways it might go wrong and defend against that (and write more tests).
-   There is a thin line between tech debt and feature debt. Tech debt is anything that impacts the engineering team. Lacking quality in tests is not tech debt, because it impacts customers. Period.
-   Story point estimates must account for development, testing, as well as documentation - whatever it takes to put that feature in front of the customer or the consumer API. Story pointing is an estimation of complexity; having to write tests does not change that because complexity is still the same. The **velocity** of the team will change and normalize over time. That is the real velocity of the team which fulfills an industry standard definition of done.
-   Writing tests and running them as you implement the feature will make your more productive
-   Shared mutable state is the source of all evil
-   However many tests you have, you can never prove your software is good. But one failing test means your software isn't good enough. Therefore use proven [[Test Methodologies]] to gain the highest confidence with the miminal investment.
-   Engineering is about learning, not already having all the answers. Good engineers optimize to be great at learning. You do that with short feedback cycles and iterating quickly.
-   Lean: maximum output with minimum work
-   You can't make a baby in a month with 9 women
-   As the [[Quality]] of our system reduced, our ability to change it reduces too
-   The more you tests resemble the way your software is used, the more confidence they give you. 
-   #Coverage is an assessment for the thoroughness or completeness of testing with respect to a model. Our model can be unit coverage, feature coverage, #mutation score, combinatorial coverage, non-functional requirement coverage, anything!
-   Pessimists sound smart, optimists make money
-   If you aim at nothing, you hit nothing
-   When you lose your money, you lose nothing. When you lose your health, you lose something. When you lose your character, you lose everything.
-   ... once battle-testing has defeated real-world issues and edge cases ...
-   ... trade offs that it made are no longer valuable
-   a test doesn't deliver value until it fails
-   what you can avoid testing is more important than what you are testing
-   We don't get to do naive UI e2e before there is confidence that the backend works. Better to use an API test client, closer to the backend code and deployments, vs our late UI e2e. At that point we need to be careful with test duplication, use minimal UI e2e & only fill in the gaps.  
There aren't too many test frameworks that support that test strategy.. [Cypress.io](https://www.linkedin.com/company/cypress.io/) is one.
*   Always look for opportunities to tweak what test is already existing as opposed to writing partially duplicated tests for new specs. The reason Cucumber / Gherkin is not great is this duplication; if every feature was mapped to a spec, there would be much duplication between the specs. What matters from a test perspective is the beginning state of a test; if reaching that state is common, then it is an opportunity for a test enhancement vs partial test duplication. At which point, the only caveat becomes the test duration for parallelization concerns.
* My golden rules in testing: - It’s always cost v confidence - cost = creation + execution + maintenance - What you can avoid testing is more important than what you are testing
*   The debate on Page Object vs module pattern is really just Inheritance vs Composition.  Inheritance (PO) is great for describing what something is; the page has x, y, z on it.  Composition (module pattern) is great for describing what something does. Which one do you think best fits component based architecture, where components are the building blocks and get reused on pages? How about user flows?
*   Study for 1 hour every morning. Even in the most ideal job, you won't be learning more than 500 hours a year, because most of it is busy work and grind. If you invest an hour a day in gaining knowledge, in 5 years you will build 10 years worth of relative tech
*   When you have purpose, you have work-life integration. Find your purpose, and never work a day in your life. Help others build their self-defined purpose, and you rarely have to people-manage.
*   The ability to hold a holistic picture in mind and help other team members see it, collaboratively navigate towards it, changing direction as new ideas arise has been a hallmark of great engineers.
*   Business demands may overload teams with too much work, driving them to optimize for feature delivery. This is a terrible metric for success. It drives engineers to shortcut and so make bad decisions that end up with them going slower over time, not faster. Developers often try to please business by reducing the quality of their work. If you've ever given an estimate that didn't include testing or refactoring, you did this.
*   Quality is a whole team responsibility. From a developer's perspective a significant part of that responsibility firmly sits with us; it is not something we can successfully abdicate to other people. If you see testing as somebody else's responsibility, QA as the gatekeepers of quality, your manager's job to tell you how much quality you can put in your code, you should think again.
*   The first principal is that you must not fool yourself, and you're the easiest person to fool.
*   **Murphy’s Law of Debugging:** The thing you believe so deeply can’t possibly be wrong so you never bother testing it, is definitely where you’ll find the bug after you pound your head on your desk and change it only because you’ve tried everything else you can possibly think of.
* `this` keyword in JavaScript will burn you one day. Then it will burn you again and again and again. If Dante Alighieri were alive today, he would put writing object-oriented JavaScript among one of the first levels of Hell for sure. ![[Pasted image 20220316143024.png]]

## ACD
-   strong opinions, weakly held
-   be alert for possibilities of paragraphing
-   most notice a single grain of sand, others see the flow through the hourglass
-   ...talk that dances around thorns 
-   small men talk big 
-   if you're not upsetting anyone, you're not changing the status quo
- -   one part of leadership is allowing others to do a worse job than you would
-   leadership of creative people is mostly about enabling creative communication
-   in danger of too violently agreeing with each other in the course of this conversation
-   what's the point of it all if all you want is to be liked by everyone and avoid trouble. The only way I got any place was by breaking some of the rules.
