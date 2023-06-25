# Quotes
## Dev
-   Testing, Engineering and the Scientific Method are all bound together. Engineering is about assuming stuff will go wrong and learn from that (then write tests) and predict the ways it might go wrong and defend against that (and write more tests).
-   There is a thin line between tech debt and feature debt. Tech debt is anything that impacts the engineering team. Lacking quality in tests is not tech debt, because it impacts customers. Period.
-   Story point estimates must account for development, testing, as well as documentation - whatever it takes to put that feature in front of the customer or the consumer API. Story pointing is an estimation of complexity; having to write tests does not change that because complexity is still the same. The **velocity** of the team will change and normalize over time. That is the real velocity of the team which fulfills an industry standard definition of done.
-   Writing tests and running them as you implement the feature will make your more productive
-   Shared mutable state is the source of all evil
-   However many tests you have, you can never prove your software is good. But one failing test means your software isn't good enough. Therefore use proven [[Test Methodologies]] to gain the highest confidence with the miminal investment.
-   Engineering is about learning, not already having all the answers. Good engineers optimize to be great at learning. You do that with short feedback cycles and iterating quickly.
-   Your undivided attention is the most precious currency in the universe.
-   Lean: maximum output with minimum work
-   You can't make a baby in a month with 9 women
-   As the [[Quality]] of our system reduced, our ability to change it reduces too
-   The more you tests resemble the way your software is used, the more confidence they give you. 
-   #Coverage is an assessment for the thoroughness or completeness of testing with respect to a model. Our model can be unit coverage, feature coverage, #mutation score, combinatorial coverage, non-functional requirement coverage, anything!
-   Pessimists sound smart, optimists make money
-   Fallacy of false authority - expert in 1 field...
-   If you aim at nothing, you hit nothing
-   Doing the correct thing poorly is worth more than executing the wrong things perfectly
-   When you lose your money, you lose nothing. When you lose your health, you lose something. When you lose your character, you lose everything.
-   ... once battle-testing has defeated real-world issues and edge cases ...
-   ... trade offs that it made are no longer valuable
-   a test doesn't deliver value until it fails
-   what you can avoid testing is more important than what you are testing
-   forget the mistake, remember the lesson
-   tests that don't give return on investment in terms of confidence
-   We don't get to do naive UI e2e before there is confidence that the backend works. Better to use an API test client, closer to the backend code and deployments, vs our late UI e2e. At that point we need to be careful with test duplication, use minimal UI e2e & only fill in the gaps.  
There aren't too many test frameworks that support that test strategy.. [Cypress.io](https://www.linkedin.com/company/cypress.io/) is one.
*   Always look for opportunities to tweak what test is already existing as opposed to writing partially duplicated tests for new specs. The reason Cucumber / Gherkin is not great is this duplication; if every feature was mapped to a spec, there would be much duplication between the specs. What matters from a test perspective is the beginning state of a test; if reaching that state is common, then it is an opportunity for a test enhancement vs partial test duplication. At which point, the only caveat becomes the test duration for parallelization concerns.
*  As technology and your organization change, your context changes too. Many of the things that you might once consider as best practice can easily become anti-patterns. For example, monorepos work great when you are a small team, but by the time you grow to hundreds or perhaps thousands of engineers, monorepos present many challenges that require complex solutions to address.

* My golden rules in testing: - It’s always cost v confidence - cost = creation + execution + maintenance - What you can avoid testing is more important than what you are testing
*   The debate on Page Object vs module pattern is really just Inheritance vs Composition.  Inheritance (PO) is great for describing what something is; the page has x, y, z on it.  Composition (module pattern) is great for describing what something does. Which one do you think best fits component based architecture, where components are the building blocks and get reused on pages? How about user flows?
*   Study for 1 hour every morning. Even in the most ideal job, you won't be learning more than 500 hours a year, because most of it is busy work and grind. If you invest an hour a day in gaining knowledge, in 5 years you will build 10 years worth of relative tech
*   When you have purpose, you have work-life integration. Find your purpose, and never work a day in your life. Help others build their self-defined purpose, and you rarely have to people-manage.
*   The ability to hold a holistic picture in mind and help other team members see it, collaboratively navigate towards it, changing direction as new ideas arise has been a hallmark of great engineers.
*   (about try hard mocking) The mess in such tests is telling us that the design isn’t right but, instead of fixing the problem by improving the code, we have to carry the **extra complexity** in both code and test
* We prefer to have the end-to-end tests exercise both the system and the **process by which it’s built and deployed**
*   ascribing greater significance to a problem than it should have in reality
*   Business demands may overload teams with too much work, driving them to optimize for feature delivery. This is a terrible metric for success. It drives engineers to shortcut and so make bad decisions that end up with them going slower over time, not faster. Developers often try to please business by reducing the quality of their work. If you've ever given an estimate that didn't include testing or refactoring, you did this.
*   Quality is a whole team responsibility. From a developer's perspective a significant part of that responsibility firmly sits with us; it is not something we can successfully abdicate to other people. If you see testing as somebody else's responsibility, QA as the gatekeepers of quality, your manager's job to tell you how much quality you can put in your code, you should think again.
*   The first principal is that you must not fool yourself, and you're the easiest person to fool.
*  "Simplicity is the ultimate sophistication". Because it is very easy to write confusing, complex code. But when you build software that is easy to understand, easy to read and write, easy to learn, it is a true achievement that must be celebrated.
*   Whoever is going to perform in the office will perform remotely. Whoever is going to pretend to seem busy and not produce anything worthwhile still is going to continue that, doesn't matter where they are.
*   **Murphy’s Law of Debugging:** The thing you believe so deeply can’t possibly be wrong so you never bother testing it, is definitely where you’ll find the bug after you pound your head on your desk and change it only because you’ve tried everything else you can possibly think of.
*   cognitive load: if our brain is overloaded focusing on unimportant details, it is harder to reason about higher order thinking
*   No plan survives contact with the enemy. No software survives contact with the users.
*   The reasons why most people push back are cultural. They’re about familiarity and comfort. About not wanting to look or feel stupid in front of others.
*   Most engineers find pw syntax simpler than cy because they do not get functional pipeline of commands, and have no idea how to debug or reason about it. Unless they are familiar with functional programming, the shift in mental modal is painful, hence the preference to pw.
*   TypeScript saves you so much time by preventing many programming mistakes & runtime errors, leaving you more time to do other things such as deciphering & fixing insignificant TypeScript errors. (David Khorshid)
* `this` keyword in JavaScript will burn you one day. Then it will burn you again and again and again. If Dante Alighieri were alive today, he would put writing object-oriented JavaScript among one of the first levels of Hell for sure. ![[Pasted image 20220316143024.png]]


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
- More harm is caused in this world by stupidity and ignorance than outright evil. The incompetent and stupid are far more dangerous than those who are overtly evil, because we are never quite sure where they are leading us, until it is too late.
-   Develop your sense of self worth from internal standards, and not incessant comparisons.
