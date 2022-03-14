---
layout: post
title: Test your network automation with Pytest & Mock
date: 2022-02-18 09:46 +0100
---
Recently, we have been challenged on how we could test our [ansible-cvp](https://github.com/aristanetworks/ansible-cvp) development to make sure changes are not breaking the code.

One of the solution would be to build a test topology we can access from our CI and use it to run ansible content to validate. Regardless the security aspect of such topology, we thought it is not efficient: if your code is broken, you will eat an ansible error message and not a pointer to the python code.

So we decided to take the well known software approach: [pytest to run unit & system tests](https://docs.pytest.org/en/7.0.x/). And to use it with no [Cloudvision](https://www.arista.com/en/products/eos/eos-cloudvision) nor EOS devices, we put [Mock](https://docs.python.org/3/library/unittest.mock.html) in place to emulate API endpoint.

![](/assets/img/test-your-python-network-automation/pytest-mocking.png)

In this article, we will go through the process we put in place to support ansible-cvp development and go through some basic examples.

## Create a mock of CVPRAC / CVP

Since `arista.cvp` collection is in charge of communicating with Cloudvision, we need to emulate it in our testing. So the mock will recreate all cvprac function and use static data coming from CV.

So first of all, we need to load [MagicMock](https://docs.python.org/3/library/unittest.mock.html) and [cvprac](https://github.com/aristanetworks/cvprac) libraries

```python
from unittest.mock import MagicMock, create_autospec
import pprint
import logging
from cvprac.cvp_client import CvpClient, CvpApi
```

Now we can create a Mock from [MagickMock](https://realpython.com/python-mock-library/). This approach gives us option to create side_effects to define how function should work.

```python
def get_cvp_client(cvp_database:  MockCVPDatabase) -> MagicMock:
    # Instantiate a Mock from CvpClient to support authentication process
    mock_client = create_autospec(CvpClient)

    # Instantiate a Mock from CvpApi to support CVP interaction
    mock_client.api = create_autospec(CvpApi)

    # Create side_effects to expose cvprac methods
    mock_client.api.get_configlets_and_mappers.side_effect = cvp_database.get_configlets_and_mappers

    # Expose created MOCK
    return mock_client
```

As you can see in the [`side_effects`](https://docs.python.org/3/library/unittest.mock.html#unittest.mock.Mock.side_effect) definition, we use cvprac method name as it is automatically populated from the [`create_autospec`](https://docs.python.org/3/library/unittest.mock.html#create-autospec). In our case, we have an object named `cvp_database` where we have all the cvprac method we use. Main reason is this object can instantiate a fake database with user's defined JSON object. [Auospeccing](https://docs.python.org/3/library/unittest.mock.html#autospeccing) also add security guard to recursively speccing attributes and avoid silent error that could break CI.

```python
class MockCVPDatabase:
    """Class to mock CVP database being modified during tests"""
    def __init__(self, configlets_mappers: dict = None):
        self.configlets_mappers = configlets_mappers if configlets_mappers is not None else {}
```

> This code is not a complete one as it can be a bit too long.

And now we have to create our side_effects method. In our example, we will do a side_effect for get_containers method

```python
class MockCVPDatabase:
    """Class to mock CVP database being modified during tests"""
    def get_configlets_and_mappers(self):
        """
        get_configlets_and_mappers Return Mapping for configlets
        """
        return self.configlets_mappers
```

This function exposes data as cvprac would do if it has some real CV connection. So to build this function, you need to define what your JSON structure will be in your MOCK and data structure sent back by cvprac. So in this example, we use the following structure:

```json
{
    "data": {
        "configlets": [
            {
                "key": "configlet_0b5f1972-bb04-4d31-931d-baf7061caa13",
                "name": "spine-2-unit-test",
                "reconciled": "false",
                "configl": "alias mock show ip route",
                "user": "cvpadmin",
                "note": "",
                "containerCount": 0,
                "netElementCount": 0,
                "dateTimeInLongFormat": 1640858616983,
                "isDefault": "no",
                "isAutoBuilder": "",
                "type": "Static",
                "editable": "true",
                "sslConfig": "false",
                "visible": "true",
                "isDraft": "false",
                "typeStudioConfiglet": "false"
            },
            //[...]
        ]
    }
}
```

> This output is basically what CV provides when you GET configlet_mappers with curl:

```bash
curl -X 'GET' \
  'https://<cv-ip-address>/cvpservice/configlet/\
    getConfigletsAndAssociatedMappers.do' \
    -H 'accept: application/json'
```

Everything seems cool, but how can we use it with Pytest and build relevant test cases ?

## Configure Pytest

### Create fixture to instantiate DB

Let's consider writing tests for [`cv_facts_v3`](https://github.com/aristanetworks/ansible-cvp/blob/devel/ansible_collections/arista/cvp/plugins/module_utils/facts_tools.py) as it is a fairly easy one.

Let's create a [__fixture__](https://docs.pytest.org/en/7.0.x/explanation/fixtures.html?highlight=fixture) to instantiate environment each time we run a test. This fixture creates an object named `database` from `mock.MockCVPDatabase` and uses data from test environment.

```python
#!/usr/bin/python
# coding: utf-8 -*-

from __future__ import (absolute_import, division, print_function)
import pytest
from tests.lib import mock, mock_ansible
from tests.data import facts_unit
from ansible_collections.arista.cvp.plugins.module_utils.facts_tools import CvFactsTools

LOGGER = setup_custom_logger(__name__)

@pytest.fixture
def cvp_database():
    database = mock.MockCVPDatabase(
        devices=facts_unit.MOCKDATA_DEVICES,
        configlets=facts_unit.MOCKDATA_CONFIGLETS,
        containers=facts_unit.MOCKDATA_CONTAINERS,
        configlets_mappers=facts_unit.MOCKDATA_CONFIGLET_MAPPERS
    )
    yield database
    LOGGER.debug('Final CVP state: %s', database)
```

This approach is based on a generic content available for all cases. But another approach can be to leverage indirect parametrisation to inject data specific to each test. We won't go through each approach here since both presents pro and cons. But if you want to read more, you can refer to [this page](https://docs.pytest.org/en/6.2.x/example/parametrize.html#indirect-parametrization).

Another fixture is used to instantiate `arista.cvp` utils and is based on the same approach:

```python
@pytest.fixture()
def fact_unit_tools(request, cvp_database):
    cvp_client = mock.get_cvp_client(cvp_database)
    instance = CvFactsTools(cv_connection=cvp_client)
    LOGGER.debug('Initial CVP state: %s', cvp_database)
    yield instance
    LOGGER.debug('Mock calls: %s', pprint.pformat(cvp_client.mock_calls))
```

As you can see in example above, we log state when we instantiate and release data in fixture. So we can see changes in Cloudvision database and inspect result of tests anytime. Also, `yield` is header of code runs when we release code after test execution.

### Parametrize tests

Pytest aims to create a test we can run against different dataset. These dataset can be pass to the method by using what we call [__parametrize__](https://docs.pytest.org/en/7.0.x/example/parametrize.html?highlight=parametrize). In our situation, we use same data we loaded in our fixture so it is easy to validate results.

In our example, `facts_unit.MOCKDATA_DEVICES` is a list, so we do not have to call a method to transform and pytest will iterate to generate 1 test case per entry. Also by default, name for tests genrated by Pytest will have no sense. So we use a function to generate IDs automatically

```python
def generate_test_ids_dict(val):
    if 'name' in val.keys():
        # note this wouldn't show any hours/minutes/seconds
        return val['name']
    elif 'hostname' in val.keys():
        return val['hostname']

@pytest.mark.parametrize
(
    "device_def",
    facts_unit.MOCKDATA_DEVICES,
    ids=generate_test_ids_dict
)
```

### build a test case

So now we are ready to build a test with all these elements:

- Use fixture to instantiate DB.
- Use fixture to instantiate a module utils.
- Use a parametrize to increase test examples.

```python
@pytest.mark.usefixtures("fact_unit_tools")
@pytest.mark.usefixtures("cvp_database")
@pytest.mark.parametrize
(
    "device_def",
    facts_unit.MOCKDATA_DEVICES,
    ids=generate_test_ids_dict
)
def test_CvFactsTools__device_get_configlets_with_configlets
(
    fact_unit_tools,
    device_def
):
    # Only target devices that have configlets configured.

    # Run command to get configlet for device on Cloudvision
    result = fact_unit_tools._CvFactsTools__device_get_configlets(
        netid=device_def['systemMacAddress']
    )

    # If we can get the device in the input data (i.e. not from cloudvision),
    # we can play with assert
    if device_def['systemMacAddress'] in [x['objectId']
        for x in facts_unit.MOCKDATA_CONFIGLET_MAPPERS['data']['configletMappers']]:
        assert len(result) > 0
    else:
        # If not covered by test (i.e. has no configlet attached), we simply skip it
        pytest.skip("skipping DEVICES with no configlet")
```

This example run a method to ask for all configlets for the device and then read the database to compare result with what the method provided.

Because `fact_unit_tools` has been provisioned with a cvp_client based on our mock, the function `__device_get_configlets` is getting data based on the correct CVP format and well formated by the mock of CVPRAC with `get_configlets_and_mappers` method.

## Run testing

Execute testing is fairly easy and use basic pytest CLI commands. You don't have to build a lab somewhere to run unit test of your automation. So CI can catch very quickly any deviation as you can write test for any single method or object you have in your code.

```python
pytest \
    -rA -q --cov-report term:skip-covered \
    -v --html=report.html --self-contained-html --cov-report=html \
    --color yes \
    --cov=ansible_collections.arista.cvp.plugins.module_utils \
    --log-cli-level=INFO -m 'image' unit/
```

In our case, we can easily test our ansible modules functions before starting to play with ansible. And in reality we do unit test with Pytest & Mock, integration tests with Pytest and CV and finally we can do manual or leverage Molecule to test from Ansible

and a big shout out to the whole ansible-cvp development team: [Tamas](https://github.com/noredistribution), [Guillaume](https://github.com/guillaumeVilar), [Matthieu](https://github.com/mtache) for ideas, brainstorming, testing, and implementation.

## Resources

- [Pytest](https://docs.pytest.org/en/7.0.x/)
- [unittest.mock](https://docs.python.org/3/library/unittest.mock.html) for Pytest
- What is a Pytest [Fixture](https://docs.pytest.org/en/7.0.x/explanation/fixtures.html?highlight=fixture)
- [Parametrizing Tests](https://docs.pytest.org/en/7.0.x/example/parametrize.html?highlight=parametrize)
