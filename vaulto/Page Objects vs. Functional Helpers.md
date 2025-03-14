### **Page Objects vs. Functional Helpers**

  

A while ago, I published [Functional Programming Test Patterns with Cypress](https://dev.to/muratkeremozcan/functional-test-patterns-with-cypress-27ed), where I went in-depth on **why page objects are unnecessary in modern test automation**. Back then, I didn’t realize just how **ahead of its time** that take was—since many people still **insist on using page objects today**.

  

This post is a **simpler, updated version** of that argument.

  

---

  

### **🚀 Why Functional Helpers > Page Objects?**

  

The **Page Object Model (POM)** follows **inheritance**, while **functional helpers follow composition**.

  

But modern web apps are built with **component-based architecture**, where **components** are the real building blocks—not pages.

  

❓ **If components compose and pages are just collections of them, does it really make sense to abstract pages with classes?** Or does it introduce **unnecessary duplication and over-abstraction**?

  

In modern testing frameworks like **Playwright** and **Cypress**, strict **Page Object Model (POM)** is often **overkill**, especially when:

  

✅ You’re using **data selectors (`data-qa`, `data-cy`)** for stable locators.

✅ The tools already offer **powerful built-in utilities** for UI interactions.

✅ POM introduces **extra complexity that makes debugging harder**.

  

---

  

## **❌ Why Page Objects No Longer Make Sense**

  

🔙 Why POM Needed Classes in the Past

  

It’s worth acknowledging that POM wasn’t originally a bad idea—in the Selenium/WebDriver days, classes were necessary to manage workflows across different pages because test frameworks lacked built-in ways to handle:

• State → A base page maintained shared properties like driver (in Selenium) or page (in Playwright).

• Inheritance → Page classes extended a BasePage to inherit common navigation, helper methods, or interactions.

• Encapsulation → Pages wrapped UI interactions to abstract implementation details and expose only relevant actions.

  

At the time, this structure solved real problems—but modern frameworks already provide solutions for these things, making POM mostly redundant.

  

For example, Playwright already manages state with context.newPage(), provides a clean API for interactions, and supports direct functional composition without forcing abstraction layers.

  

📌 POM was useful when test frameworks were low-level, but today? The problems it solved either no longer exist or are solved better without it.

  

### **1️⃣ Unnecessary Abstraction**

  

- POM adds an extra **layer** that often **doesn’t provide feasible value**.

- Modern test frameworks are already **powerful enough** without it.

  

### **2️⃣ Base Page Inheritance is Overkill**

  

- Having a `BasePage` class with generic methods (`click()`, `fill()`) **just to wrap Playwright’s API** (or Cypress) makes no sense.

- Playwright (or Cy) **already** has `page.locator()`, `page.click()`, `page.fill()`, etc.

  

### **3️⃣ Harder Debugging**

  

- With **POM**, if a test fails, you **have to jump between multiple files** to figure out what went wrong.

- With **direct helper functions**, you **see exactly what’s happening**.

  

---

  

## **🔴 Traditional Page Object Model (POM)**

  

🚨 **Problems with POM:**

❌ **Unnecessary complexity** → Extra class & inheritance

❌ **Harder debugging** → Need to jump between files

❌ **Wrapping Playwright’s own API for no reason**

  

🔹 **Example (`LoginPage.js` - POM Approach)**

  

```js

class LoginPage {

constructor(page) {

this.page = page;

this.usernameField = page.locator('[data-testid="username"]');

this.passwordField = page.locator('[data-testid="password"]');

this.loginButton = page.locator('[data-testid="login-button"]');

}

  

async login(username, password) {

await this.usernameField.fill(username);

await this.passwordField.fill(password);

await this.loginButton.click();

}

}

  

export default LoginPage;

```

  

🔹 **Usage in a Test**

  

```js

import { test, expect } from "@playwright/test";

import LoginPage from "./LoginPage.js";

  

test("User can log in", async ({ page }) => {

const loginPage = new LoginPage(page);

await loginPage.login("testUser", "password123");

  

await expect(page.locator('[data-testid="welcome-message"]')).toHaveText(

"Welcome, testUser"

);

});

```

  

---

  

## **✅ Functional Helper Approach (Better)**

  

### **📌 Why is this better?**

  

✅ **No extra class** → Directly use Playwright API

✅ **No unnecessary `this.page` assignments**

✅ **Much easier to maintain & debug**

  

🔹 **Example (`loginHelpers.js` - Functional Helper Approach)**

  

```js

export async function login(page, username, password) {

await page.fill('[data-testid="username"]', username);

await page.fill('[data-testid="password"]', password);

await page.click('[data-testid="login-button"]');

}

```

  

🔹 **Usage in a Test**

  

```js

import { test, expect } from "@playwright/test";

import { login } from "./loginHelpers.js";

  

test("User can log in", async ({ page }) => {

await login(page, "testUser", "password123");

  

await expect(page.locator('[data-testid="welcome-message"]')).toHaveText(

"Welcome, testUser"

);

});

```

  

---

  

## **🔥 Final Thoughts**

  

Helper functions are **simpler**, **faster to debug**, and **scale better** in component-driven apps.

  

💡 **POM was useful in Selenium/WebDriver days, but today? Just use functions.**

  

🔥 **What do you think?** Are you still using POM? Have you already switched to functional helpers?

💬 **Drop a comment below**—I’d love to hear your take on this!

  

---

  

## Addendum

  

Thanks everyone for their contributions!

  

One thing I identified is that Component in testing isn’t used like in UI frameworks.

  

For frontend devs, a component is a reusable UI unit with state, props, and lifecycle hooks.

  

For QA, a component is often just a logical grouping of elements & interactions within a larger page.

  

🔹 ex: A checkout form in testing might represent the entire checkout UI, while in React, it might be broken into FormField, Button, AddressInput.

  

🔹 In POM, subcomponents are often instantiated as properties within a page object, mirroring the UI structure but without true reusability.

  

While I understand this approach, I don’t fully agree with the terminology.

  

I advocate for UI-component-driven testing, as I discuss in my book: https://muratkerem.gitbook.io/cctdd.

  

With tools like PW Component Testing, Cy Component Testing & Vitest with UI, we can now test at a lower level, reducing the need for full-page interactions.

  

Though still uncommon in QA, this shift solves many POM complexities:

  

✅ test components in isolation before e2e

✅ many cases covered at the UI component level

✅ smaller tests = quicker execution

  

We only move up the testing pyramid when we must: routing, user flows, backend calls.

  

---

  

I see that even the strongest advocates of POM now acknowledge its role has shifted—it’s mainly used to organize and store locators. But with AI-assisted test debugging advancing rapidly, it’s time to rethink this approach entirely.

  

Watch this [Playwright demo](https://www.youtube.com/live/NcSk9fOGEac) ~13 min mark by [Debbie O'Brien](https://www.linkedin.com/in/debbie-obrien/) & [Simon Knott](https://www.linkedin.com/in/simon-knott/)

  

They showcase an experimental Playwright config that outputs the DOM in a structured way, similar to [aria snapshots](https://playwright.dev/docs/aria-snapshots#aria-snapshots):

  

This creates a clean, machine-readable format (like YML) that represents the DOM in a concise way. The AI then analyzes the DOM snapshot, diagnoses the failure, and suggests stable ARIA selectors.

  

What does this mean for QA?

  

Instead of manually managing locators in a POM abstraction, we should adopt ARIA selectors as the standard for DOM interaction.

  

1.ARIA selectors provide a shared understanding between screen readers, test tools like Playwright, and AI.

  

2.Debugging can be AI-powered—rather than relying on a static POM file, AI can analyze the ARIA selector tree and suggest better selectors dynamically.

  

3.The DOM becomes the source of truth, not a manually maintained POM abstraction

  

If the argument is still **“store all selectors in a POM,” I disagree.** The future of test automation is **ARIA-driven**, where the **source code and AI-powered ARIA snapshots** become the **true source of truth**—not a manually curated **selectors file**.

  

---

  

This post took off, didn’t it?

  

The conversation in the comments has revealed perspectives I initially overlooked. One key takeaway is that many teams still use non-JS/TS languages (Java, C#) for UI testing, meaning they don’t even have the option of functional helpers—they are locked into POM and class-based structures by default.

  

I didn’t consider this because, for me, the core unwritten rule is that UI tests should be written in the same language as the source code. If the app is built in JavaScript/TypeScript, it makes sense to test it in JS/TS.

  

For teams transitioning from non-JS/TS to modern test frameworks, I now see how this post could feel completely foreign. In that case, the first step isn’t functional helpers vs. POM—it’s reevaluating whether Java/C# are the right tools for UI testing at all.