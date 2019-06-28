---
layout: post
title:  "pytest fixture"
date:   2019-06-29 10:00:00 +0800
categories: pytest, fixture
---

In test automation, one common requirement is to do some setup/cleanup work before/after a test case, a script or a session is executed. One pytest session may run multiple scripts. One script may have multiple test cases. In pytest, all these requirement can be met be using fixture.

Fixture is functions marked with pytest decorator `pytest.fixture`. To use a fixture, simply add the fixture function name in argument of a test case function/method, or in arguments of another fixture. Pytest knows which argument is a fixture and need to be injected.

When define a fixture, we can specify its scope. Available scopes:
* function: The fixture is called before/after each function or method (a test case)
* module: The fixture is called before/after each script
* session: The fixture is called before/after each session

## Pass data from test case to fixture

Fixture can pass parameters to test cases using `return` or `yield`. What if we need to pass parameters from test case to fixture? The factory pattern can be used.

test_something.py
``` python
from __future__ import print_function
import pytest


@pytest.fixture
def setup_teardown():

    print("Start setup")

    def setup(data_in_case):
        print("Setup using data from case: " + str(data_in_case))

    print("Done setup")

    yield setup

    print("Start teardown")
    print("Teardown")
    print("Done teardown")


def test_case1(setup_teardown):
    print("Start test case")

    print("Prepare data")
    var1 = "some data"

    print("Pass the data to fixture and do setup")
    setup_teardown(var1)

    print("Run test steps")

    print("Done test case")
```

Run the test:
```
# pytest test_something.py -s -vvvv
========================================== test session starts ==========================================
platform linux2 -- Python 2.7.12, pytest-4.5.0, py-1.8.0, pluggy-0.12.0 -- /usr/bin/python
cachedir: .pytest_cache
ansible: 2.0.0.2
rootdir: /root/tmp
plugins: ansible-2.0.2
collected 1 item

test_something.py::test_case1 Start setup
Done setup
Start test case
Prepare data
Pass the data to fixture and do setup
Setup using data from case: some data
Run test steps
Done test case
PASSEDStart teardown
teardown
Done teardown


======================================= 1 passed in 0.01 seconds ========================================
```

## Use passed in data in teardown

The passed in data may need to be used in the teardown section. There is a trick to do that:

test_something.py
``` python
from __future__ import print_function
import pytest


@pytest.fixture
def setup_teardown():

    print("Start setup")
    datastore = []

    def setup(data_in_case):
        print("Setup using data from case: " + str(data_in_case))
        datastore.append(data_in_case)
        print("Stored %s for later use" % str(data_in_case))

    print("Done setup")

    yield setup

    print("Start teardown")
    data_in_case = datastore[0]
    print("Got the passed in data in teardown: " + str(data_in_case))
    print("Teardown")
    print("Done teardown")


def test_case1(setup_teardown):
    print("Start test case")

    print("Prepare data")
    var1 = "some data"

    print("Pass the data to fixture and do setup")
    setup_teardown(var1)

    print("Run test steps")

    print("Done test case")
```

Run the test:
```
# pytest test_something.py -s -vvvv
========================================== test session starts ==========================================
platform linux2 -- Python 2.7.12, pytest-4.5.0, py-1.8.0, pluggy-0.12.0 -- /usr/bin/python
cachedir: .pytest_cache
ansible: 2.0.0.2
rootdir: /root/tmp
plugins: ansible-2.0.2
collected 1 item

test_something.py::test_case1 Start setup
Done setup
Start test case
Prepare data
Pass the data to fixture and do setup
Setup using data from case: some data
Stored some data for later use
Run test steps
Done test case
PASSEDStart teardown
Got the passed in data in teardown:some data
Teardown
Done teardown


======================================= 1 passed in 0.01 seconds ========================================
```
