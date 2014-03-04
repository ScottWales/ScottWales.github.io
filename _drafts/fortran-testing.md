---
layout: post
title: "Testing Fortran Programs"
---

Testing your code is an important part of software development. It helps you to
ensure your code is working as you create it as well as providing a way to show
that it's still working as you make changes.

Goal is to get the spatial derivative of a scalar field

Create a sketch
===============

Start by writing out how you'd like to call the function

{% highlight fortran linenos=table %}
! Calculate a spatial derivative
a = SpatialDerivative(b)
{% endhighlight %}

Build a test case around this, giving sample inputs and outputs

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
    call testSpatialDerivative
end program
{% endhighlight %}

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

    subroutine assignScalar(f, v)
        type(Field), intent(inout) :: f
        real, intent(in) :: v

        f%values = v
    end subroutine

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
single one. This can be fixed using the [`ALL`](http://software.intel.com/sites/products/documentation/doclib/stdxe/2013/composerxe/compiler/fortran-lin/GUID-CF39CC9E-9EF8-4F18-B13B-3F17655EC137.htm) intrinsic to reduce the array:

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
instance parallelising it with OpenMP or MPI. 


Keep it working
===============

 - Automatic testing with Travis
 - Unit testing frameworks
