Cypress component testing was released about a year prior to this blog post and has been a disruptor not only to the de-facto JS framework component testing solutions but also initiated a paradigm shift in developing frontend components. At the core of this is the ability to render a component in isolation in the actual browser, and to see and observe how the component behaves in isolation while time-travel debugging via Cypress. This changes the entire development experience, with testing - the actual intent - being the icing on the cake.

Rolling out component testing in a community that is so invested in de-facto component testing solutions can be a daring challenge. In this post, we will analyze the differences between React Testing Library (RTL) and Cypress Component Testing (CyCT), and provide you with resources that can help with adopting component testing in your organization.

**Toc:**

- [CyCT vs RTL examples - Tour of Heroes (ToH)](#cyct-vs-rtl-examples---tour-of-heroes-toh)
  - [HeaderBarBrand component](#headerbarbrand-component)
  - [InputDetail component](#inputdetail-component)
  - [NavBar component](#navbar-component)
- [Comparison of low level spies \& mocks: Sinon vs Jest](#comparison-of-low-level-spies--mocks-sinon-vs-jest)
  - [Sinon vs Jest: Spy](#sinon-vs-jest-spy)
  - [Sinon vs Jest: Stub/Mock](#sinon-vs-jest-stubmock)
- [Comparison of network spies \& mocks: `cy.intercept` vs `MSW`](#comparison-of-network-spies--mocks-cyintercept-vs-msw)
- [CyCT vs RTL examples - Epic React](#cyct-vs-rtl-examples---epic-react)
- [Wrapping up](#wrapping-up)
- [Addendum: Gleb Bahmutov's The Missing Comparison Part video: a comparison of the developer experience](#addendum-gleb-bahmutovs-the-missing-comparison-part-video-a-comparison-of-the-developer-experience)
  - [1. Compare the devex making a simple breaking change to the source code: HeaderBarBrand](#1-compare-the-devex-making-a-simple-breaking-change-to-the-source-code-headerbarbrand)
  - [2. Compare test stability by simulating an asynchronous process: InputDetail](#2-compare-test-stability-by-simulating-an-asynchronous-process-inputdetail)
  - [3. Compare the devex making a "complex" change: Heroes](#3-compare-the-devex-making-a-complex-change-heroes)

### Our experience at Extend

Our first component test at Extend was at the end of 2022, and we have been piloting it in one of our apps for about six months. Comparing the Cypress Cloud reports between then and now, we can see a preference towards component testing.

Bear in mind that we are migrating from Enzyme to CyCT, to be able to upgrade React beyond version 16. If we had RTL tests already in this app - as we do in a second UI app which had already migrated out of Enzyme before CyCT rollout - then the RTL tests and CyCT could coexist. We have about 130 Enzyme tests remaining to migrate in our pilot app. This means we will potentially have around 300 Cypress component tests once the migration is complete. This is aligned with the proportion of e2e to CT we have observed in other apps; usually it is anywhere between 1:3 to 1:5.

December 2022 (first component test commit):

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hkq5eoj2d8u12op15fi3.png)

June 2023:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b962ejpe1nntx9kgiiz5.png)

> If you are curious about how the e2e execution time became so low with more tests, check out the video [Improve Cypress e2e test latency by a factor of 20!!](https://www.youtube.com/watch?v=diB6-jikHvk)

## CyCT vs RTL examples - [Tour of Heroes](https://github.com/muratkeremozcan/tour-of-heroes-react-vite-cypress-ts) (ToH)

ToH is the final app built in the book [CCTDD: Cypress Component Test Driven Design](https://muratkerem.gitbook.io/cctdd/). It has a few dozen Cypress component tests and their RTL mirrors. We will cover a few examples to showcase the main differences.

### [HeaderBarBrand component](https://muratkerem.gitbook.io/cctdd/ch03-headerbarbrand)

The way the component is mounted is very similar between CyCT and RTL. So are custom [mounts](https://slides.com/muratozcan/cctdd#/3/9) / [renders](https://slides.com/muratozcan/cctdd#/3/10).

There are less imports in CyCT, because either these are built-in or come with the browser.

The API is the primary difference; with Cypress we have a left-to-right chain style, with RTL we have a right-to-left variable assignment style.

28 lines CyCT vs 35 lines in RTL.

![HeaderBarBrand](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/13icumizquvedq2vgkjn.png)

[HeaderBarBrand.cy.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.cy.tsx)

```tsx
import HeaderBarBrand from "./HeaderBarBrand";
import { BrowserRouter } from "react-router-dom";

describe("HeaderBarBrand", () => {
  beforeEach(() => {
    cy.mount(
      <BrowserRouter>
        <HeaderBarBrand />
      </BrowserRouter>
    );
  });

  it("should verify external link attributes", () => {
    cy.get("a")
      .should("have.attr", "href", "https://reactjs.org/")
      .and("have.attr", "target", "_blank")
      .and("have.attr", "rel", "noopener noreferrer");
    cy.getByCy("header-bar-brand").within(() => cy.get("svg"));
  });

  it("should verify internal link spans and navigation", () => {
    cy.getByCy("navLink").within(() =>
      ["TOUR", "OF", "HEROES"].map((part: string) => cy.contains("span", part))
    );
    cy.getByCy("navLink").click();
    cy.url().should("contain", "/");
  });
});
```

[HeaderBarBrand.test.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.test.tsx)

```tsx
import HeaderBarBrand from "./HeaderBarBrand";
import { render, screen, within } from "@testing-library/react";
import { BrowserRouter } from "react-router-dom";
import userEvent from "@testing-library/user-event";
import "@testing-library/jest-dom";

describe("HeaderBarBrand", () => {
  beforeEach(() => {
    render(
      <BrowserRouter>
        <HeaderBarBrand />
      </BrowserRouter>
    );
  });
  it("should verify external link attributes", async () => {
    const link = await screen.findByTestId("header-bar-brand-link");
    expect(link).toHaveAttribute("href", "https://reactjs.org/");
    expect(link).toHaveAttribute("target", "_blank");
    expect(link).toHaveAttribute("rel", "noopener noreferrer");

    // not easy to get a tag with RTL, needed to use a test id
    within(await screen.findByTestId("header-bar-brand")).getByTestId(
      "react-icon-svg"
    );
  });

  it("should verify internal link spans and navigation", async () => {
    const navLink = await screen.findByTestId("navLink");
    const withinNavLink = within(navLink);
    ["TOUR", "OF", "HEROES"].forEach((part) => withinNavLink.getByText(part));

    await userEvent.click(navLink);
    expect(window.location.pathname).toBe("/");
  });
});
```

### [InputDetail component](https://muratkerem.gitbook.io/cctdd/ch05-inputdetail)

We see similar contrast between CyCT and RTL as before; similar mount/render, different API styles, terser syntax on Cypress side. Note that testing library has a Cypress version and can be used to make the examples even more similar.

The key difference we want to point out is stubbing the `onChange` property the component. `cy.stub()` vs its counterpart `jest.fn()`. Cypress comes with Sinon, and Cypress' API allows us to stub in-line.

```tsx
// CyCT: we can stub the property in-line
cy.mount(
  <InputDetail
    name={name}
    value={value}
    placeholder={placeholder}
    onChange={cy.stub().as("onChange")}
  />
);

// RTL: variable assignment first
const onChange = jest.fn();

render(
  <InputDetail
    name={name}
    value={value}
    placeholder={placeholder}
    onChange={onChange}
  />
);
```

![InputDetailComponent](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fo5i22l7tv5e13kssy0m.png)

[InputDetail.cy.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/InputDetail.cy.tsx)

```tsx
import InputDetail from "./InputDetail";
import "@testing-library/cypress/add-commands";

describe("InputDetail", () => {
  const placeholder = "Aslaug";
  const name = "name";
  const value = "some value";
  const newValue = "42";

  it("should allow the input field to be modified", () => {
    cy.mount(
      <InputDetail
        name={name}
        value={value}
        placeholder={placeholder}
        onChange={cy.stub().as("onChange")}
      />
    );

    cy.contains("label", name);
    cy.findByPlaceholderText(placeholder).clear().type(newValue);
    cy.findByDisplayValue(newValue).should("be.visible");
    cy.get("@onChange").its("callCount").should("eq", newValue.length);
  });

  it("should not allow the input field to be modified", () => {
    cy.mount(
      <InputDetail
        name={name}
        value={value}
        placeholder={placeholder}
        readOnly={true}
      />
    );

    cy.contains("label", name);
    cy.findByPlaceholderText(placeholder)
      .should("have.value", value)
      .and("have.attr", "readOnly");
  });
});
```

[InputDetail.test.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/InputDetail.test.tsx)

```tsx
import InputDetail from "./InputDetail";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

describe("InputDetail", () => {
  const placeholder = "Aslaug";
  const name = "name";
  const value = "some value";
  const newValue = "42";

  it("should allow the input field to be modified", async () => {
    const onChange = jest.fn();
    render(
      <InputDetail
        name={name}
        value={value}
        placeholder={placeholder}
        onChange={onChange}
      />
    );

    await screen.findByText(name);
    const inputField = await screen.findByPlaceholderText(placeholder);
    await userEvent.clear(inputField);
    await userEvent.type(inputField, newValue);
    expect(inputField).toHaveDisplayValue(newValue);
    expect(onChange).toHaveBeenCalledTimes(newValue.length);
  });

  it("should not allow the input field to be modified", async () => {
    render(
      <InputDetail
        name={name}
        value={value}
        placeholder={placeholder}
        readOnly={true}
      />
    );

    await screen.findByText(name);
    const inputField = await screen.findByPlaceholderText(placeholder);
    expect(inputField).toHaveDisplayValue(value);
    expect(inputField).toHaveAttribute("readOnly");
  });
});
```

### [NavBar component](https://muratkerem.gitbook.io/cctdd/ch08-navbar)

We can observe the same similarities and contrast between CyCT and RTL in this component. The most striking of them all is the API style difference; with Cypress we are able to cover a each link with `forEach`, to be able to do the same in RTL we have to do a little bit of more work with `it.each`.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3gj6egrrkyz6h6iq9hcz.png)

[NavBar.cy.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/NavBar.cy.tsx)

```tsx
import NavBar from "./NavBar";
import { BrowserRouter } from "react-router-dom";

const routes = ["heroes", "villains", "boys", "about"];

describe("NavBar", () => {
  it("should navigate to the correct routes", () => {
    cy.mount(
      <BrowserRouter>
        <NavBar />
      </BrowserRouter>
    );

    cy.contains("p", "Menu");
    cy.getByCy("menu-list").children().should("have.length", routes.length);

    routes.forEach((route: string) => {
      cy.get(`[href="/${route}"]`)
        .contains(route, { matchCase: false })
        .click()
        .should("have.class", "active-link")
        .siblings()
        .should("not.have.class", "active-link");

      cy.url().should("contain", route);
    });
  });
});
```

[NavBar.test.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/NavBar.test.tsx)

```tsx
import NavBar from "./NavBar";
import { render, screen, within, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { BrowserRouter } from "react-router-dom";
import "@testing-library/jest-dom";

const routes = ["Heroes", "Villains", "Boys", "About"];

describe("NavBar", () => {
  beforeEach(() => {
    render(
      <BrowserRouter>
        <NavBar />
      </BrowserRouter>
    );
  });

  it("should verify route layout", async () => {
    expect(await screen.findByText("Menu")).toBeVisible();

    const menuList = await screen.findByTestId("menu-list");
    expect(within(menuList).queryAllByRole("link").length).toBe(routes.length);

    routes.forEach((route) => within(menuList).getByText(route));
  });

  it.each(routes)("should navigate to route %s", async (route: string) => {
    const link = async (name: string) => screen.findByRole("link", { name });
    const activeRouteLink = await link(route);
    userEvent.click(activeRouteLink);
    await waitFor(() => expect(activeRouteLink).toHaveClass("active-link"));
    expect(window.location.pathname).toEqual(`/${route.toLowerCase()}`);

    const remainingRoutes = routes.filter((r) => r !== route);
    remainingRoutes.forEach(async (inActiveRoute) => {
      expect(await link(inActiveRoute)).not.toHaveClass("active-link");
    });
  });
});
```

In all these examples we have not said anything about the component code, but instead shared a CyCT screen shot to communicate what the component is about. On one side we are staring at html, on the other we are looking at the component in the browser. The picture demonstrates a bug during development that we caught by just looking at the component.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2b5vbold5ljtr0cc788p.png)

## Comparison of low level spies & mocks: Sinon vs Jest

We prepared a [repository](https://github.com/muratkeremozcan/sinon-vs-jest) to cover core examples of using Sinon with Cypress, and created mirror tests in Jest.

> The original Sinon examples are from Gleb Bahmutov's [Cypress Examples](https://t.co/LgUA1YB1Cx).

### [Sinon vs Jest](https://github.com/muratkeremozcan/sinon-vs-jest): Spy

A spy does not modify the behavior of the function - it is left perfectly intact. A spy is most useful when you are testing the contract between multiple functions and you don't care about the side effects the real function may create (if any).

[spy-practice.cy.ts](https://github.com/muratkeremozcan/sinon-vs-jest/blob/main/cypress/e2e/spy-practice.cy.ts) vs [spy-practice-test.ts](https://github.com/muratkeremozcan/sinon-vs-jest/blob/main/src/spy-practice.test.ts) are linked for you to view the files in comparison; we will summarize the key differences:

- **Spying on methods**: Both libraries allow you to create a spy that wraps around a function and records all calls to it, along with arguments and return values. Both Jest and Sinon allow you to assert if the spy was called, how many times it was called, and with what arguments. They do differ in syntax and some functionalities.

  `cy.spy(obj, 'method')` vs `jest.spyOn(obj, 'method')`

- **Syntax**: `have.been.called` vs `toHaveBeenCalled`, `have.been.calledTwice` vs `toHaveBeenCalledTimes(2)`

- **Matchers**: `sinon.match.string` vs `expect.any(String)`

- **Custom matchers**: [easy with Sinon](https://github.com/muratkeremozcan/sinon-vs-jest/blob/d596daa1a507707dcff33907052dd1081a718d41/cypress/e2e/spy-practice.cy.ts#L127), [not great devex with Jest](https://github.com/muratkeremozcan/sinon-vs-jest/blob/d596daa1a507707dcff33907052dd1081a718d41/src/spy-practice.test.ts#L104). We will not share the Jest code, because it is very verbose.

```ts
// Cy + sinon
const { match } = Cypress.sinon;
const isEven = (x: number) => x % 2 === 0;
const isOdd = (x: number) => x % 2 === 1;

const spy = cy.spy(calculator, "add").as("add");
calculator.add(2, 3);

// expect the value to pass a custom predicate function
// the second argument to "match(predicate, message)"
// is shown if the predicate does not pass and assertion fails
expect(spy).to.be.calledWith(match(isEven), match(isOdd, "is odd"));
```

- **Asynchronous testing**: In Jest, it is required to explicitly wait for asynchronous operations to complete before assertions are checked. This is illustrated in using the `setTimeout` and `await new Promise` combination. Cypress + Sinon does not necessitate this explicit wait. There are certain advantages to Cypress chain style and built-in retry ability. In the below example with Jest we have to gather the promises, `await` them with `Promise.all` and have to use `toHaveBeenNthCalledWith` to specify the order.

```ts
// Cy + Sinon
it("resolved value (promises)", () => {
  const calc = {
    async add(a: number, b: number) {
      return /* await */ Cypress.Promise.resolve(a + b).delay(100); // don't use await redundantly
    },
  };

  cy.spy(calc, "add").as("add");
  // wait for the promise to resolve then confirm its resolved value
  cy.wrap(calc.add(4, 5)).should("equal", 9);
  cy.wrap(calc.add(1, 90)).should("equal", 91);
  cy.wrap(calc.add(-5, -8)).should("equal", -13);

  // example of confirming one of the calls used add(4, 5)
  cy.get("@add").should("have.been.calledWith", 4, 5);
  cy.get("@add").should("have.been.calledWith", 1, 90);
  cy.get("@add").should("have.been.calledWith", -5, -8);

  // now let's confirm the resolved values
  // first we need to wait for all promises to resolve
  cy.get("@add")
    .its("returnValues")
    .then((ps) => Promise.all(ps))
    .should("deep.equal", [9, 91, -13]);
});
```

```ts
// Jest
it("resolved value (promises)", async () => {
  const calc = {
    async add(a: number, b: number) {
      return a + b;
    },
  };

  const spy = jest.spyOn(calc, "add");

  // Let's gather the promises first
  const promises = [calc.add(4, 5), calc.add(1, 90), calc.add(-5, -8)];

  // Now we wait for all the promises to resolve
  const results = await Promise.all(promises);

  // We can check if the spy was called with the correct arguments at each call
  expect(spy).toHaveBeenNthCalledWith(1, 4, 5);
  expect(spy).toHaveBeenNthCalledWith(2, 1, 90);
  expect(spy).toHaveBeenNthCalledWith(3, -5, -8);

  // Finally, we can verify the resolved values
  expect(results).toEqual([9, 91, -13]);
});
```

### [Sinon vs Jest](https://github.com/muratkeremozcan/sinon-vs-jest): Stub/Mock

A stub is a way to modify a function and delegate control over its behavior to you (the programmer).

Create a standalone stub (generally for use in unit test):

```js
cy.stub();
jest.fn();
```

Replace obj.method() with a stubbed function:

```js
cy.stub(obj, "method");
jest.spyOn(obj, "foo").mockImplementation(jest.fn());
```

Force obj.method() to return a value:

```js
cy.stub(obj, "method").returns("Cliff");
jest.spyOn(obj, "method").mockReturnValue("Cliff");
```

Force obj.method() when called with "bar" argument to return "foo":

```js
cy.stub(obj, "method").withArgs("bar").returns("foo");

jest.spyOn(obj, "method").mockImplementation((arg) => {
  if (arg === "bar") return "foo";
});
```

Force obj.method() to return a promise which resolves to "foo"

```js
cy.stub(obj, "method").resolves("foo");

jest.spyOn(obj, "method").mockImplementation(() => {
  return Promise.resolve("foo");
});
```

Force obj.method() to return a promise rejected with an error

```js
cy.stub(obj, "method").rejects(new Error("foo"));

jest.spyOn(obj, "method").mockImplementation(() => {
  return Promise.reject(new Error("foo"));
});
```

It is interesting to note that the equivalent of `cy.stub()` is `jest.fn()` but in many of the comparisons we are using `jest.spyOn(...).mockImplementation(...)`

In Jest, we can use `.mockImplementation()` to provide a custom implementation for the mock function.
In Sinon, we can use `.callsFake()` or `.returns()` to specify custom behavior for the stub.

`jest.fn()` can be used more in scenarios where you're not spying on or modifying existing object methods, but rather creating standalone mock functions. For instance, when testing if a function passed as a prop or callback is called correctly in a component test or when needing to create a mock implementation for a function from a module that your function under test is calling.

Here is an example scenario where jest.fn() could be used

```js
it("should call the callback", () => {
  const mockCallback = jest.fn();

  function doSomething(callback: (arg: string) => void) {
    callback("test argument");
  }

  doSomething(mockCallback);

  expect(mockCallback).toHaveBeenCalledTimes(1);
  expect(mockCallback).toHaveBeenCalledWith("test argument");
});
```

Comparing [stub-practice.cy.ts](https://github.com/muratkeremozcan/sinon-vs-jest/blob/main/cypress/e2e/stub-practice.cy.ts) vs [stub-practice.test.ts](https://github.com/muratkeremozcan/sinon-vs-jest/blob/main/src/stub-practice.test.ts), here are some of the other highlights:

**Restoring the original method after stub:**

```ts
const person = {
  getName() {
    return "Joe";
  },
};

/// Cy + Sinon
expect(person.getName()).to.eq("Joe");

const stub = cy.stub(person, "getName").returns("Cliff");
expect(person.getName()).to.eq("Cliff");

// restore the original method
stub.restore();
expect(person.getName()).to.eq("Joe");

/// Jest
expect(person.getName()).toBe("Joe");

const stub = jest.spyOn(person, "getName").mockReturnValue("Cliff");
expect(person.getName()).toBe("Cliff");

// restore the original method
stub.mockRestore();
expect(person.getName()).toBe("Joe");
```

**Matchers: .callThrough(), withArgs(), match.type, match(predicate)**:

```ts
describe("matchers: .callThrough(), withArgs(), match.type, match(predicate)", () => {
  const { match } = Cypress.sinon;

  it("Matching stub depending on arguments", () => {
    const greeter = {
      greet(name: string | number | undefined) {
        return `Hello, ${name}!`;
      },
    };

    const stub = cy.stub(greeter, "greet");

    stub.callThrough(); // if you want non-matched calls to call the real method
    stub.withArgs(match.string).returns("Hi, Joe!");
    stub.withArgs(match.number).throws(new Error("Invalid name"));

    expect(greeter.greet("World")).to.equal("Hi, Joe!");
    expect(() => greeter.greet(42)).to.throw("Invalid name");
    expect(greeter.greet).to.have.been.calledTwice;

    // non-matched calls goes the actual method
    expect(greeter.greet()).to.equal("Hello, undefined!");
  });
});
```

There is no direct equivalent in Jest, but the below does the same thing

```ts
describe("matchers: .mockImplementation()", () => {
  it("Matching stub depending on arguments", () => {
    const greeter = {
      greet(name: string | number) {
        return `Hello, ${name}!`;
      },
    };

    jest
      .spyOn(greeter, "greet")
      .mockImplementation((name: string | number | undefined) => {
        if (typeof name === "string") {
          return "Hi, Joe!";
        } else if (typeof name === "number") {
          throw new Error("Invalid name");
        } else {
          return "Hello, undefined!";
        }
      });

    expect(greeter.greet("World")).toEqual("Hi, Joe!");
    expect(() => greeter.greet(42)).toThrow("Invalid name");
    expect(greeter.greet).toHaveBeenCalledTimes(2);

    expect(greeter.greet()).toEqual("Hello, undefined!");
  });
});
```

**Calling the original method from the stub**:

```ts
describe("Call the original method from the stub: callsFake(...), wrappedMethod()", () => {
  it("Sometimes you might want to call the original method from the stub and modify it", () => {
    const person = {
      getName() {
        return "Joe";
      },
    };

    cy.stub(person, "getName").callsFake(() => {
      // call the real person.getName()
      return person.getName.wrappedMethod().split("").reverse().join("");
    });

    expect(person.getName()).to.eq("eoJ");
  });
});
```

There is no direct equivalent in Jest, but the below does the same thing.

```ts
describe("Call the original method from the stub: mockImplementation(), originalName", () => {
  it("Sometimes you might want to call the original method from the stub and modify it", () => {
    const person = {
      getName() {
        return "Joe";
      },
    };

    const originalGetName = person.getName.bind(person);

    jest.spyOn(person, "getName").mockImplementation(() => {
      return originalGetName().split("").reverse().join("");
    });

    expect(person.getName()).toEqual("eoJ");
  });
});
```

**Controlling time** `cy.clock` vs `jest.useFakeTimers`:

When running Cypress tests, the tests themselves are outside the application's iframe. When you use `cy.clock()` command you change the application clock, and not the spec's clock.

```ts
describe("cy.clock", () => {
  it("control the time in the browser", () => {
    const specNow = new Date();
    const now = new Date(Date.UTC(2017, 2, 14)).getTime();

    cy.clock(now) // sets the application clock and pause time
      .then(() => {
        // spec clock keeps ticking
        const specNow2 = new Date();
        // confirm by comparing the timestamps in milliseconds
        expect(+specNow2, "spec timestamps").to.be.greaterThan(+specNow);
      });
    // but the application's time is frozen
    cy.window()
      .its("Date")
      .then((appDate) => {
        const appNow = new appDate();
        expect(+appNow, "application timestamps")
          .to.equal(+now)
          .and.to.equal(1489449600000); // the timestamp in milliseconds
      });
    // we can advance the application clock by 5 seconds
    cy.tick(5000);
    cy.window()
      .its("Date")
      .then((appDate) => {
        const appNow = new appDate();
        expect(+appNow, "timestamp after 5 synthetic seconds").to.equal(
          1489449605000
        );
      })
      // meanwhile the spec clock only advanced by probably less than 200ms
      .then(() => {
        const specNow3 = new Date();
        expect(+specNow3, "elapsed on the spec clock").to.be.lessThan(
          +specNow + 200
        );
      });
  });
});
```

The Jest mirror cannot be run in the browser window, and has a slightly different approach.

```ts
describe("jest.useFakeTimers", () => {
  it("control the time in the browser", () => {
    jest.useFakeTimers();
    const specNow = new Date();
    const now = new Date(Date.UTC(2017, 2, 14)).getTime();

    jest.setSystemTime(now);

    // application time is frozen
    const appNow = new Date();
    expect(appNow.getTime()).toBe(now);
    expect(appNow.getTime()).toBe(1489449600000); // the timestamp in milliseconds

    // we can advance the application clock by 5 seconds
    jest.advanceTimersByTime(5000);
    const appNow2 = new Date();
    expect(appNow2.getTime()).toBe(1489449605000);

    // spec clock only advanced by probably less than 200ms
    const specNow3 = new Date();
    expect(specNow3.getTime()).toBeLessThan(specNow.getTime() + 200);

    jest.useRealTimers();
  });
});
```

## Comparison of network spies & mocks: `cy.intercept` vs `MSW`

If you have been through [Kent C. Dodd's Epic React](https://epicreact.dev/), you are already convinced that the farther away from our component the mocking is, the more we are testing our code and having better confidence. The farthest we can mock from our code is mocking the network. To mock the network Cypress has the [`intercept`](https://docs.cypress.io/api/commands/intercept#docusaurus_skipToContent_fallback) api, and the exact mirror on RTL side is [Mock Service Worker (`msw`)](https://mswjs.io/docs/).

Let us compare [RTL + MSW Heroes.test.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.test.tsx) vs [CyCT + cy.intercept](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.cy.tsx) from Tour of Heroes repo.

Here is a summary of the key points; with `msw` we have a bit more setup, and a required clean up, on the other side with `cy.intercept` the syntax is succinct and no clean up is required.

```tsx
// RTL + MSW
it("should see error on initial load with GET", async () => {
  // have to define handlers, setup server and listen
  const handlers = [
    rest.get(
      `${process.env.REACT_APP_API_URL}/heroes`,
      async (_req, res, ctx) => res(ctx.status(400))
    ),
  ];
  const server = setupServer(...handlers);
  server.listen({
    onUnhandledRequest: "warn",
  });

  // test code...

  // have to clean up
  server.resetHandlers();
  server.close();
});

// CyCT + cy.intercept
it("should see error on initial load with GET", () => {
  // in comparison, this is all we have to do with cyct
  cy.intercept("GET", `${Cypress.env("API_URL")}/heroes`, {
    statusCode: 400,
    delay: 100,
  }).as("notFound");

  // test code...

  // no clean up needed
});
```

The difference is more clear when we want to structure our tests. RTL + `msw` ends up being plenty more boilerplate compared to CyCT and `cy.intercept`. Not only the concise syntax, but also the ability to define or change our network mock on the fly with `cy.intercept` makes our code easier to understand. Check out the code comments below for precise examples:

```tsx
// RLT + msw
describe("200 flows", () => {
  const handlers = [
    rest.get(
      `${process.env.REACT_APP_API_URL}/heroes`,
      async (_req, res, ctx) => res(ctx.status(200), ctx.json(heroes))
    ),
    // we have to have all definitions in the handler
    // once declared, these are it;
    // we would need a new describe block to change the mock
    rest.delete(
      `${process.env.REACT_APP_API_URL}/heroes/${heroes[0].id}`, // use /.*/ for all requests
      async (_req, res, ctx) => res(ctx.status(400), ctx.json("expected error"))
    ),
  ];
  const server = setupServer(...handlers);
  beforeAll(() => {
    server.listen({
      onUnhandledRequest: "warn",
    });
  });
  afterEach(server.resetHandlers);
  afterAll(server.close);

  it("should display the hero list on render, and go through hero add & refresh flow", async () => {
    expect(await screen.findByTestId("list-header")).toBeVisible();
    expect(await screen.findByTestId("hero-list")).toBeVisible();

    await userEvent.click(await screen.findByTestId("add-button"));
    expect(window.location.pathname).toBe("/heroes/add-hero");

    await userEvent.click(await screen.findByTestId("refresh-button"));
    expect(window.location.pathname).toBe("/heroes");
  });

  const deleteButtons = async () => screen.findAllByTestId("delete-button");
  const modalYesNo = async () => screen.findByTestId("modal-yes-no");
  const maybeModalYesNo = () => screen.queryByTestId("modal-yes-no");
  const invokeHeroDelete = async () => {
    userEvent.click((await deleteButtons())[0]);
    expect(await modalYesNo()).toBeVisible();
  };

  it("should go through the modal flow, and cover error on DELETE", async () => {
    expect(screen.queryByTestId("modal-dialog")).not.toBeInTheDocument();

    await invokeHeroDelete();
    await userEvent.click(await screen.findByTestId("button-no"));
    expect(maybeModalYesNo()).not.toBeInTheDocument();

    await invokeHeroDelete();
    await userEvent.click(await screen.findByTestId("button-yes"));

    expect(maybeModalYesNo()).not.toBeInTheDocument();
    expect(await screen.findByTestId("error")).toBeVisible();
    expect(screen.queryByTestId("modal-dialog")).not.toBeInTheDocument();
  });
});
```

```tsx
// CyCT + cy.intercept
context("200 flows", () => {
  beforeEach(() => {
    // the GET is common to both tests
    cy.intercept("GET", `${Cypress.env("API_URL")}/heroes`, {
      fixture: "heroes.json",
    }).as("getHeroes");

    cy.wrappedMount(<Heroes />);
  });

  it("should display the hero list on render, and go through hero add & refresh flow", () => {
    cy.wait("@getHeroes");

    cy.getByCy("list-header").should("be.visible");
    cy.getByCy("hero-list").should("be.visible");

    cy.getByCy("add-button").click();
    cy.location("pathname").should("eq", "/heroes/add-hero");

    cy.getByCy("refresh-button").click();
    cy.location("pathname").should("eq", "/heroes");
  });

  const invokeHeroDelete = () => {
    cy.getByCy("delete-button").first().click();
    cy.getByCy("modal-yes-no").should("be.visible");
  };

  it("should go through the modal flow, and cover error on DELETE", () => {
    cy.getByCy("modal-yes-no").should("not.exist");

    cy.log("do not delete flow");
    invokeHeroDelete();
    cy.getByCy("button-no").click();
    cy.getByCy("modal-yes-no").should("not.exist");

    cy.log("delete flow");
    invokeHeroDelete();

    // DELETE mock is unique to this test
    // we can define it or change our network mock on the fly
    // we could for example have a new GET definition here
    cy.intercept("DELETE", "*", { statusCode: 500 }).as("deleteHero");

    cy.getByCy("button-yes").click();
    cy.wait("@deleteHero");
    cy.getByCy("modal-yes-no").should("not.exist");
    cy.getByCy("error").should("be.visible");
  });
});
```

In our opinion, the intercept API is simpler and is more flexible compared to MSW. We can observe this in the significant difference in the amount of code we have to write, doing the same thing in RTL + `msw` vs CyCT + `cy.intercept`. You can compare them here [RTL + MSW Heroes.test.tsx](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.test.tsx) vs [CyCT + cy.intercept](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.cy.tsx).

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h4wuhxdry0s6nd301nxw.png)

## CyCT vs RTL examples - Epic React

In the interest of brevity, we will share links to [13 Cypress component tests](https://github.com/search?q=repo%3Amuratkeremozcan%2Fcypress-react-component-test-examples+1%3A1+comparison+with+RTL&type=code) and their RTL mirrors. All CyCT examples are from the repo [cypress-react-component-test-examples](https://github.com/muratkeremozcan/cypress-react-component-test-examples) where you can find 400+ individual CyCT examples.

Simple counter: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.0.testing-react-apps/02.simple-rtl-example/counter.cy.jsx#L14) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/epic-react/06.testing-react-apps/src/__tests__/exercise/03.js)

Testing with context: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.0.testing-react-apps/07.testing-with-context/easy-button.cy.js#L4) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/epic-react/06.testing-react-apps/src/__tests__/exercise/07.js)

Simple redux: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.1.testingJs/14.redux/redux.cy.js#L7) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/testing-js/03.rtl/src/__tests__/14.KEY-redux-02.js)

A11y: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.1.testingJs/07.a11y/a11y.cy.js#L39) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/testing-js/03.rtl/src/__tests__/07.a11y.js)

Geolocation: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.0.testing-react-apps/06.mocking-browser-apis/location.cy.js#L15) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/epic-react/06.testing-react-apps/src/__tests__/exercise/06.js)

Mocking http (intercept vs msw) : [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.0.testing-react-apps/05.mocking-http-requests/login-submission.cy.jsx#L13) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/epic-react/06.testing-react-apps/src/__tests__/exercise/05.js#L24), another [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.1.testingJs/08-09.mocking-http-requests/09.stub-newtork.cy.jsx#L27) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/testing-js/03.rtl/src/__tests__/09.http-msw-mock.js)

Router-redirect: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.1.testingJs/11.router-redirect/editor.cy.js#L13) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/testing-js/03.rtl/src/__tests__/12.KEY.tdd-04-router-redirect.js)

React-router: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.1.testingJs/13.react-router/main.cy.jsx#L21) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/testing-js/03.rtl/src/__tests__/13.KEY.react-router-02.js)

Modal: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.1.testingJs/16.modal-portals/modal.cy.js#L14) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/testing-js/03.rtl/src/__tests__/16.KEY-modal-portals.js)

Stub window fetch: [cyct](https://github.com/muratkeremozcan/cypress-react-component-test-examples/blob/3f5ddb6eccdf13705d643eb3eabcfd9962e619ab/cypress/component/hooks/kent.c.dodds-epic-react/06.1.testingJs/08-09.mocking-http-requests/08.stub-window.fetch.cy.jsx#L23) vs [rtl](https://github.com/muratkeremozcan/epic-react-testingJs/blob/main/testing-js/03.rtl/src/__tests__/08.KEY-http-jest-mock.js)

## Wrapping up

We went through, in detail, 3 examples of CyCT vs RTL from the repo & book [CCTDD: Cypress Component Test Driven Design](https://muratkerem.gitbook.io/cctdd/) where you can find a few dozen more examples of CyCT vs RTL.

We compared low level mocking in CyCT with Sinon vs mocking in RTL with Jest, with many examples and a [sample cheat sheet repo](https://github.com/muratkeremozcan/sinon-vs-jest).

We compared network level mocking using `cy.intercept` vs `msw` with repository links.

Finally we shared about a dozen more CyCT vs RTL examples from Kent C. Dodds' Epic React.

Equipped with these resources, you have cheat sheets at your fingertips and an information toolset to begin rolling out Cypress component testing in your organizations.

## Addendum: Gleb Bahmutov's [The Missing Comparison Part video](https://www.youtube.com/watch?v=rFUf7xdtt-I): a comparison of the developer experience

In his video, Gleb covered an important part of comparing RTL and CyCT; the developer experience. _"Jest (with RTL) use the terminal JSDom, while Cypress gives you the real browser with time traveling debugger, making debugging errors so so so much simpler in Cypress"_ We strongly suggest to see the video, and we will cover the three key points below.

### 1. Compare the devex making a simple breaking change to the source code: [HeaderBarBrand](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.tsx#L20)

Pull the [repo](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts), install and start CyCT with RTL side by side; `yarn cy:open-ct` , `yarn test`. Execute the CyCT and RTL tests for `HeaderBarBrand`.

Make a breaking change in [HeaderBarBrand](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/HeaderBarBrand.tsx#L20) component; on line 20 change the spelling of `OF` to `ON`.

```tsx
<NavLink data-cy="navLink" to="/" className="navbar-item navbar-home">
  <span className="tour">TOUR</span>
  {/* Change OF to ON */}
  <span className="of">ON</span>
  <span className="heroes">HEROES</span>
</NavLink>
```

Compare how you would diagnose this failure in RTL and CyCT.

Here is the RTL failure:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1zn3xyyqxqo1s04z341p.png)

Here is the CyCT failure:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q1b0rtxpjud4c1ywxnf8.png)

### 2. Compare test stability by simulating an asynchronous process: [InputDetail](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/InputDetail.tsx#L34)

Execute the CyCT and RTL tests for `InputDetail`.

On line 34 of [InputDetail](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/components/InputDetail.tsx#L34) component, introduce a `setTimeout` to simulate an asynchronous process. These are very common in real applications, and it is a solid comparison of the stability of the two tools, especially in CI.

```tsx
<input
  name={name}
  role={name}
  defaultValue={shownValue}
  placeholder={placeholder}
  // @ts-expect-error add a setTimeout to simulate an asynchronous delay
  onChange={() => setTimeout(onChange, 1000)}
  readOnly={readOnly}
  className="input"
  type="text"
></input>
```

CyCT executes the same, with a slight delay retrying.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ul0a0jj0dcx4r91kdfp2.png)

We cannot make the RTL test pass as is; it is synchronous. We would have to add asynchronous assertions to the test to make it work.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4ti6z6p3y12nqv8jhbts.png)

### 3. Compare the devex making a "complex" change: [Heroes](https://github.com/muratkeremozcan/tour-of-heroes-react-cypress-ts/blob/main/src/heroes/Heroes.tsx)

The test of concern stubs the deleting of a hero with a 500 network response (MSW with RTL, and cy.intercept with CyCT) to simulate a deletion error.

In `Heroes` component, comment out line 40 so that we are doing nothing upon hero deletion.

```tsx
const handleDeleteFromModal = () => {
  // heroToDelete ? deleteHero(heroToDelete) : null
  setShowModal(false);
};
```

We are in the dark, trying to identify why the test did not work looking at RTL results (which are a few times longer than the screenshot):

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r2pn0yavzl3s39w3zagx.png)

Looking at the CyCT failure, we can easily tell that the 500 network call never happened:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/b8dwaq3tnqqo9gfiilan.png)
