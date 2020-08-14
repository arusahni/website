<!--
    .. title: A DevOps Workflow, Part 2: Continuous Integration
    .. slug: devops-workflow-ci
    .. date: 2017-01-19 09:48:21 UTC-05:00
    .. tags: tech, devops
    .. link:
    .. description: The second part of a series on my sample devops process, focusing on continuous integration.
-->

_This series is a longform version of an internal talk I gave at a former
company. It wasn't recorded. It has been mirrored here for posterity._

Look at you – [all fancy with your consistent and easily-managed development
environment](link://slug/devops-workflow-local-development). However, that's
only half of the local development puzzle. Sure, now developers can no longer
use "it works on my machine" as an excuse, but all that means is they know that
something runs. Without validation, your artisanal ramen may be
indistinguishable from burned spaghetti. This is where unit testing and
continuous integration really prove their worth.

<!-- TEASER_END -->

# Unit Testing

You can't swing a dead cat without hitting a billion different Medium posts and
Hacker News articles about the One True Way to do testing. Protip: there isn't
one. I prefer [Test Driven Development
(TDD)](https://en.wikipedia.org/wiki/Test-driven_development), as it helps me
design for failure as I build features. Others prefer to write tests after the
fact, because it forces them to take a second pass over a chunk of
functionality. All that matters is that you have and maintain tests. If you're
feeling really professional, you should make test coverage a requirement for
any and all code that is intended for production. Regardless, code verification
through linting and tests is a vital part of a good DevOps culture.

## Getting Started

Writing a test is easy. For Python, a preferred language at HumanGeo, there
exist many different test frameworks and tools. One great option is pytest. It
allows you to associate test classes with your code without boilerplate. For
example:

```python
# my_code.py

def get_country(country_code):
    return COUNTRIES.get(country_code)
```

```python
# test_my_code.py

import my_code

def test_get_country(): # All tests start with 'test_'
    assert my_code.get_country('DE') == 'Germany'
```

When executed, the output will indicate success:

```
=============================== test session starts ===============================
platform darwin -- Python 3.6.0, pytest-3.0.5, py-1.4.32, pluggy-0.4.0
rootdir: /private/tmp, inifile:
collected 1 items

test_code.py .

========================== 1 passed in 0.01 seconds ===============================
```

or failure:

```
=============================== test session starts ===============================
platform darwin -- Python 3.6.0, pytest-3.0.5, py-1.4.32, pluggy-0.4.0
rootdir: /private/tmp, inifile:
collected 1 items

test_code.py F

==================================== FAILURES =====================================
________________________________ test_get_country _________________________________

    def test_get_country():
>       assert my_code.get_country('DE') == 'Germany'
E       assert 'Denmark' == 'Germany'
E         - Denmark
E         + Germany

test_code.py:6: AssertionError
============================ 1 failed in 0.03 seconds =============================
```

The inlining of failing code frames makes it easy to pinpoint the failing
assertion, thus reducing unit testing headaches and boilerplate. For more on
pytest, check out Jacob Kaplan-Moss's [great introduction to the
library](http://jacobian.org/writing/getting-started-with-pytest/).

## Mocking

Mocking is vital part of the testing equation. I don't mean making fun of your
tests (that would be downright rude), but instead substituting fake (_mock_)
objects in place of ones that serve as touchpoints to external code. This is
nice because a good test shouldn't care about certain implementation details -
just ensure that all cases are correctly handled. This especially holds true
when relying on components outside of the purview of your application, such as
web services, datastores, or the filesystem.

[`unittest.mock`](https://docs.python.org/3/library/unittest.mock.html) is my
library of choice. To see how it's used, let's dive into an example:

```python
# my_code.py

def country_data_exists():
    return os.path.exists('/tmp/countries.json')
```

```python
# test_my_code.py

from unittest.mock import patch
import my_code

@patch('os.path.exists')
def test_country_data_exists_success(path_exists_mock):
    path_exists_mock.return_value = True
    data_exists = my_code.country_data_exists()
    assert data_exists == True
    path_exists_mock.assert_called_once_with('/tmp/countries.json')

@patch('os.path.exists')
def test_country_data_exists_failure(path_exists_mock):
    path_exists_mock.return_value = False
    data_exists = my_code.country_data_exists()
    assert data_exists == False
    path_exists_mock.assert_called_once_with('/tmp/countries.json')
```

The `patch` function replaces the object at the provided path with a `Mock`
object. These objects use Python magic to accept arbitrary calls and return
defined values. Once the function that uses the mocked object has been invoked,
we can inspect the mock and make various assertions about how it was called.

If you're using the Requests library (which you should always do),
[responses](https://github.com/getsentry/responses) allows you to intercept
specific requests and return custom data:

```python
# my_code.py

import requests

def get_flag_image(country_code):
    response = requests.get(f'http://example.com/flags/{country_code}.png')
    if not response.ok:
        raise MediaDownloadError(f'Error downloading the image: HTTP {response.status_code}:\n{response.text}')
    return response.content
```

```python
# test_my_code.py

import pytest
import responses

@responses.activate # Tell responses to intercept this function's requests
def test_get_flag_image_404():
    responses.add(responses.GET, # The HTTP method to intercept
                  'http://example.com/flags/de.gif', # The URL to intercept
                  body="These aren't the gifs you're looking for", # The mocked response body
                  status=404) # The mocked response status
    with pytest.raises(my_code.MediaDownloadError) as download_error:
        my_code.get_flag_image('de')
    assert '404' in download_error.message
```

More information on mocking in Python can be found
[here](http://fgimian.github.io/blog/2014/04/10/using-the-python-mock-library-to-fake-regular-functions-during-tests/).

# Continuous Integration: Jenkins

Throughout this process, we've been trusting our developers when they say their
code works locally without issue. The better approach here is to _trust, but
verify_. From bad merges to broad refactors, a host of issues can manifest
themselves during the last few phases of task development. A good DevOps
culture accepts that these are inevitable and must be addressed through
automation. The practice of validating the most recent version of your codebase
is called [Continuous Integration
(CI)](https://en.wikipedia.org/wiki/Continuous_integration).

For this, we will use [Jenkins](https://jenkins.io/), a popular open source
tool designed for flexible CI workflows. It has a large community that provides
plugins for integration with common tools, such as
[GitLab](https://about.gitlab.com/), Python Virtual Environments, and various
test runners.

Once Jenkins has access to your GitLab instance, it can:

1. Poll for merge requests targeting the main development branch;  
    ![Jenkins polling for MR screenshot](https://i.imgur.com/WuWIMMa.png)
2. Attempt a merge of the feature branch into the trunk;  
    ![Jenkins MR detection screenshot](https://i.imgur.com/LhlFds9.png)
3. Run your linter;  
    ![Jenkins lint command screenshot](https://i.imgur.com/LhGmD6q.png)
    * Define your acceptable lint severity thresholds  
        ![Jenkins polling for MR screenshot](https://i.imgur.com/aqzc3jM.png)
4. Run unit tests; and  
    ![Jenkins unit test screenshot](https://i.imgur.com/gmLIHxW.png)
5. If any of the above steps result in a failure state, Jenkins will comment on
   the MR. Otherwise, the build is good, and the MR is given the green light.

By integrating Jenkins CI with GitLab merge requests, low quality code can be
detected and addressed before it enters your main branch. Newer versions of
Jenkins even provide for defining your CI workflow as a [file hosted within
your repository](https://jenkins.io/2.0/#pipelines). This way, your pipeline
always corresponds to your codebase. GitLab has also launched a [CI
capability](https://about.gitlab.com/gitlab-ci/) that may also fit your needs.

_This concludes the continuous integration portion of our dive into DevOps. In
the next installment, we'll cover
[deployment](link://slug/devops-workflow-deployment)!_
