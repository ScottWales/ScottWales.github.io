---
layout: post
title: "Testing Fortran Programs"
tags:
 - fortran
 - testing
---

Testing your code is an important part of software development. It helps you to
ensure your code is working as you create it as well as providing a way to show
that it's still working as you make changes.

There are a variety of ways you can go about testing your code. In this article
I'll go over some simple manual testing of a function, later we'll go over ways
to automate this so that our code is always tested.

There are two things to keep in mind when developing tests - functions should
be specific in what they do, so that the amount of testing you need for each
one is limited, and tests should run quickly, you want to be able to run your
tests as often as possible so running them shouldn't interrupt your work.

Create a sketch
===============

Our goal today is to get the spatial derivative of a field, which we can then
use in the timestepping of a model (e.g. something like the heat equation
\\(\frac{\partial u}{\partial t} = \alpha\nabla^2 u\\))

Start by writing out how you'd like to call the function:
{% highlight fortran linenos=table %}
! Calculate a spatial derivative
a = SpatialDerivative(b)
{% endhighlight %}

Once you know how you're going to call the test create a testing function
around the call. This should set up some sample inputs and outputs, then
compare the results against the expected value. At this point we're just
working out what types and helper functions we'll need.

{% highlight fortran linenos=table %}
! testField.f90
subroutine testSpatialDerivative
    use fieldmod
    type(Field) :: source, dest, expected
    
    ! Initialise
    source   = 1.0
    dest     = -9999.0 ! Starts with invalid content
    expected = 0.0
    
    ! Calculate a spatial derivative
    dest = SpatialDerivative(source)

    ! Check
    if (.not. (dest == expected)) then
        write(*,*) "Test failed"
    end if
end subroutine

program testField
    ! Other tests ...

    call testSpatialDerivative
end program
{% endhighlight %}

In the test we're using a custom `Field` type which holds all the information
about a scalar field (size, resolution, values etc.) rather than a single
array. This will let us add information to the fields in the future without
having to rewrite every functon call, as well as provide some extra
type-checking by the compiler.

Make it compile
===============

Create the minimum to get the test to compile. If functions are trivial fill
them in, otherwise we just want to make sure we've got the functions and
interfaces correct.

{% highlight fortran linenos=table %}
! field.f90
module fieldmod
    type Field
        real, dimension(20,20) :: values
    end type

    interface assignment(=)
        procedure assignScalar
    end interface
    interface operator(==)
        procedure testFieldEqual
    end interface
contains
    function SpatialDerivative(f) result(deriv)
        type(Field), intent(in) :: f
        type(Field) :: deriv

        ! Dummy implementation
        deriv = f
    end function

    ! called like 'f = 1.0'
    subroutine assignScalar(f, v)
        type(Field), intent(inout) :: f
        real, intent(in) :: v

        f%values = v
    end subroutine

    ! called like 'c = (a == b)'
    function testFieldEqual(a, b) result(c)
        type(Field), intent(in) :: a, b
        logical :: c

        c = a%values == b%values
    end function
end module
{% endhighlight %}

Try compiling the program (make sure to turn warnings on):

    $ gfortran -Wall -Wextra -Werror -fimplicit-none field.f90 testField.f90 -o testField

If there are any errors with the function calls we want to fix them here, at
this point the test should run and print 'test failed'.

In this case there's a compilation error in the `testFieldEqual` function, as
`a%values == b%values` returns an array of `logical` values, rather than a
single one. This can be fixed using the
[`ALL`](http://software.intel.com/sites/products/documentation/doclib/stdxe/2013/composerxe/compiler/fortran-lin/GUID-CF39CC9E-9EF8-4F18-B13B-3F17655EC137.htm)
intrinsic to reduce the array to a single value:

{% highlight fortran linenos=table %}
    function testFieldEqual(a, b) result(c)
        type(Field), intent(in) :: a, b
        logical :: c

        c = ALL(a%values == b%values)
    end function
{% endhighlight %}

Make it work
============

Next step is to fix the failing test by filling in the implementation of
`SpatialDerivative`. We do this last as it's likely the most complex operation
-- we want everything else working first so that we know the utility functions
aren't causing errors.

{% highlight fortran linenos=table %}
    function SpatialDerivative(f) result(deriv)
        type(Field), intent(in) :: f
        type(Field) :: deriv

        real, dimension(:,:), allocatable :: dfdx, dfdy
        integer :: minx, maxx, miny, maxy

        minx = LBOUND(f%values, dim=1)
        maxx = UBOUND(f%values, dim=1)
        miny = LBOUND(f%values, dim=2)
        maxy = UBOUND(f%values, dim=2)

        allocate(dfdx(minx:maxx,miny:maxy))
        allocate(dfdy(minx:maxx,miny:maxy))
        dfdx = 0
        dfdy = 0

        ! dfdx(i) = (x(i+1) - x(i-1))/2
        dfdx(minx+1:maxx-1,miny+1:maxy-1) = &
            (f%values(minx+1:maxx-1,miny+1:maxy-1) &
            - f%values(minx:maxx-2,miny+1:maxy-1)) / 2

        ! dfdy(i) = (x(i+1) - x(i-1))/2
        dfdy(minx+1:maxx-1,miny+1:maxy-1) = &
            (f%values(minx+1:maxx-1,miny+2:maxy) &
            - f%values(minx+1:maxx-1,miny:maxy-2)) / 2

        deriv%values = dfdx + dfdy

        deallocate(dfdx)
        deallocate(dfdy)
    end function
{% endhighlight %}

Once we've got something that passes our initial test we can add more tests to
be sure we've picked up edge cases:

{% highlight fortran linenos=table %}
subroutine testSpatialDerivativeLinear
    use fieldmod
    type(Field) :: source, dest, expected
    integer :: i, j
    
    ! Initialise
    do j=1,20
        do i=1,20
            source%values(i,j) = real(i + 2*j)
            expected%values(i,j) = 3.0
        end do
    end do
    dest     = -9999.0 ! Starts with invalid content
    
    ! Calculate a spatial derivative
    dest = SpatialDerivative(source)

    ! Check
    if (.not. (dest == expected)) then
        write(*,*) "Test failed"
    end if
end subroutine
{% endhighlight %}

This can help highlight any assumptions -- in this case the test will fail both
because it doesn't take into account boundary conditions and because the
floating point values are unlikely to be exactly equal. The first issue can be
helped by adding documentation to the `SpatialDerivative` function, while the
latter may require adjustment to `testFieldEqual`.

Make it fast
============

Once your function is working correctly you can then start to optimise it, for
instance parallelising it with OpenMP or MPI. As you work continue running the
tests to make sure the changes aren't breaking anything.

This is also the point where you can try profiling your code to identify any
slow spots. Since tested functions are generally smaller it's easy to identify
bottlenecks from the profiler - you can then go through and optimise individual
functions.

Keep it working
===============

Once your code is passing its tests it's important to keep it that way. To do
this you'll want to keep running your tests as part of your regular build
process.

If you use Makefiles it's common to have a 'check' target that builds and runs
all of a program's tests, e.g.
{%highlight make linenos=table%}
TESTS += testField
check: $(TESTS)
	# Run each test in a loop
	@ for test in $^; do echo $test; ./$test; done
{%endhighlight%}

There are tools that will run your tests for you each time you commit changes
to your source repository, then email you if any of the tests fail. This is
especially useful if you're sharing your code with other people, as they can
tell at a glance if the current version of the code is passing all of its
tests. [Travis](http://travis-ci.org) is one such tool, you can link it to a
repository on Github so it will build your program each time the repository
gets pushed.

As you add more tests to your code you can also think about using testing
frameworks. These are tools that help simplify creating and running tests,
providing predefined test functions to use and automatically printing a summary
of test results every time you check your code. A good starting point for
Fortran is [pFunit](http://sourceforge.net/projects/pFunit), which was
developed by NASA for testing MPI programs.

You can see some examples of these features in my
[fortran-build-testing](https://github.com/ScottWales/fortran-build-testing)
repository, including an example of test-driven development in the 'operations'
branch.
