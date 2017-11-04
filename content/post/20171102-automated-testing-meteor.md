---
title: "Automated testing with Meteor"
author: Matt Dorn
date: 2017-11-02
tags:
  - meteor
  - testing
thumbnailImagePosition: top
thumbnailImage: https://www.nasa.gov/sites/default/files/styles/ubernode_alt_horiz/public/iss028e024847_0.jpg
coverImage: https://www.nasa.gov/sites/default/files/styles/ubernode_alt_horiz/public/iss028e024847_0.jpg
metaAlignment: center
---

In this post I examine the current state of the art in testing Meteor apps and describe the approach we've taken with IFDB, including some of the troubles experienced and successes enjoyed along the way.

<!--more-->

<!-- TOC -->

[Meteor](https://www.meteor.com/)'s place in the JavaScript ecosystem has long been a subject of controversy.  Through version 1.2, this opinionated framework was famous for enabling even inexperienced developers to develop real-time single-page web apps with relatively little effort and few lines of code.  This simplicity came at a cost.  For extending app functionality, developers were restricted to using (via [Atmosphere](atmospherejs.com/)) third-party libraries that were specifically packaged for Meteor.  On the front end, initially you were required to use a Meteor-specific library, [Blaze](http://blazejs.org/), even if you preferred mainstream alternatives like [Angular](https://angularjs.org/) or [React](https://reactjs.org/).  Finally, automated testing was an afterthought, if not practically impossible.

The release of Meteor 1.3, however, made possible the use of ES2015 modules, and opened up Meteor developers to the entire universe of packages on [npm](https://www.npmjs.com/).  This in turn made it possible to incorporate best testing practices and libraries from the JS ecosystem.  While Meteor is still not without its idiosyncrasies and challenges, these and other recent developments have made Meteor -- now at version 1.6 -- a very serious contender in the field of web app platforms (even if it's sacrificed much of the simplicity that made it famous pre-1.3) particularly if your app needs real-time functionality.

## Overview

Testing a Meteor app is not a simple affair, however.  Even if you restrict yourself to the [documentation](https://guide.meteor.com/testing.html) in the [Meteor Guide](https://guide.meteor.com/), you'll need to become familiar with a hodgepodge of testing libraries that serve various purposes, and if your app is even moderately sophisticated you're likely to run into roadblocks that are not accounted for in the guide.

Here's a partial list of the NPM libraries that IFDB's testing setup relies on:

- [Mocha](https://mochajs.org/): core test framework
- [Chai](http://chaijs.com/): assertions
- [Enzyme](http://airbnb.io/enzyme/): React component testing
- [Chimp](chimp.readme.io/): Browser testing
- [Sinon](sinonjs.org): Mocks and stubs
- [rewire](https://github.com/jhnns/rewire): For mocking modules, which Sinon won't do

And the Meteor-specific packages:

- `meteortesting:mocha`
- `dburles:factory`
- `johanbrook:publication-collector`
- `hwillson:stub-collections`
- `tmeasday:acceptance-test-driver`
- `xolvio:cleaner`
- `xolvio:backdoor`

Most of this is in line with the recommendations made in the Meteor docs.

## Writing tests

I recommend taking a look at the [source](https://github.com/meteor/todos/tree/react) for the official Meteor "todos" app for some guidance and inspiration, but below I distill some of what you can learn there, and also address some pitfalls.

### Unit tests

In Meteor, test files are named `*.test[s].*` and can be placed anywhere in your app's directory structure.  They'll only be bundled when your app runs in test mode.  We chose to put server-side unit tests in a `_test` directory for each "domain" in the API (e.g. `/imports/api/products/_test` for the "products" domain in an online store app), and, for client-side unit tests (see below) for each logical grouping in the `/imports/ui/components` directory.

#### Server: methods and publications

On the server side, we focus on testing our Meteor methods and publications.  So a simplified test file might look like this:

```javascript
/* eslint-env mocha */
/* eslint-disable func-names, prefer-arrow-callback */

import { Meteor } from 'meteor/meteor';
import { Factory } from 'meteor/dburles:factory';
import { PublicationCollector } from 'meteor/johanbrook:publication-collector';
import { resetDatabase } from 'meteor/xolvio:cleaner';

import { Random } from 'meteor/random';
import { assert } from 'chai';

import { Todos } from '../todos';
import { insertTodo } from '../methods';

if (Meteor.isServer) {
  import '../server/publications.js';
  describe('todos', () => {
    // PUBLICATIONS
    describe('publications', () => {
      before(() => {
        resetDatabase();
        Factory.create('todo' { name: 'Todo 1' });
        Factory.create('todo' { name: 'Todo 2' });
      });
      describe('todos.list', () => {
        it('publishes todos', (done) => {
          const collector = new PublicationCollector({ userId: Random.id() });
          collector.collect(
            'todos.list',
            collections => {
              assert.equal(collections.alerts.length, 2);
              done();
            }
          );
        });
      });
    });
    // METHODS
    describe('methods', () => {
      before(() => {
        resetDatabase();
      });
      describe('insert todo', () => {
        it('creates a todo', (done) => {
          const context = { userId: note.userId };
          const args = { name: 'My todo item' };
          const result = insertTodo._execute(context, args);
          const checkTodo = Todos.findOne();
          assert.equal(checkTodo.name, 'My todo item');
        }
      }); 
  });
}
```

Note the use of a few utilities that facilitate testing, including `PublicationCollector` and `resetDatabase`.

> **NOTE**: Not shown here, but you should use [Stub Collections](https://atmospherejs.com/hwillson/stub-collections) whenever possible to make your tests run faster and to isolate the functionality being tested by your unit tests.  The tests above depend on an instance of Mongo, introducing an undesired dependency and also some unpredictability in how fast your tests will run. **However**, Stub Collections can not be used for publications that use Mongo aggregations (e.g. via [`jcbernack:reactive-aggregate`](https://github.com/JcBernack/meteor-reactive-aggregate)).  You may need to introduce a `this.timeout([ms])` line to your tests if you find they're not completing within the expected default timeout.

#### Client: React components

On the client side, we use Enzyme to test our React components in isolation from other components as well as the components that provide data to them.  A super simple example:

```javascript
/* eslint-env mocha */

import { Factory } from 'meteor/dburles:factory';

import React from 'react';
import Enzyme, { shallow } from 'enzyme';
import Adapter from 'enzyme-adapter-react-15';
import { assert } from 'chai';

import '../path/to/fixtures/definitions';
import Alerts from '../Alerts';

Enzyme.configure({ adapter: new Adapter() });

describe('Alert', () => {
  it('renders', () => {
    const alerts = [
      Factory.build('alert', { title: 'testing 1' }),
      Factory.build('alert', { title: 'testing 2' }),
    ];
    const item = shallow(<Alerts alerts={alerts} />);
    assert(item.hasClass('alerts'));
    assert.equal(item.children().length, 2);
  });
});
```

Note here the `Factory.build` method which simply recreates the data structure for the item(s) expected by the React component, rather than inserting the item into Mongo.  You'll need to take some steps to ensure that a browser executable is available to render your components for testing.  More on that below.

### Integration testing

Integration tests are useful for ensuring that two or more distinct components work well together.  For the purposes of this article, we'll skip them.

### End-to-end/acceptance/browser testing

Writing and running these tests differs from what we've seen so far in that they don't rely on anything Meteor-specific (with one exception which you'll see below).  Tests are still written using Mocha, but you'll make use of [Chimp](https://chimp.readme.io/) to drive a browser instance to perform user actions (clicks, form input and the like) and test that the results are as expected.

You'll instruct Chimp on where to find these tests when you run it, so you don't need to follow Meteor test naming conventions.  It's best to keep these tests outside of your app directory.  Here's an example of a simple test located at `tests/features/login.js` in a Meteor project directory:

```javascript
/* eslint-env mocha */
// chimp globals
/* globals browser assert server */

const PORT = 3002;

describe('login', () => {
  before(done => {
    server.call('generateFixtures');
    done();
  });

  beforeEach(() => {
    browser.url(`http://localhost:${PORT}`);
  });

  it('user with right credentials can login @watch', () => {
    browser.waitForExist('#app');
    const emailInput = browser.elements('#login-email');
    const pwdInput = browser.elements('#login-password');
    emailInput.setValue('me@example.com');
    pwdInput.setValue('p@ssw0rd');
    browser.click('#btn-login');
    browser.waitForExist('#dashboard');
    const dashTitle = 'Dashboard';
    assert.equal(browser.getTitle(), dashTitle);
  });
});
```

A couple of things to note here:

- Depending on the complexity of your app, the performance characteristics of your test environment, etc. your components may or may not render in the time expected by `waitForExist`.  You can adjust the timeout on this call if needed, e.g.     `browser.waitForExist('.dashboard', 1000)`.
- Note the line that says `server.call('generateFixtures');` in the code that runs before any tests are executed.  `server` is a Meteor-specific object that allows you to call Meteor methods to via DDP.  In this case we have a `generateFixtures` method that is [only available](https://github.com/tmeasday/acceptance-test-driver) when the app is run in test mode.  When it's executed, the Meteor test database is cleared, and some test fixture data is loaded into the database.

> **WARNING**: It's important to make sure the `generateFixtures` method is only available in test mode, in part because this kind of function normally clears the active database.  Take a look at the [`tmeasday:acceptance-test-driver` package](https://github.com/tmeasday/acceptance-test-driver).

## Running tests

### Unit tests

By default, using the testing approach recommended in the docs results in a nicely formatted test report in your browser.  It's pretty, but of limited utility when your end goal is to have your tests running in your [CI environment](https://circleci.com).  To that end, use [`meteortesting:mocha`](https://github.com/meteortesting/meteor-mocha) rather than the often recommended `practicalmeteor:mocha` package.

    meteor add meteortesting:mocha
  
You may run into conflicts, complete with confusing error messages, if you've been using other Mocha-related packages, so either start your install from scratch with a properly pared down `.meteor/packages` file, or remove the offending packages; for example:

    meteor remove dispatch:mocha
    meteor remove dispatch:mocha-phantomjs
    meteor remove practicalmeteor:mocha

Our tests make use of a database removal function provided by a `cleaner` package:

    meteor add xolvio:cleaner

Once you've installed `meteortesting:mocha` you can run your tests.  In Meteor, test files are named `*.test[s].*` and can be placed anywhere in your app's directory structure.  We chose to put them in a `_test` directory for each "domain" in the API (e.g. `/imports/api/products/_test` for the "product" domain in an online store app), and, for client-side tests (see below) for each logical grouping in the `/imports/ui/components` directory.  When you run:

    meteor test --once --driver-package meteortesting:mocha --port 3002

you'll see the results in your console; failures will be reported to stdout with an appropriate exit code for detection by CI.

To run the client tests, you'll need to make sure you've enabled a headless browser driver:

    meteor npm i -E --save-dev selenium-webdriver@3.0.0-beta-2
    meteor npm i --save-dev chromedriver

Now you can run both your client and server tests in one go:

    TEST_BROWSER_DRIVER=chrome meteor test --once --driver-package meteortesting:mocha --port 3002
  
See the [`meteortesting:mocha` README](https://github.com/meteortesting/meteor-mocha) for more options.

> **NOTE**: Normally you'll want to add these lengthy commands as `scripts` in your `packages.json` file so you can run them as `npm run test` or similar.  E.g.:

```
  {
    "name": "my-website",
    "private": true,
    "scripts": {
    "dev": "meteor run --settings settings-dev.json",
    "test": "TEST_BROWSER_DRIVER=chrome meteor test --once --driver-package meteortesting:mocha  --port 3002",
    ...
```

### Acceptance/browser tests

Install the `chimp` executable globally:

    npm install -g chimp

To enable Chimp to interact properly with Meteor and make the `server` global available in Chimp tests (see above) add the following additional dependency to your Meteor project:

    meteor add xolvio:backdoor

After that, once you've written some tests and save them to `tests/features` (in this example, see above), you can run an instance of your app that loads test fixtures:

    meteor test --full-app --driver-package tmeasday:acceptance-test-driver --settings settings-dev.json --port 3002

And once it's up and running run your Chimp tests like so:

    chimp --ddp=http://localhost:3002 --mocha --path=tests/features/

There are some challenges involved with enabling this type of testing in a CI environment, which we'll cover in a later post.

### Watch mode

In all cases you can run your tests in "watch" mode so that your tests are run immediately when a change is made.

For your unit tests this looks like:

    TEST_WATCH=1 TEST_BROWSER_DRIVER=chrome meteor test --driver-package meteortesting:mocha  --port 3002 --raw-logs
  
The `--raw-logs` flags helps remove the noisy timestamp line from the Meteor development server output.

And for Chimp:

    chimp --ddp=http://localhost:3002 --mocha --path=tests/features/ --watch

Note you'll also need the `@watch` flag in the "it" description of any test you might be working on for this to work.

## Continuous integration

We'll cover CI with [CircleCI](https://circleci.com) in a separate post, but [this article](https://medium.com/@kimeshan/continuous-integration-with-meteor-chimp-galaxy-and-circleci-5ee809ca6116) offers a nice starting point.

## Future considerations

As you can see this approach to Meteor testing (based, as we've noted, on the official [documentation](https://guide.meteor.com/testing.html)) has a lot of dependencies and a lot of moving parts, which translates into potential failure points.  It also means your tests run slower than what you might like.

In a [post](http://www.east5th.co/blog/2016/05/02/meteor-unit-testing-with-testdoublejs/) on using [Testdouble.js](https://github.com/testdouble/testdouble.js/) for Meteor tests, Pete Corey describes a technique for stubbing our various Meteor dependencies more completely for faster, more reliable unit tests.

More recently, a [post](https://blog.meteor.com/real-world-unit-tests-with-meteor-and-jest-3d557e84e84a) appeared on the official Meteor blog for using Facebook's [Jest](https://facebook.github.io/jest/) testing framework with Meteor.  Jest is a full-featured framework that comes with its own components for mocking, assertions, React, etc., and could presumably remove a lot of the complexity of the approach described here.

Both of these merit further investigation for future iterations of automated testing for IFDB.  Maybe by the time we get there, official Meteor recommendations will have caught up with and validated these approaches.
