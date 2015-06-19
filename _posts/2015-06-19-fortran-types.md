---
layout: post
tags: fortran
title: Custom Types in Fortran
---

Types like `INTEGER` and `LOGICAL` are used by the compiler to make sure you're
passing the correct arguments to functions. This is really helpful when
functions have lots of arguments, as it lets you know if you've missed one, or
entered them in the wrong order. You can create your own types to bundle
together related variables, or to provide more granularity to argument
checking.

Types collect one or more variables into a single bundle. They must be defined
inside a module, a type definition looks like:

{% highlight fortran %}
MODULE field_m
  USE unit_m, ONLY: unit_t
  IMPLICIT NONE
  PRIVATE
  PUBLIC :: field_t
  TYPE field_t
      REAL, ALLOCTABLE :: dat(:,:)
      TYPE(unit_t) :: unit
  END TYPE
END MODULE
{% endhighlight %}

This type combines a Fortran array with a `unit`, allowing functions to check
to make sure you're not doing something silly like adding a length to a
velocity. Combining the two variables in the same type means that they can't
get separated as they get passed between functions - you always know that the
data in the array `dat` has the unit given by `unit`.

To create variables of the type you must first `USE` the module, then declare
variables with '`TYPE(`*type_name*`)`':

{% highlight fortran %}
USE field_m, ONLY: field_t
TYPE(field_t) :: temperature
{% endhighlight %}

The variables contained inside the type are called 'member variables', and can
be accessed using the `%` operator - '*variable*`%`*member*'. A unit-checking
addition function might look like:

{% highlight fortran %}
SUBROUTINE add_fields(b, a)
    USE field_m, ONLY: field_t
    IMPLICIT NONE
    TYPE(field_t), INTENT(IN)    :: a
    TYPE(field_t), INTENT(INOUT) :: b
    IF (a%unit /= b%unit) THEN 
        ! Incompatible units, abort
        CALL EXIT()
    END IF
    b%dat = b%dat + a%dat
END SUBROUTINE
{% endhighlight %}

By default the values of member variables will be undefined. It's common to
create a constructor function with the same name as the type, this should
return a new variable of that type. As with all interfaces you can add multiple
functions to allow the type to be constructed from different arguments, for
instance by specifying the field size or by initialising the field from an
existing array:

{% highlight fortran %}
MODULE field_m
  ! ...
  INTERFACE field_t
      PROCEDURE new_field_xy
      PROCEDURE new_field_array
  END INTERFACE
CONTAINS
  FUNCTION new_field_xy(nx, ny, unit)
      USE constants_m, ONLY: NAN
      IMPLICIT NONE
      INTEGER, INTENT(IN) :: nx
      INTEGER, INTENT(IN) :: ny
      TYPE(unit_t), INTENT(IN) :: unit
      TYPE(field_t) :: new_field

      ALLOCATE(new_field%dat(nx, ny))
      new_field%dat = NAN
      new_field%unit = unit
  END FUNCTION

  FUNCTION new_field_array(dat, unit)
      IMPLICIT NONE
      REAL, INTENT(IN) :: dat(:,:)
      TYPE(unit_t), INTENT(IN) :: unit
      TYPE(field_t) :: new_field

      ALLOCATE(new_field%dat, SOURCE=dat)
      new_field%unit = unit
  END FUNCTION
END MODULE
{% endhighlight %}

You can then initialise variables by calling the constructor:

{% highlight fortran %}
temperature = field_t(10, 15, kelvin)
{% endhighlight %}

From here it's also possible to add operators to the type, so that you can add
two fields with `+`, scale a field by a constant with `*`, or even add entirely
new operators. You can also extend a type, creating new types which contain the
same data but that you can control how it's used - say a `temperature_field_t`
and a `pressure_field_t` so that you don't confuse the two fields in function
arguments.
