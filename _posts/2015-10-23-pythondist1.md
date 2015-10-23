---
title: Distributing Python Libraries
layout: post
tags: python
---

Recently I created a Python library called ARCCSSive, which allows researchers
to search through the CMIP5 archives mirrored at NCI. I also wanted to use this
as an opportunity to investigate how to make a high-quality library that can
easily be used by others. This includes putting the library into public source
control, setting up documentation, unit tests & coding standards, and
publishing the library to the Python Package Index. I also wanted all of this
to be as automated as possible - so that I could create a new release of the
library with a single command and know it's tested, documented and available to
others.

You can see the results at <https://github.com/coecms/ARCCSSive>

Source control
--------------

The first step before starting any programming project is to set up version
control. This lets you keep a backup copy of the code on an external service,
and allows you to trace through the history of any changes that you make.
Version control systems also make it simple for you to accept code
contributions from other people - there are special tools for merging two
different versions of a file.

There are a number of different version control programs you can use, [Git][]
and [Subversion][] (svn) are the ones you're most likely to run into. They all
do substantially the same thing - you create a repository, select the files you
want to monitor, then regularly upload any changes to the server.

There are also a number of servers that let you host open-source projects for
free (and some that let you host private projects as well). I primarily use
[Github][], though you can also check out [Gitlab][] and [Bitbucket][].

I like Github as it integrates with a large number of external services for
things like testing, and supports using both Git and Subversion for the same
repository. The interface for creating a new repository is pretty simple, you
can easily add a `README.md` file to let others know what the repository is
for, create a `.gitignore` file so the source control doesn't add compiler
output and select a licence for your repository.

Choosing a licence is important, as it is what gives other people permission to
use your code. The ARCCSS CMS team normally uses the [Apache][] licence for our
work, which allows other people to copy & use your code provided they keep the
licencing information with the list of authors. An alternative is the [GPL][],
which adds the requirement that users release the source code of their projects
that make use of your work.

Once you've created the repository Github will give you instructions on how to
download the project onto your own computer. Once that's been done we can start
setting up Python:
{% highlight bash %}
$ git clone git@github.com:coecms/ARCCSSive
$ # or
$ svn checkout https://github.com/coecms/ARCCSSive
{% endhighlight %}
From here on I'll only be showing Git commands, but equivalent Subversion
commands are available if you prefer using `svn`.

[Git]: https://git-scm.com
[Subversion]: https://subversion.apache.org
[Github]: https://github.com
[Gitlab]: https://gitlab.com
[Bitbucket]: https://bitbucket.org
[Apache]: https://tldrlegal.com/license/apache-license-2.0-%28apache-2.0%29
[GPL]: https://tldrlegal.com/license/gnu-general-public-license-v3-%28gpl-3%29

Module setup
------------

To start out our library we'll first create a meta-data file - `meta.py` in the
top level of the repository. Things like the repository version number are used in multiple places such as the install script and documentation, it's most convenient to have these in one place.

This script uses `git describe` to get the version number directly from the
repository (the `decode('utf-8')` is for Python3):
{% highlight python %}
# meta.py
from subprocess import Popen, PIPE

# Version of the current repository
describe   = Popen(['git','describe','--dirty','--always'],stdout=PIPE)
version    = describe.communicate()[0].decode('utf-8').strip()

# Most recent release
describe   = Popen(['git','describe','--abbrev=0'],stdout=PIPE)
release    = describe.communicate()[0].decode('utf-8').strip()

name       = 'ARCCSSive'
author     = 'Scott Wales'
copyright  = '2015 ARC Centre of Excellence for Climate Systems Science'
# ...
{% endhighlight %}

The file `setup.py` is what tools like `pip` and `easy_install` use to install 
Python libraries. In the file should be a call to the function
`setuptools.setup()`, which specifies the library name, version, any
dependencies and a list of Python packages that it includes. The latter can be
found automatically with the `setuptools.find_packages()` function, which adds
any directory with an `__init__.py` file:

{% highlight python %}
# setup.py
from setuptools import setup, find_packages
import meta

dependencies = [
    'numpy',
]

setup(
    name             = meta.name,
    version          = meta.version,
    packages         = find_packages(),
    install_requires = dependencies,

    author           = meta.author,
    # ...
    )
{% endhighlight %}

The `setup()` function also allows you to add extra meta-data, such as the
author, licence and a description of the library. This information can be used
by websites like [PyPI][] to help people find your library. More information
about `setup()` can be found in the [Setuptools documentation][Setuptools basic
use], for instance how to [install scripts][Setuptools install scripts]

Once your `setup.py` file is created you should install the module in
'editable' mode. This allows you to make changes to the module's source files
without having to re-install it afterwards.

{% highlight bash %}
$ pip install --user -e .
{% endhighlight %}

[Setuptools basic use]: http://pythonhosted.org/setuptools/setuptools.html#basic-use
[Setuptools install scripts]: http://pythonhosted.org/setuptools/setuptools.html#automatic-script-creation


Next Steps
----------

Once you've got the install script working you should add the new files to
version control and commit your changes:
{% highlight bash %}
$ git add .
$ git commit -a -m "Add setup.py"
$ git push
{% endhighlight %}

To create a Python package you should then create a new directory, and add a
`__init__.py` file to it. This file can be empty, it's mostly a marker for
Python to know which directories to look in:
{% highlight bash %}
$ mkdir ARCCSSive
$ touch ARCCSSive/__init__.py
{% endhighlight %}

You'll now be able to import your library from scripts:
{% highlight python %}
$ import ARCCSSive
{% endhighlight %}

Next I'll look at how to add automatic tests to your library, so that you know
things aren't breaking as you add new code.
