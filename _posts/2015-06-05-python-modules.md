---
title: Python Environment Modules
layout: post
tags: hpc python
---

Raijin, [NCI](http://nci.org.au)'s supercomputer, like many HPC environments uses
[Environment Modules](http://modules.sourceforge.net) to provide access to programs
and libraries. Using Modules allows different users on the supercomputer to
access different versions of compilers and libraries without having them
conflict or messing about with PATH environment variables. We've been using
environment modules in the ACCESS area for a while now to provide access to
Climate focused tools and libraries, and have built up a decent stash of Python
libraries.

NCI themselves don't support Python libraries in their central directory,
primarily because managing dependencies between them can be a pain. As Python
is getting used increasingly frequently in climate science however it's
important for us to support them as part of ACCESS. I wanted a system where
installing new libraries from [PyPI](pypi.python.org), the central Python
package index, was as easy as running:

{% highlight bash %}
$ admin/install_python_lib.sh netCDF4
{% endhighlight %}

To do this, the install script needed to download & install the library to a
custom path as well as set up the Modulefile needed to load the library into a
user's environment.

Libraries are installed into a path that looks like
`apps/pythonlib/$LIBRARY/$VERSION`, so that multiple versions of a library can
be supported at the same time. By default the latest version is installed, but
I don't know what version that is without doing the installation. Happily PyPI
provides an API to get what versions of a library are available, you can get
the most recent with a script like:

{% highlight python %}
# Use PyPI's API to find the latest version of the given library
import xmlrpclib
import sys
client = xmlrpclib.ServerProxy('http://pypi.python.org/pypi')
releases = client.package_releases(sys.argv[1])
if len(releases) > 0:
    print releases[0]
else:
    exit(-1)
{% endhighlight %}

Knowing the version number, it's then a simple matter of using `easy_install`
to perform the installation:

{% highlight bash %}
VERSION=$(latest_pypi_version $LIBRARY)
PREFIX=$APPS/$LIBRARY/$VERSION
mkdir -p $PREFIX
easy_install --install_dir=$PREFIX $LIBRARY=$VERSION
{% endhighlight %}

The Modulefile used by Environment Modules is written in a domain specific
language based on TCL. Commands let you modify environment variables, primarily
`PATH` variables used when searching for commands.

A basic Modulefile might look like:

{% highlight tcl %}
#%Module
set prefix "/apps/pythonlib/netCDF4/1.0.4"
prepend-path PATH       $prefix
prepend-path PYTHONPATH $prefix
{% endhighlight %}

This adds the library path to the environment variables `PATH` and
`PYTHONPATH`. Since Environment Modules are independent of the shell you're
using the same file works for both Bash and Csh users.

For the ACCESS modules I've also created a generic include file, which sets up
paths for the standard `bin`, `include`, `lib` directory setup as well as
creating help text for the `module help` command. This looks like:

{% highlight tcl %}
module-whatis "[module-info name]: $help"

proc ModulesHelp { } {
    variable help
    variable url
    variable install-contact
    puts stderr "Module [module-info name]"
    puts stderr "$help"
    puts stderr "Information available at $url"
    puts stderr "Contact ${install-contact} for more information"
}

# Only add directories if they exist
if [ file exists $prefix/bin ] {
    prepend-path PATH $prefix/bin
}
if [ file exists $prefix/lib ] {
    prepend-path RPATH           $prefix/lib
    prepend-path LD_RUN_PATH     $prefix/lib
    prepend-path LIBRARY_PATH    $prefix/lib
    prepend-path LD_LIBRARY_PATH $prefix/lib
}
# ...
{% endhighlight %}

Of note are the multiple variables used for the library path. This makes sure
both static and dynamic libraries are found, while setting `LD_RUN_PATH` means
the linker will save the full path to shared libraries. This means that the
module doens't need to be loaded to run programs.

Having this setup means that we can quickly respond to requests for new
installs - provided the library is listed on PyPI our response is pretty much
instant. The full list of Python modules we have available can be found on the
[ACCESS wiki](https://accessdev.nci.org.au/trac/wiki/Raijin%20Apps).
