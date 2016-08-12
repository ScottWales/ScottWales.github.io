---
layout: post
title: xarray and terrabyte datasets
---

As models get to higher and higher resolutions researchers have an ever
increasing amount of output data to sift through.  The [1/10th degree ocean
model](https://www.youtube.com/watch?v=8VMSF28J9H4) currently being used in the
Centre spits out 70 GB of data per month for temperature alone.

This has got me interested in the different ways to slice and dice such large
datasets.

The standard file format for climate data is NetCDF, and there are a wide
variety of Python libraries that can read this format and return Numpy arrays
to work with. Libraries like `netcdf4` and `xarray` even load the varaiables
within the file only as you use them to save memory.

This is still a problem for very large files however, as even a single variable
may not fit into the computer's memory. The `dask` library is one way to get
around this, it combines with Xarray to allow easy use of bigger than memory
datasets.

Dask breaks up a Numpy array into chunks, and then will convert any operations
performed on that array into lazy operations. This means when you add two Dask
arrays together nothing happens immediately, calculations are only done when
the data is used, say by saving to a file or printing to the screen.

Doing calculations in this way allows for some clever optimisations. If a value
is not needed it will not be calculated, and if a value is needed in two
different parts of a calculation it will only be calculated once.

A Dask array is created by adding a `chunks` argument to Xarray's
`open_dataset`:

{% highlight python3 %}
import xarray

filename = '/g/data/v45/mom/mom01v3/output055/temp_snap.nc'
data = xarray.open_dataset(filename, chunks={'time':5, 'st_ocean':1})
{% endhighlight %}

This argument is a dictionary saying how big each slice should be in each
dimension, in this case each chunk has 5 time values, a single vertical level
and the entire horizontal domain. The exact settings require some thought - for
this example I'm going to calculate the mean global surface temperature. Since
I'm only looking at the surface I only need to load one vertical level, and
loading the whole horizontal field at once means less communcations when
calculating the mean.

Once the Dask array has been created you can then use Xarray as normal.
I quite like Xarray's interface - it's powerful, and lets you work in a
functional manner without needing to worry about writing loops.

{% highlight python3 %}
surf_temp = data.temp.isel(st_ocean = 0)
mean_surf_temp = surf_temp.mean(dim=('yt_ocean','xt_ocean'))
{% endhighlight %}

Up to now nothing's actually happened. No data has been read (apart from some
metadata to determine the variable size). Only when you write the dataset do
things start happening:

{% highlight python3 %}
# Enable a Dask progress bar
from dask.diagnosics import ProgressBar
ProgressBar().register()

new = xarray.Dataset({'mean_surf_temp':mean_surf_temp})
new.attrs['history'] = "%s: Mean surf temp of %s"%(datetime.now(), filename)
new.to_netcdf('mean.nc')
{% endhighlight %}

When `to_netcdf()` is called data needs to be moved, so each chunk is loaded
one by one, the surface mean is calculated, then the parts are glued back
together to be written to the output file.

## Installing

Dask and Xarray are available in Anaconda:

{% highlight bash %}
conda create -n xarray python=3 xarray
source activate xarray
{% endhighlight %}

## Combining multiple files

If the data is split across multiple files (say one file per month of data) you
can either load all files at once with `xarray.open_mfdataset()`, or you can
load the files individually and join them later with `xarray.concat()`.

## Parallel Processing

Dask can run calculations in parallel using either threads or multiple
processes. Unfortunately only the default threaded method appears to work with
NetCDF files at the moment, and even then all NetCDF IO is a serial operation.
If you're doing simple operations this means you may not see much speed-up from
using multiple cpus.

You can set the number of threads Dask uses with:
{% highlight python3 %}
import dask
from multiprocessing.pool import ThreadPool

ncpus = 4
dask.set_option(pool = ThreadPool(ncpus))
{% endhighlight %}

## Profiling

Dask also has a nifty profiler built-in, which lets you see CPU usage and the
like:

{% highlight python3 %}
from dask.diagnostics import Profiler, ResourceProfiler, visualize
prof = Profiler()
prof.register()
rprof = ResourceProfiler()
rprof.register()

# Run your program...

visualize([prof, rprof], show=False)
{% endhighlight %}

This will create a file `profile.html` that you can open in a browser to see
the profile report (use `show=True` to automatically launch a browser, this
won't work on Raijin)
