# vue-testing-examples

The documentation is still under construction.

To document:
1. Testing v-model
1. Testing navigation guards
1. Testing filter
1. Testing directive
1. Writing complex integration test for a component
1. Mocking vuex
1. Mocking router
1. Inversify and mocking router/store in tests
1. mock store in the router

Introduce the dev stack:
* vue, vuex, vue-router, vue-property-decorator, vuex-class, axios, lodash, inversify (vanilla js solution), sinon, mocha, sinon-chai, sinon-stub-promise flush-promises, lolex

Provide examples for
* ??? testing mutations, actions ???
* ??? testing `router.push()` to the route with async component using `import()` ???

Issues:
* !!! `trigger("click")` on `<button type="submit">` doesn't trigger submit on form. !!!

# Testing pyramid, dumb vs smart components, mount vs shallow

If you are not familiar with [testing pyramid](https://martinfowler.com/articles/practical-test-pyramid.html), check out the article about it written by [Martin Fowler](https://martinfowler.com/). Usually, only writing unit tests for a VUE app is not enough, especially if you have a component with lots of logic and lots of dependencies. It would be better to test it together. You can try to do that in E2E tests, but E2E tests should be running against fully working and fully configured system with DB, all back-end services and without any mocks. That's very hard to achieve locally, and if you want to test all edge cases, E2E tests can take hours to run. Usually, E2E tests run after your changes were merged to master branch and were fully tested by your QA and were integrated by your CI process. It's not too late to spot the bugs here, but it delays delivery of your product because bug must be fixed first and go through PR, QA and CI again.

Integration tests are a much better solution for this. These tests can integrate a couple of components together and allows you to mock API calls, so you can easily simulate all edge cases which you cannot achieve in E2E tests. Integration tests are also much faster than E2E because they don't fully load the app and don't rely on the network.

You should still keep writing E2E tests, but rather testing every edge case, test only a [happy path](https://en.wikipedia.org/wiki/Happy_path) and common errors (like validation errors), and put all possible scenarios to the integration tests.

The app which docs you are reading now is using following rules to determine whether to use unit or integration tests and what should be tested or mocked out. The main difference is whether you are testing [smart or dumb components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0).
## Dumb components
A dumb component should never be aware of its parent and children, all data to display should be coming from props. The component should not perform any operation in store, should not navigate to different routes and should not perform any API calls. The only test we can do is a unit test, using only shallow rendering.

## Smart components
A smart component is responsible for loading data (from API or store), interacting with the store, passing props down and waiting for events from children, thus the component should be tested using integration test and be aware of its children during the tests. For that, we should always be using `mount` instead of `shallow` rendering. We should mock out only a necessary minimum from the component. Store and router should be used without mocking, so we can test actions dispatched by the component. The only thing to mock is back-end API. If API can return multiple HTTP Codes (200, 302, 400, 401, 403, 404, 412), test all possibilities here, because it's cheap with mocked API. Later, in E2E tests, test only a happy path (200) and validation errors (400).

We should focus on testing as much as possible here because [the less we are mocking we get more confidence in our code](https://twitter.com/kentcdodds/status/977018512689455106).

Use user interactions instead of calling methods in component directly. That also brings your tests closer to the way how your components are used in production.

## Functional components
A functional component is considered dumb component and should follow the same rules.

## Services
Services should be autonomous and should not rely on VUE. A unit tests are just enough. If service is calling back-end API, mock it and test all possible return codes (same as smart components). If service depends on another service, mock them too.

## Filters
A filter should be a simple pure function without any side effects and should not be using the store, router, any service or API calls. Unit testing is considered enough here.

## Directives
A directive usually requires another HTML element and an user interaction to be triggered. Integration tests and `mount` are better for these scenarios.

## Store
A store should have tested all actions and mutations using unit tests. Mock all services and API calls. Smart components should be testing interaction with the store in their tests. If the store uses a router, mock it.

## Router
If there is a logic in your router config (e.g. `beforeRouteEnter`), test this logic in unit tests. If the same logic uses a store, mock it too. Everything else should be mocked as well. Routing between components should be tested in integration tests in smart components or in E2E tests.

|What            |Test type  |shallow/mount|Service calls|API calls|store      |router     |
|----------------|-----------|-------------|-------------|---------|-----------|-----------|
|Dumb components |Unit       |shallow      |mock         |N/A      |N/A        |N/A        |
|Smart components|Integration|mount        |do not mock  |mock     |do not mock|do not mock|
|Func components |Unit       |shallow      |mock         |N/A      |N/A        |N/A        |
|Services        |Unit       |N/A          |mock         |mock     |mock       |mock       |
|Filters         |Unit       |N/A          |N/A          |N/A      |N/A        |N/A        |
|Directives      |Integration|mount        |N/A          |N/A      |N/A        |N/A        |
|Store           |Unit       |N/A          |mock         |mock     |N/A        |mock       |
|Router          |Unit       |N/A          |mock         |mock     |mock       |N/A        |

# Mocking axios

Let's have a simple service which calls external API:
```js
class AuthService {
  login (username, password) {
    return axios.post('/api/login', { username, password }).then(response => response.data);
  }
}
```
For testing different scenarios (200, 400, 404 status codes) we can use [axios-mock-adapter](https://www.npmjs.com/package/axios-mock-adapter), but there are also other alternatives available (e.g. [moxios](https://www.npmjs.com/package/moxios)).
```js
import axios from 'axios';
import AxiosMockAdapter from 'axios-mock-adapter';

beforeEach(function () {
  this.axios = new AxiosMockAdapter(axios);
});

afterEach(function () {
  this.axios.verifyNoOutstandingExpectation();
  this.axios.restore();
});

it('should call external API with given params', function () {
  let fakeData = {};
  let username = 'fake_username';
  let password = 'fake_password';
  this.axios.onPost('/api/login', { username, password }).replyOnce(200, fakeData);
  return this.authService.login(username, password).then(function (response) {
    expect(response).to.deep.equal(fakeData);
  });
});
```
### Dos
* When defining mock, always try to set expected params or body (be as much specific as it gets). It prevents your tests going green because of the functionality under test was accidentally called from other parts of your code.
* Always specify the number of calls you expect. Use `replyOnce` when you know the external call should happen only once (99% test cases). If your code has a bug and calls API twice, you will spot the problem because the 2nd call will fail with `Error: Request failed with status code 404`.
* Always restore axios back to the state before your test was running. Use `axios.restore()` function to do so.
* Verify after each test that all registered mocks have been called using `axios.verifyNoOutstandingExpectation()`. It can catch issues when your test didn't executed part of the code you expected, but still went green. You can implement your own expectation if you need to, this is the current implementation:

```js
import AxiosMockAdapter from 'axios-mock-adapter';
import { expect } from 'chai';

AxiosMockAdapter.prototype.verifyNoOutstandingExpectation = function () {
  for (let verb in this.handlers) {
    let expectations = this.handlers[verb];
    if (expectations.length > 0) {
      for (let expectation of expectations) {
        expect.fail(1, 0, `Expected URL ${expectation[0]} to have been requested but it wasn't.`);
      }
    }
  }
};
```
### Don'ts
* Do not mock original axios functions because you loose huge portion of functionality done by axios (e.g. custom interceptors).

```js
import axios from 'axios';
import sinon from 'sinon';

beforeEach(function () {
  axios.get = sinon.stub(); // <- please don't
});
```
If you really need to check whether the `get` or `post` have been called, use rather `sinon.spy(axios, 'post');` so the original logic is preserved.

# Assert console.error() has not been called

Let's have a test expecting no action to happen (e.g. a disabled button that should not be enabled while an input is empty). The test loads the form and then it checks if the button is disabled. Everything goes green and everybody's happy. But there is a different problem your test didn't cover. The button was not kept disabled because of the application logic, but because of a javascript error and part of the code didn't execute. To catch cases like this ...

### Dos
* Make sure, after each test, that `console.error()` has not been called. You can spy on `error` function and verify expectations in `afterEach()`.

```js
import sinon from 'sinon';
import { expect } from 'chai';

beforeEach(function () {
  if (console.error.restore) { // <- check whether the spy isn't already attached
    console.error.restore(); // restore it if so. this might happen if previous test crashed the test runner (e.g. Syntax Error in your spec file)
  }
  sinon.spy(console, 'error'); // use only spy so the original functionality is still preserved, don't stub the error function
});

afterEach(function () {
  expect(console.error, `console.error() has been called ${console.error.callCount} times.`).not.to.have.been.called;
  console.error.restore(); // <- always restore error function to its initial state.
});
```

# Page objects pattern

If you are new to the page objects, please follow [great overview](https://martinfowler.com/bliki/PageObject.html) written by [Martin Fowler](https://martinfowler.com/). The basic idea is to keep things [DRY](https://code.tutsplus.com/tutorials/3-key-software-principles-you-must-understand--net-25161) and reusable, but there are few more things to mention: refactoring and readability. Let's have a page object like this:
```js
class LoginPage {
  get usernameInput() {
    return findById("username");
  }

  get passwordInput() {
    return findByCss("input.password");
  }

  get submitButton() {
    return findByXPath("/form/div[last()]/button[@primary]"); // <- please don't do this ever
  }

  login(username, password) {
    this.usernameInput.setValue(username);
    this.passwordInput.setValue(password);
    this.submitButton.click();
  }
}
```
The example above has following issues:
* Every time an ID or CSS class change, you have to update the page object.
* If you want to remove ID from the username, you have to check whether is it used in tests or not.
* If you want to remove a CSS class, you have to check whether is it used in tests or not. If it's used in the test, you have to keep class assigned to the element but the class will no longer exist in CSS file. This causes many confusions.
* Every time a new object is added to the form, it breaks the XPath for the submit button. Because of it, using XPath instead of CSS selector is always considered a bad idea.

None of these is giving you the confidence to do minor/major refactoring of your CSS or HTML because you might break the tests. To remove this coupling, you can give a unique identifier for your element through `data-` attributes introduces in [HTML5](https://developer.mozilla.org/en-US/docs/Learn/HTML/Howto/Use_data_attributes) or you can use your own attribute. You can choose a name of the attribute such as `data-test-id`, `data-qa` or simply `tid` if you prefer short names. The page object will look like then:
```js
class LoginPage {
  get usernameInput() {
    return findByCss("[tid='login__username']");
  }

  get passwordInput() {
    return findByCss("[tid='login__password']");
  }

  get submitButton() {
    return findByCss("[tid='login__submit-button']");
  }

  login(username, password) {
    this.usernameInput.setValue(username);
    this.passwordInput.setValue(password);
    this.submitButton.click();
  }
}
```
In the next step you can make the selector little bit shorter by extracting part of the selector to another function:
```js
function findByTId(tid) {
  return findByCss(`[tid*='${tid}']`);
}
```
Adding `*` into the selector allows us to assign more than one identifier to a single element and having better versatility in your page objects, e.g.:
```html
<nav>
  <ul>
    <li tid="home nav-item nav-item-1">Home</li>
    <li tid="about nav-item nav-item-2">About</li>
    <li tid="help nav-item nav-item-3">Help</li>
  </ul>
</nav>
```
Now you can select individual items by unique ID (`home`, `about`, `help`), or by index (`nav-item-{index}`) or you can select all of them (`nav-item`):
```js
class Nav {
  get homeLink() {
    return findByTId("home");
  }

  get aboutLink() {
    return findByTId("about");
  }

  get helpLink() {
    return findByTId("help");
  }

  get allItems() {
    return findAllByTId("nav-item");
  }

  getItemByIndex(index) {
    return findByTId(`nav-item-${index}`);
  }
}
```

# Dependency Injection

Mocking dependencies and imports in tests might be really tedious. Using [inject-loader](https://github.com/plasticine/inject-loader) can help a lot but requires lots of ugly coding. Another option is to involve dependency injection, such as [inversify](https://github.com/inversify/InversifyJS). They are primarily focused on Typescript but they support [vanilla JS](https://github.com/inversify/inversify-vanillajs-helpers) too. In scenarios where you don't have control over instantiating components and services, there is another very handy [toolbox](https://github.com/inversify/inversify-inject-decorators) which gives you a set of decorators usable in Vue components.

## Setting up DI in VUE

We need to create a container instance for holding all registered injections. We need only one instance per application:
```js
import { Container } from 'inversify';
let container = new Container();
export default container;
```
and then just execute it in the bootstrap phase of the application:
```js
import './di';
```
Let's create a service:
```js
export default class AuthService {
  login (username, password) {
    // do something
  }
}
```
... register it in `di.js`:
```js
import { Container } from 'inversify';
import AuthService from 'services/auth';
let container = new Container();
container.bind('authService').to(AuthService);
export default container;
```
... and then use it in VUE component:
```js
import container from './di';

class LoginView extends Vue {
  constructor() {
    this.authService = container.get('authService');
  }
  login (username, password) {
    return this.authService.login(username, password);
  }
}
```
There are a couple of issues with this approach:
1. If we register all our services in `di.js`, then we nullified code splitting because everything is required during the application bootstrap. To solve this issue, let's register service only when is required for the first time:
```js
import container from './di';

export default class AuthService {
  login (username, password) {
    // do something
  }
}

container.bind('authService').to(AuthService);
```
2. We have lots of [magic strings](http://deviq.com/magic-strings/) everywhere (e.g. what we would do if `authService` changes the name? How we prevent naming collisions?). Well, ES6 introduced [symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) that we can use. Don't forget to export the symbol so it can be used later:
```js
import container from './di';

export const AUTH_SERVICE_ID = Symbol('authService');

export default class AuthService {
  login (username, password) {
    // do something
  }
}

container.bind(AUTH_SERVICE_ID).to(AuthService);
```
... and use it in VUE component:
```js
import container from './di';
import { AUTH_SERVICE_ID } from './services/auth';

class LoginView extends Vue {
  constructor() {
    this.authService = container.get(AUTH_SERVICE_ID);
  }
  login (username, password) {
    return this.authService.login(username, password);
  }
}
```

Check out the inversify [docs](https://github.com/inversify/InversifyJS/blob/master/wiki/symbols_as_id.md) for more.

## Using decorators
Decorators can help us to eliminate lots of code repetition and make our code cleaner.

### @Register decorator
The @Register() decorator is just a syntactic sugar for registering classes to the container. In Typescript, you would have `@injectable` and `@inject` decorators available, but this project is not using TS, it's plain JS. The maintainers of InversifyJS have provided another set of decorators/helpers called [inversify-vanillajs-helpers](https://github.com/inversify/inversify-vanillajs-helpers), for using inversify without TS. They just need a unique identifier for each registered class, because they cannot tell what parameter in constructor belongs to which registered class.

Let's have classes defined as:
```js
class C {}

class B {
  constructor(c) {
    this.c = c;
  }
}

class A {
  constructor(b, c) {
    this.b = b;
    this.c = c;
  }
}
```
You can register all classes to the container and give each class unique identifier.
```js
container.bind("c").to(C);
container.bind("b").to(B);
container.bind("a").to(A);

let c = container.get("c");
console.log(c instanceof C); // true
```
Unfortunately, this is not enough for `B` and `A`. When you ask in the code `container.get("A")`, the container will try to instantiate A but there are 2 unknown parameters `b` and `c` in the constructor. We need to tell the container, that for parameter `b`, use class registered with key `"b"` and for `c`, use class with key `"c"`. This can be done this way:
```js
inversify.decorate(inversify.injectable(), A);
inversify.decorate(inversify.inject("b"), B, 0);
inversify.decorate(inversify.inject("c"), C, 1);
container.bind("a").to(A);
```
In TS, this is done by `@injectable` and `@inject` decorators automatically. Doing this for every class in your project would be annoying, so let's use rather a vanillajs helper:
```js
let register = helpers.register(container);
register("a", ["b", "c"])(A);
```
^^ this helper called `register` is now coupled with the container and does exactly the same thing as a previous example. And the helper can also be used as a decorator:
```js
@register("a", ["b", "c"])
class A {
  constructor(b, c) {}
}
```
... is equivalent of calling:
```js
register("a", ["b", "c"])(
class A {
  constructor(b, c) {}
});
```

In this project, the decorators are defined in `di.js`:
```js
import { helpers } from 'inversify-vanillajs-helpers';

let container = new Container();
let register = helpers.register(container);

export { register as Register } // we are exporting decorator with capital R because other decorators we are already using (e.g. for Vuex) also have a capital letter
```

... so they can be accessed anywhere in the code:
```js
import { Register } from './di';
import { CREDENTIALS_SERVICE_ID } from './credentials';

export const AUTH_SERVICE_ID = Symbol('authService');

@Register(AUTH_SERVICE_ID, [ CREDENTIALS_SERVICE_ID ])
export default class AuthService {
  constructor(credentialsService) {
    this.credentialsService = credentialsService;
  }

  login (username, password) {
    username = this.credentialsService.sanitize(username);
    password = this.credentialsService.sanitize(password);
    // do something else
  }
}
```
DI will ensure that the credentialsService will be instantiated first, using `CREDENTIALS_SERVICE_ID` to locate the constructor.
Check out documentation for [inversify-vanillajs-helpers](https://github.com/inversify/inversify-vanillajs-helpers#usage) to see all possibilities

### @LazyInject decorator

The LazyInject decorator can be very useful for injecting services into the VUE components. Because we don't have control over instantiating of the component, we need to inject services into properties. Let's import the decorator from [inversify-inject-decorators](https://github.com/inversify/inversify-inject-decorators) and make it available in `di.js` first:
```js
import getDecorators from 'inversify-inject-decorators';

let container = new Container();
let { lazyInject } = getDecorators(container);

export { lazyInject as LazyInject }
```
Now we can adjust code in VUE component:
```js
import { LazyInject } from './di';
import { AUTH_SERVICE_ID } from './services/auth';

class LoginView extends Vue {
  @LazyInject(AUTH_SERVICE_ID) authService;

  login (username, password) {
    return this.authService.login(username, password);
  }
}
```
`LazyInject` caches the instance of `authService` until the component is destroyed. Check out the documentation for [inversify-inject-decorators](https://github.com/inversify/inversify-inject-decorators#basic-property-lazy-injection-with-lazyinject) to see more options and other decorators.

## Why Dependency Injection?

### 1. Mocking behavior in development

Let's say that you have email service which sends email to given address, but you don't want to send real emails in dev/tst/uat environments. You can have two different implementations of the same interface and register them conditionally:
```js
class RealEmailService {
  sendEmail(from, to, body) {
    // send the email
    return true; // or false if sending failed
  }
}

class FakeEmailService {
  sendEmail(from, to, body) {
    console.log(`Sending email from ${from} to ${to} with body ${body} skipped.`)
    return true; // always return true
  }
}

// config.js
if (process.env.NODE_ENV === "production") {
  container.bind("emailService").to(RealEmailService);
} else {
  container.bind("emailService").to(FakeEmailService);
}

@Register("orderService", ["emailService"])
class OrderService {
  constructor(emailService) {
    this.emailService = emailService; // this would be real implementation in prod, but fake in other environments
  }

  registerOrder(order) {
    // ...
    this.emailService.sendEmail("system", order.emailAddress, `Order ${order.number} has been sent.`);
    // ...
  }
}
```

### 2. Swapping dependencies in the runtime based on a configuration and other flags

DI containers often support named bindings, which allows registering multiple classes to one key. See also the [docs](https://github.com/inversify/InversifyJS/blob/master/wiki/named_bindings.md) about named bindings.

```js
class VIPDiscountService {
  getDiscount() {
    return 10;
  }
}

class RegularDiscountService {
  getDiscount() {
    return 5;
  }
}

container.bind("discountService").to(VIPDiscountService).whenTargetNamed("VIP");
container.bind("discountService").to(RegularDiscountService).whenTargetNamed("Regular");

@Register("orderService")
class OrderService {
  registerOrder(order) {
    // ...
    let customerType = order.customer.isVIP ? "VIP" : "Regular";
    let discountService = container.getNamed("discountService", customerType);
    let discount = discountService.getDiscount();
    // ...
  }
}
```

### 3. Mocking in unit tests
Replacing registered classes allows you to control dependencies of the service under test. See next chapter [Testing using Dependency Injection](#testing-using-dependency-injection)

## Testing using Dependency Injection

### Mocking service dependencies

Let's have a service which depends on other services. Before we even start writing tests, make sure that the dependency is really needed. Huge dependency trees are always hard to test and to maintain because of the complexity. You can try to decouple the services and separate their logic. If it's not possible, then you should always mock other services out. Consider following services:
```js
class ServiceA {
  constructor(serviceB, serviceC) {}
}

class ServiceB {
  constructor(serviceD) {}
}

class ServiceC {
  constructor(serviceE) {}
}

class ServiceD {}

class ServiceE {
  constructor(serviceD) {}
}
```
How can we test ServiceA with all these dependencies? Let's do a naive approach first:
```js
describe('Service A', function () {
  beforeEach(function () {
    let serviceD = new ServiceD();
    let serviceE = new ServiceE(serviceD);
    let serviceB = new ServiceB(serviceD);
    let serviceC = new ServiceC(serviceE);
    let serviceA = new ServiceA(serviceB, serviceC);
  });

  it('should do something', function () {
    // test logic using serviceA
  });
});
```
A couple of issues here:
1. You have to be aware of service instantiation order, `serviceD` must be created before `serviceE` and `serviceB`, `serviceE` before `serviceC` and once you have all dependencies ready, you can finally call the constructor of `ServiceA`. If dependency tree changes, you have to adjust all tests.
1. Every time new service is added to the tree, you have to update the tests too.

When you want to mock any service in the tree, consider using a fake object with stub methods instead of calling a real constructor. If you are using [sinon](https://www.npmjs.com/package/sinon), check out their [mock API](http://sinonjs.org/releases/v4.5.0/mocks/) which enables lots of nice features and gives you better control over your mocked services.
```js
describe('Service A', function () {
  beforeEach(function () {
    let serviceD = {}; // instead of using real ServiceD constructor, create an empty object and fake all methods that are used from serviceA
    serviceD.doSomething = sinon.spy();
    this.serviceDMock = serviceD;

    let serviceE = new ServiceE(serviceD);
    let serviceB = new ServiceB(serviceD);
    let serviceC = new ServiceC(serviceE);
    let serviceA = new ServiceA(serviceB, serviceC);

    this.serviceBMock = sinon.mock(serviceB); // mock serviceB using sinon.mock API. this will create Proxy object which you can configure to expect calls on serviceB instance.
  });

  afterEach(function () {
    this.serviceBMock.verify(); // verify all expectations have been called
  });

  it('should do something', function () {
    let doSomethingSpy = this.serviceBMock.expects('doSomething').returns(42).once();
    let result = serviceA.runMethodUnderTest(24); // let's assume that this method calls serviceB and serviceD
    expect(result).to.equal(66);
    expect(doSomethingSpy).to.have.been.calledWith(24);
    expect(this.serviceDMock.doSomething).to.have.been.calledWith(24);
  });
});
```

Now, let's do better approach and let DI container to resolve our dependencies.
```js
import { Register } from './di';

@Register('serviceA', ['serviceB', 'serviceC'])
class ServiceA {
  constructor(serviceB, serviceC) {}
}

@Register('serviceB', ['serviceD'])
class ServiceB {
  constructor(serviceD) {}
}

@Register('serviceC', ['serviceE'])
class ServiceC {
  constructor(serviceE) {}
}

@Register('serviceD')
class ServiceD {}

@Register('serviceE', ['serviceD'])
class ServiceE {
  constructor(serviceD) {}
}
```
... and then in spec file you can just type
```js
import container from './di';

describe('Service A', function () {
  beforeEach(function () {
    let serviceA = container.get('serviceA');
  });
});
```

It's much nicer now, but wait, the mocking ability is now gone. How can `serviceC` can be mocked during execution of these tests? The service registration in DI container can be replaced with our own mock.
```js
import container from './di';

describe('Service A', function () {
  beforeEach(function () {
    let serviceC = container.get('serviceC');
    this.serviceCMock = sinon.mock(serviceC);
    container.rebind('serviceC').toConstantValue(serviceC); // removes old registered class and replaces it with singleton constant value
    let serviceA = container.get('serviceA');
  });

  afterEach(function () {
    this.serviceCMock.verify(); // verify all expectations have been called
  });

  it('should do something', function () {
    let doSomethingSpy = this.serviceCMock.expects('doSomething').returns(42).once();
    let result = serviceA.runMethodUnderTest(24); // let's assume that this method calls serviceC
    expect(result).to.equal(66);
    expect(doSomethingSpy).to.have.been.calledWith(24);
  });
});
```
Everything that test changes in the container registration should be restored after the test is done, otherwise, changes might affect the following test. You can achieve this by using built-in functionality in inversify called [snapshots](https://github.com/inversify/InversifyJS/blob/master/wiki/container_snapshots.md). Create a snapshot before you perform changes to registration and restore it back to normal after your test is done.

```js
import container from './di';

describe('Service A', function () {
  beforeEach(function () {
    container.snapshot();
    // ...
    container.rebind('serviceC').toConstantValue(serviceC);
    //
  });

  afterEach(function () {
    container.restore();
  });
});
```
You can move this calls to the global scope of the test to avoid repetition in your code.
```js
// globals.js
import container from './di';

beforeEach(function () {
  container.snapshot();
});

afterEach(function () {
  container.restore();
});
```
... and then use this file when running mocha
```json
// package.json
{
  "test": "vue-cli-service test src/**/*.spec.js --include ./test/unit/globals.js",
}
```
If you are using `--watch` mode, files included are not executed after any change, so the better solution is to import this file at the top of your spec file instead.

### Mocking global objects

Testing a functionality which requires global objects (`window.location` or `setTimeout`) is always a challenge. `window.location` is property which cannot be overridden but many tests require this property to be mocked. If you have a component which sets `window.location.href = 'some/new/url';`, then this code must be avoided in the tests, otherwise, it might break your current test execution (especially if you are running tests in the browser, e.g. with [karma-runner](https://www.npmjs.com/package/karma)). Dependency Injection can actually sort out this problem because we can inject the global object into your component and then we can pass custom object, with no restrictions, into a component in your tests.
```js
// di.js
export const WINDOW_ID = Symbol('window');
container.bind(WINDOW_ID).toConstantValue(window);

// my-components.js
import Vue from 'vue';
import { LazyInject, WINDOW_ID } from './di';

export default class MyComponent extends Vue {
  @LazyInject(WINDOW_ID) window; // uses real window in PROD build, but can be mocked object in tests

  onClick() {
    this.window.location.href = 'some/new/url';
  }
}
```
... and your tests:
```js
// my-component.spec.js
import container { WINDOW_ID } from './di';

beforeEach(function () {
  container.snapshot();

  this.windowMock = { location: { href: '' } };

  container.rebind(WINDOW_ID).toConstantValue(this.windowMock);
});

afterEach(function () {
  container.restore();
});

it('should navigate to new url', function () {
  let myComponent = mount(MyComponent); // LazyInject in the component will inject windowMock instead of real window object
  myComponent.onClick(); // this call now operates with windowMock, so no real navigation happens in the browser

  expect(this.windowMock.location.href).to.equal('some/new/url'); // assert correct URL has been set
});
```

# Time travelling and testing setTimeout
There are several ways how to test `setTimeout` functions in your code. For all solutions, we are going to use LoginView component which displays some helpful links after 5s. Please don't judge the UX, it's for demonstration purposes:
```js
class LoginView extends Vue {
  displayHelp = false;

  created () {
    setTimeout(() => {
      this.displayHelp = true;
    }, 5000);
  }
}
```
What we need to test:
1. The initial status of the component (displaying help links should be off)
2. Time travel to 2s after the component was created (check help links are still not being displayed)
3. Time travel beyond 5s after the component was created (check help links are finally being displayed)

Let's start with easiest, but not very robust test solution:
```js
import { expect } from 'chai';
import sinon from 'sinon';

beforeEach(function () {
  this.originalSetTimeout = window.setTimeout;
  window.setTimeout = sinon.stub();
});

afterEach(function () {
  window.setTimeout = this.originalSetTimeout; // restore original setTimeout
});

it('should not display help links when a component was just created', function () {
  let wrapper = mount(LoginView);
  expect(wrapper.vm.displayHelp).to.be.false;
});

it('should display help links only after 5s', function () {
  let wrapper = mount(LoginView);
  expect(window.setTimeout).to.have.been.calledWith(sinon.match.any, 5000);
});
```
Simple, huh? Yes, it technically does test what was specified, but it's not very convincing. We did lots of shortcuts and we actually never checked whether displayHelp became true or not. The callback in `setTimeout` is part of the component's logic but it wasn't called at all. What if there is a logic somewhere in the code which cancels the timeout? We also cannot check what is the state after 2s. Let's add a little more logic to our test:
```js
it('should display help links only after 5s', function () {
  window.setTimeout = sinon.stub().callsFake(fn => fn()); // any function that is passed to setTimeout is immediately executed

  let wrapper = mount(LoginView);
  expect(window.setTimeout).to.have.been.calledWith(sinon.match.any, 5000);
  expect(wrapper.vm.displayHelp).to.be.true;
});
```
This time callback is executed and our flag is set to true, but we created another problem. The callback is executed synchronously instead. That changes the order of execution which means we are not testing how code runs in production. And we still cannot test a state of our component after 2s and we are still unsure about timeout cancellation. Let's involve library which was built-in for this and has better control over `setTimeout` and it's execution: [lolex](https://github.com/sinonjs/lolex). Lolex has an API which can mock setTimeout and allows us to jump in time by 100ms, by 1s, by 1 day while our test is still executing a synchronous way (that means the test execution is not paused for 2s).
```js
import { expect } from 'chai';
import lolex from 'lolex';

beforeEach(function () {
  this.clock = lolex.install(window); // this mocks window.setTimeout with lolex implementation
});

afterEach(function () {
  this.clock.uninstall(); // restore original setTimeout
});

it('should not display help links when a component was just created', function () {
  let wrapper = mount(LoginView);
  expect(wrapper.vm.displayHelp).to.be.false;
});

it('should not display help links before 5s elapsed', function () {
  let wrapper = mount(LoginView);

  this.clock.tick(2000);
  expect(wrapper.vm.displayHelp).to.be.false;

  this.clock.tick(2000);
  expect(wrapper.vm.displayHelp).to.be.false;

  this.clock.tick(2000); // this is the moment when callback is finally executing
  expect(wrapper.vm.displayHelp).to.be.true;
});
```
It's looking really nice now. The clock was able to move our test 2s ahead, check the state, move another 2s, check, move another 2s and do a final check. The callback was executed after multiple calls to `this.clock.tick()` and not synchronously during `created()` lifecycle event. Lolex also supports mocking of other global functions manipulating time, see their [API reference](https://github.com/sinonjs/lolex#api-reference).
Lolex is also capable to work with mocked global objects. Let's consider a scenario from [Mocking global objects](#mocking-global-objects):
```js
import { expect } from 'chai';
import lolex from 'lolex';

import container { WINDOW_ID } from './di';

beforeEach(function () {
  container.snapshot();

  let windowMock = {};
  this.clock = lolex.install({ target: windowMock, toFake: ['setTimeout', 'clearTimeout'] }); // only fake timeout methods, don't waste time on other functions
  container.rebind(WINDOW_ID).toConstantValue(windowMock);
});

afterEach(function () {
  container.restore();
  this.clock.uninstall();
});

// ...
```

# Using stub services in dev mode

Local development usually requires working back-end services (API) to be running on developers machine. While the API is still in development, usually it's very unstable with lots of bugs (we all producing bugs). Front-end developers often using a technique of stubbing API and work with fake implementation instead. Just create a simple express server, develop your application against it and once the back-end API is ready, just unplug the stubs. Stubs also help to simulate scenarios which are really hard to achieve using real back-end and DB.

Instead of building your own express server, webpack offers another nice solution how to create stubs for your services. Let's have an `AuthService` which communicates with real back-end API:
```js
import axios from 'axios';
import { Register } from './di';

export const AUTH_SERVICE_ID = Symbol('authService');

@Register(AUTH_SERVICE_ID)
export default class AuthService {
  login (username, password) {
    return axios.post('/api/login', { username, password }).then(response => response.data);
  }

  logout () {
    return axios.post('/api/logout').then(response => response.data);
  }
}
```
And let's pretend that the back-end is not ready yet. We can create twin service with fake implementation:
```js
export default class AuthServiceStub {
  login (username, password) {
    return Promise.resolve("random_token");
  }

  logout () {
    return Promise.resolve({});
  }
}
```
Now we need to solve a problem how and when to switch these two services. How can we tell the app which one to use? Webpack has a nice feature called [resolve.extensions](https://webpack.js.org/configuration/resolve/#resolve-extensions) which is used as a decision point to determine extension of the file. In vue app, the value is set to `[".js", ".vue", ".json"]`. If you add `import AuthService from './auth-service'`, then webpack tries to find a file `./auth-service.js`, then `./auth-service.vue` and then `./auth-service.json`, the first match wins. So if we move our AuthServiceStub to another file `./auth-service.stub.js` and then tell the webpack to resolve extensions in order `[".stub.js", ".js", ".vue", ".json"]`, it will first try to import our stub service and only if a stub is not present, then it imports the real implementation.

This logic can be controlled via [configuration](https://www.npmjs.com/package/webpack-chain), in `vue.config.js`.
```js
let useServiceStub = !!process.env.npm_config_stub;

module.exports = {
  chainWebpack (config) {
    if (useServiceStub) {
      config.resolve.extensions.prepend('.stub.js');
    }
  }
};
```
`npm_config_*` is a feature of `npm run-script`. If you call any npm script with `--stub` flag, `process.env.npm_config_stub` will be set to true. In our vue app, there is already command for serving the app in `package.json`:
```bash
npm run serve
```
^^ this one sets `npm_config_stub` to false, so the app will run without stubs
```bash
npm run serve --stub
```
^^ and this one with stubs.

### Dos
* You can reuse logic from real implementation in your stub by inheriting the stub from a real one. In this example, you can see the stub not completely overriding existing code, but rather mocking API call based on the username and password. The advantage is that logic inside real implementation is now running too, which might be important for you. Also, mocking axios is better than returning resolved promises because you might have registered an error interceptor (or any other request interceptor) which you would like to see running too.
```js
import axios from 'axios';
import AxiosMockAdapter from 'axios-mock-adapter';

import { Override } from './di';
import AuthService, { AUTH_SERVICE_ID } from './auth.js'; // always use '.js' extension otherwise this module will be required.

export { AUTH_SERVICE_ID };

@Override(AUTH_SERVICE_ID)
export default class AuthServiceStub extends AuthService {
  login (username, password) {
    let axiosMock = new AxiosMockAdapter(axios);
    if (username === password) {
      axiosMock.onPost('/api/login', { username, password }).replyOnce(200, 'random_token');
    } else {
      axiosMock.onPost('/api/login', { username, password }).replyOnce(401, { error: { message: 'Invalid username or password' } });
    }

    return super.login(username, password).finally(() => axiosMock.restore()); // call original function and restore axios back to original state
  }
}
```

### Don'ts
* If you are inheriting classes, don't forget to always call `super()` methods.

# Using flush-promises vs Vue.nextTick()

This topic has been fully covered by [the official documentation](https://vue-test-utils.vuejs.org/en/guides/testing-async-components.html).
