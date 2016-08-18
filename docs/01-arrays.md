---
id: arrays
title: What is ponytest
permalink: /docs/arrays.html
prev: syntax.html
next: classes.html
---

`ponytest` is a package wrapping [unittest](https://docs.python.org/3.4/library/unittest.html), fully compatible with it. It's features:

## You can define setup/teardown fixtures with context managers.

  It feels like a better choice then mixins to the testcase class when the set of required fixtures should be evaluated dynamically.

  Usually a context manager describes a **resource** that has to be acquired and cleaned up. Later on, the word "resource" will sometimes be used as a synonym to setup + teardown fixture.

## A test resource can have multiple "providers".

  In that case, it is possible to run tests with all combination of resource providers.

  Imagine a case when a resource is fetched from a web API. The resource providers may be: 1) the actual API is available 2) user is offline 3) special cases that handled with mocks

## You can use command line switches specify the fixtures and their providers.

  With that you can enable/disable a fixture, or chose a provider for a resource, from the command line.

# Motivation

Testing ORM

# Example

Let's see how a test of some customer service may look like. Say a test expects a user with some customer plan, and, possibly, some online payment service selected. In the example below, the service is only available for the premium customer plan.

```python
from ponytest import provider

@provider(fixture='user')
@contextmanager
def get_user(test):
  user = User(name='John')
  user.save()
  test.user = user
  yield

@provider('premium', fixture='user_plan')
@contextmanager
def premium_plan():
  user.set_plan(UserPlan('premium'))
  yield

@provider('basic', fixture='user_plan')
@contextmanager
def basic_plan():
  user.set_plan(UserPlan('premium'))
  yield

@provider('paypal', fixture='payment_service')
@contextmanager
def paypal():
  user.set_payment_service(PayPal())
  yield

@provider('google_wallet', fixture='payment_service')
@contextmanager
def google_wallet():
  user.set_payment_service(GoogleWallet())
  yield


class Test(unittest.TestCase):
  pony_fixtures = {
    'test': ['user', 'user_plan', 'payment_service']
  }

  def test_service_1(self):
    with ExitStack() as stack:
      if self.user.plan.alias == 'basic':
        stack.enter_context(
          self.assertRaises(Exception, 'unavailable for given customer plan')
        )
      self.order_service_1()
```

As you can see, all fixture providers have been passed `fixture` argument, which is a string key. Than, that fixture keys were listed in `pony_fixtures` dictionary. This is how we define it:

```python
  pony_fixtures = {
    'test': [...],  # test-level fixtures
    'class': [...], # class-level fixtures
  }
```

Besides specifying fixtures in the testcase, fixtures can be registered globally. In the example above, it could be the creation of new database for the test class and initializing it.

Let's run the example:

``` command-line
$ python -m ponytest tests.Test
```

We'll have **4 tests** executed, for each of 2 user plans and 2 payment services.

We can run all tests for a given user plan with

``` command-line
$ python -m ponytest tests.Test -- --user-plan basic
```

Also, there is globally registered fixture `ipdb`, that comes with _ponytest_, so you can drop into debugger on failures:

``` command-line
$ python -m ponytest tests.Test -- --ipdb
```