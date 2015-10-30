---
title: Distributing Python Libraries, Part 2
layout: post
tags: python
---

In my [last post]({% post_url 2015-10-23-pythondist1 %}) I talked about how to
set up a new python library by creating a new source repository and the
`setup.py` script that `pip` uses to install it.  In this post I'll talk about
setting up testing for the library, and automating code quality checks using
services like [Travis-ci][] and [Landscape][].

You can see the end results at <https://github.com/coecms/ARCCSSive>

Testing
-------

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/nasamarshall/10673972765/in/photolist-hgdUec-hDa1KJ-cA5Lw3-bzHzXh-bs146-fBXyhs-iqQJyW-fUHace-8Tuan2-fuzqZa-dCBa1m-xiEN28-2jLUeT-odvyqn-nyGQf6-qhsUF-jQNHdH-pxXJSz-9urXjF-9zNLoL-6vMKp9-nSEaQ6-p3de8a-scErVz-5UaXfg-8hy3kB-92eVtj-5iQzpn-cXp8W7-cVbaUN-7vFcMw-2rgKXg-6qY8BY-xqNu4-qsBq8R-bQVpWB-7n8Lg8-6SNJgc-5uyxMS-nfdnWu-8J72u9-fQNiVk-mcNzDa-uWas7Q-a2pZX-9d9hUM-pjjiar-zq2cdL-8x3Nde-nu2raN" title="Wind Tunnel Testing, Scale Model of Space Launch System (NASA, SLS, 11/04/13)"><img src="https://farm3.staticflickr.com/2862/10673972765_6219ee18ae_z.jpg" width="640" height="431" alt="Wind Tunnel Testing, Scale Model of Space Launch System (NASA, SLS, 11/04/13)"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

There are quite a few libraries around for testing Python code. I like to use
[py.test][], as it doesn't require much boiler-plate code to use, you just
write functions with an name starting with `test_`:
{% highlight bash %}
$ pip install pytest
$ py.test
{% endhighlight %}

Testing is important as it's what lets you know your code is doing what it's supposed to be doing

A useful way to go about creating tests for your code is to do what's called
'test-driven' development. In this method you write tests before the code,
which encourages you to think about how the code's actually going to be used.

For ARCCSSive, I wanted users to be able to easily select experiments from the
CMIP5 database. My idea was to have a `query()` function that would take
various attributes to search for, and would return something that users could
loop over to get the output files:

{% highlight python %}
# tests/test_cmip5.py
import ARCCSSive
def test_connect():
    data = ARCCSSive.CMIP5.connect()

    # We want to be able to store connection inforomation in an object
    assert data

def test_query():
    data    = ARCCSSive.CMIP5.connect()
    results = data.query(model = 'ACCESS1_0')

    # Query should return something that's iterable
    assert len(results)

    # We can get the files from the results
    for r in results:
        assert r.files()
{% endhighlight %}

Py.test will by default search the `tests` directory for functions beginning
with `test_`, then run them. The `assert` statement will report an error if
it's argument is `False` or `None`, I'm using it as a way to check functions
are returning something.

At this stage I'm not saying what the return values of the functions should be,
just the properties that I'd like them to have. If I run `py.test` now it will
collect all of the tests and run them for me, reporting any failures:

{% highlight bash %}
$ py.test
============================= test session starts ==============================
platform linux2 -- Python 2.7.3 -- py-1.4.26 -- pytest-2.6.4
plugins: cov, pep8, cache
collected 2 items 

tests/test_cmip5.py FF

=================================== FAILURES ===================================
_________________________________ test_connect _________________________________

    def test_connect():
    >       data = ARCCSSive.CMIP5.connect()
    E       AttributeError: 'module' object has no attribute 'CMIP5'
{% endhighlight %}

The first error is easy to fix - I don't have a `CMIP5` module yet. From here
it's a process of writing code to make the tests pass. Once the tests are
passing we can commit both the code and tests to the repository, then add a new
test for the next bit of functionality we need. As we add functionality to the
library we know from the tests that existing functionality isn't being broken.
You can also add new tests if a bug is reported to ensure that it's fixed in
the next release.

[py.test]: http://pytest.org/latest/

Automation
----------

<a data-flickr-embed="true"  href="https://www.flickr.com/photos/free-stock/6837286686/in/photolist-bqbT8d-dX2FY1-b5EC2z-9jjwHP-irHcH7-7eiuud-7eRUA6-9jnDQy-9jnDvC-v7miCm-vLSLFg-9jjx6R-c19J1f-c19J31-9jjwWF-9jjwZR-o9dqN6-9jnD6U-a621F2-dbRvHc-4M6c9A-a64Sw1-nYoaEB-PFFes-d5Cqa-qTNCEE-qaxH3U-6TEVKo-a63abh-bF7fBn-j39bZj-gpFLth-auGFo7-5gkkDN-a61ZWe-qEU8y6-rk82kC-qEFGC9-rBFkE8-c19s8E-c19s7j-c19s5E-c19s6w-c19s57-c19s3W-8Ae5yv-9gkJWi-5SE2bA-a63adb-9oNhUi" title="Automation-heating-mixing-bypass-storage-tank__16296"><img src="https://farm8.staticflickr.com/7065/6837286686_5d80379fbc_z.jpg" width="640" height="480" alt="Automation-heating-mixing-bypass-storage-tank__16296"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

Once you've got some tests set up it's a good idea to automate them, so that
they are run whenever you commit code. [travis-ci][] is a service that links up
with Github to run your tests for you, it will also add information to branches
and pull requests on Github saying if tests are passing and send you an email
if tests start failing.

Travis is controlled by a file `.travis.yml` in the top level of your
repository. This file tells Travis how to install and run your tests as well as
what language versions to use. Travis will build your code on a virtual machine
that's deleted when the run has finished, so you're free to install your own
packages using `pip` (or `apt-get`, though you'll need to request `sudo` as well)

{% highlight yaml %}
# .travis.yml
language: python
python:
  - 2.7
  - 3.4

install:
  - pip install --upgrade pytest coverage codecov
  - pip install .

script:
  - coverage run --source ARCCSSive -m py.test

after_success:
  - codecov
{% endhighlight %}

This file says to run two sets of tests, using Python versions 2.7 and 3.4.
This is a good way to make sure your library is compatible with Python 3
without needing to install extra libraries on your own computer. Nowdays
supporting Python 2 and 3 at the same time is pretty simple, but there are some
corner cases that can catch you if you don't test regularily. With each Python
version Travis will run the `install` commands to install the package and some
testing libraries, then it will run the `script` commands and report any
errors.

With the `.travis.yml` file in your repository you can log into Travis with
your Github account, then activate any repositories you want to test. Travis
will run whenever you add a commit or someone makes a pull request.

[travis-ci]: https://travis-ci.org

[ARCCSSive on Travis](https://travis-ci.org/coecms/ARCCSSive):
[![Build Status](https://travis-ci.org/coecms/ARCCSSive.svg?branch=master)](https://travis-ci.org/coecms/ARCCSSive)

### Coverage

You'll notice that the Travis script above isn't running `py.test` directly,
instead it's using a program called `coverage`. This is a helper function that
measures what lines in your code are actually being run, so that you can see
where you're missing tests. The information is uploaded to a service called
[codecov][] to produce a nice view of the test coverage. Codecov can also make
suggestions on what functions you could add tests to, and gives you a badge to
add to your `README.md` to show the percent of code you're testing.

[codecov]: https://codecov.io

[ARCCSSive on Codecov](https://codecov.io/github/coecms/ARCCSSive):
[![codecov.io](https://codecov.io/github/coecms/ARCCSSive/coverage.svg?branch=master)](https://codecov.io/github/coecms/ARCCSSive?branch=master)

### Code Quality

Another helpful web service to use with Python libraries is [landscape][],
which is an automated code review tool. Landscape checks your code's quality,
for instance making sure it adheres to the [PEP 8][] standard and that
functions aren't too large or complex. It's not neccessary to follow all of its
suggestions, but they can help make sure your code is understandable to others.
Like Travis you can activate your repository on Landscape using your Github
account, and it also gives you a badge for your `README.md`.

[landscape]: https://landscape.io
[PEP 8]: https://www.python.org/dev/peps/pep-0008

[ARCCSSive on Landscape](https://landscape.io/github/coecms/ARCCSSive):
[![Code Health](https://landscape.io/github/coecms/ARCCSSive/master/landscape.svg?style=flat)](https://landscape.io/github/coecms/ARCCSSive/master)

Next Steps
----------

Now that our code is working, tested and reasonably readable the next step is
to put together some documentation. In the next post I'll talk about using
Sphinx to semi-automatically document your code, and how to upload the
documentation to ReadTheDocs.
