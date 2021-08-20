.. _handling_constraints:

Handling Constraints
====================

A very common scenario that we face when working with tabular data is
finding columns that have very particular relationships between them
which are very hard to model and easily confuse the Tabular Models.

Some simple examples of these scenarios include:

-  A table that has the columns ``country`` and ``city``: In such
   scenario, it might be very hard to learn which country each city
   belongs to, and when sampling probabilistically, the model is likely
   to end up generating invalid country/city combinations.
-  A table that contains both the ``age`` and the ``date of birth`` of a
   user. The model will learn the age and date of birth distributions
   and mostly generate valid combinations, but in some cases it might
   end up giving back ages that do not correspond to the given date of
   birth.

These kind of special relationships between columns are called
``Constraints``, and **SDV** provides a very powerful and flexible
mechanism to take them into account and guarantee that the sampled data
always respects them.

Let us explore a few ``Constraint`` examples and learn how to handle
them:

Load a Tabular Demo
-------------------

We will start by loading a small table that contains data with some
constraints:

.. ipython:: python
    :okwarning:

    from sdv.demo import load_tabular_demo

    employees = load_tabular_demo()
    employees


This step loaded a simple table that gives us some basic details about
simulated employees from several companies.

If we observe the data closely we will find a few **constraints**:

1. There should be at most one ``employee_id`` for every employee
   at a ``company``.
2. Each ``company`` has employees from two or more ``departments``, but
   ``department`` names are different across ``companies``. This implies
   that a ``company`` should only be paired with its own ``departments``
   and never with the ``departments`` of other ``companies``.
3. We have an ``age`` column that represents the age of the employee at
   the date when the data was created and an ``age_when_joined`` that
   represents the age of the employee when they joined the ``company``.
   Since all of them joined the ``company`` before the data was created,
   the ``age_when_joined`` will always be equal or lower than the
   ``age`` column.
4. We have a ``years_in_the_company`` column that indicates how many
   years passed since they joined the company, which means that the
   ``years_in_the_company`` will always be equal to the ``age`` minus
   the ``age_when_joined``.
5. We have a ``salary`` column that should always be rounded to 2
   decimal points.
6. The ``age`` column is bounded, since realistically an employee can only be
   so old (or so young).
7. The ``full_time``, ``part_time`` and ``contractor`` columns
   are related in such a way that one of them will always be one and the others
   zero, since the employee must be part of one of the three categories.

How does SDV Handle Constraints?
--------------------------------

**SDV** handles constraints using two different strategies:

Transform Strategy
~~~~~~~~~~~~~~~~~~

When using this strategy, **SDV** applies a transformation to the data
before learning it in a way that allows the model to better capture the
data properties. For example, if we have one column that needs to be
always greater than the other one, SDV can do the following:

1. Replace the higher column with the difference between the two
   columns, which will always be positive.
2. Model the transformed data and sample new values.
3. Recompute the value of the high column by adding the values of the
   lower column to it.

The **Transform** strategy is very efficient and does not affect the
speed of the modeling and sampling process, but in some cases might
affect the quality of the learning process or simply not be possible.

Reject Sampling Strategy
~~~~~~~~~~~~~~~~~~~~~~~~

In the cases where applying a **Transform** strategy is not possible or
may affect the quality of the learning process, **SDV** can apply a
**Reject Sampling** strategy.

When using this strategy, **SDV** validates the sampled rows, discards
the ones that do not adjust to the constraint, and re-samples them. This
process is repeated until enough rows have been sampled.


Predefined Constraints
----------------------

Let us go back to the demo data that we loaded before and define
**Constraints** that indicate **SDV** how to work with this data.

Unique Constraint
~~~~~~~~~~~~~~~~~

Sometimes a table may have a column that is not the primary key, but still needs
to be unique throughout the table. In some cases, there may even be a collection
of columns for which each unique combination of their values can only show up once
in the table. The ``Unique`` constraint enforces that the provided column(s) at
most have one instance of each possible combination of values in the synthetic data.

As an example, let us apply this constraint to the ``employee_id`` and ``company``
columns, since the ``employee_id`` should be unique for each ``company``.

To use this constraint, we must make an instance and provide:

- The name of the column or a list of names of columns that need to have unique values

.. ipython:: python
    :okwarning:

    from sdv.constraints import Unique

    unique_employee_id_company_constraint = Unique(
        columns=['employee_id', 'company']
    )

UniqueCombinations Constraint
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The next constraint that we will explore is the ``UniqueCombinations``
constraint.

This Constraint class can handle the situation number 1 indicated above,
in which the values of a set of columns can only be combined exactly as
seen in the original data, and new combinations are not accepted. In
order to use this constraint we will need to import it from the
``sdv.constraints`` module and create an instance of it indicating:

-  the names of the affected columns
-  which strategy we want to use: ``transform`` or ``reject_sampling``

.. ipython:: python
    :okwarning:

    from sdv.constraints import UniqueCombinations

    unique_company_department_constraint = UniqueCombinations(
        columns=['company', 'department'],
        handling_strategy='transform'
    )

GreaterThan Constraint
~~~~~~~~~~~~~~~~~~~~~~

The second constraint that we need for our data is the ``GreaterThan``
constraint. This constraint guarantees that one column is always greater
than the other one. In order to use it, we need to create an instance
passing:

-  the name of the ``low`` column
-  the name of the ``high`` column
-  the handling strategy that we want to use

.. ipython:: python
    :okwarning:

    from sdv.constraints import GreaterThan

    age_gt_age_when_joined_constraint = GreaterThan(
        low='age_when_joined',
        high='age',
        handling_strategy='reject_sampling'
    )

The ``GreaterThan`` constraint can also be used to guarantee a column is greater
or lower than a scalar value or specific datetime value instead of another column. 
To use this functionality, we can pass:

-  the scalar value for ``low`` or the scalar value for ``high``
-  a flag indicating whether ``low`` or ``high`` is a scalar

.. ipython:: python
    :okwarning:

    salary_gt_30000_constraint = GreaterThan(
        low=30000,
        high='salary',
        scalar='low',
        handling_strategy='reject_sampling'
    )

.. note::
    If you want to indicate that the column must be *lower than* a scalar value, 
    all you need to do is invert the arguments, pass the scalar value as the ``high`` 
    argument, the column name as the ``low`` argument, and set the `scalar` flag to ``"high"``.

Optionally, when constructing ``GreaterThan`` constraint we can specify 
more than a single column in either the ``high`` or ``low`` arguments. 
For example, we can create a ``GreaterThan`` constraint that that ensures 
that both the years in the company and prior years of experience is more 
than one year.

.. ipython:: python
    :okwarning:

    experience_years_gt_one_constraint = GreaterThan(
        low=1,
        high=['years_in_the_company', 'prior_years_experience'],
        scalar='low',
        handling_strategy='reject_sampling'
    )

.. warning::

    Warning! Passing a list of columns to the `high` or `low` arguments is only possible 
    when the other one has been passed as a single column name or scalar value! If you need 
    to compare multiple ``high`` columns against multiple ``low`` columns (or vice versa), 
    you need to decompose one of the ends, ``high`` or ``low``, into multiple single column
    names and define one ``GreaterThan`` constraint for each one of them.


Positive and Negative Constraints
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Similar to the ``GreaterThan`` constraint, we can use the ``Positive``
or ``Negative`` constraints. These constraints enforce that the specified
column(s) are always positive or negative. We can create an instance passing:

- the name of the column(s) for ``Negative`` or ``Positive`` constraints
- a boolean specifying whether to make the data strictly above or below 0, 
  or include 0 as a possible value
- the handling strategy that we want to use

.. ipython:: python
    :okwarning:

    from sdv.constraints import Positive

    positive_age_constraint = Positive(
        columns='age',
        strict=False,
        handling_strategy='reject_sampling'
    )

ColumnFormula Constraint
~~~~~~~~~~~~~~~~~~~~~~~~

In some cases, one column will need to be computed based on the other
columns using a custom formula. This is, for example, what happens with
the ``years_in_the_company`` column in our demo data, which will always
need to be computed based on the ``age`` and ``age_when_joined`` columns
by subtracting them. In these cases, we need to define a custom function
that defines how to compute the value of the column:

.. ipython:: python
    :okwarning:

    def years_in_the_company(data):
        return data['age'] - data['age_when_joined']

Once we have defined this function, we can use the ``ColumnFormula``
constraint by passing it:

-  the name of the column that we want to generate
-  the function that generates the column values
-  the handling strategy that we want to use

.. ipython:: python
    :okwarning:

    from sdv.constraints import ColumnFormula

    years_in_the_company_constraint = ColumnFormula(
        column='years_in_the_company',
        formula=years_in_the_company,
        handling_strategy='transform'
    )

Rounding Constraint
~~~~~~~~~~~~~~~~~~~

In order for data to be realistic, we also might want to round data
to a certain number of digits. To do this, we can use the Rounding
Constraint. We will pass this constraint:

-  the name of the column(s) that should be rounded.
-  the number of digits each column should be rounded to.
-  the handling strategy that we want to use
-  (optional) if reject sampling, we can customize the threshold of
   the sampled values.

.. ipython:: python
    :okwarning:

    from sdv.constraints import Rounding

    salary_rounding_constraint = Rounding(
        columns='salary',
        digits=2,
        handling_strategy='transform'
    )

Between Constraint
~~~~~~~~~~~~~~~~~~

Another possibility is the ``Between`` constraint. It guarantees
that one column is always in between two other columns/values. For example,
the ``age`` column in our demo data is realistically bounded to the ages of
15 and 90 since acual employees won't be too young or too old.

In order to use it, we need to create an instance passing:

-  the name of the ``low`` column or a scalar value to be used as the lower bound
-  the name of the ``high`` column or a scalar value to be used as the upper bound
-  the handling strategy that we want to use

.. ipython:: python
    :okwarning:
    
    from sdv.constraints import Between

    reasonable_age_constraint = Between(
        column='age',
        low=15,
        high=90,
        handling_strategy='transform'
    )

OneHotEncoding Constraint
~~~~~~~~~~~~~~~~~~~~~~~~~

Another constraint available is the ``OneHotEncoding`` constraint.
This constraint allows the user to specify a list of columns where each row 
is a one hot vector. Then, the constraint will make sure that the output
of the model is transformed so that the column with the largest value is
set to 1 while all other columns are set to 0. To apply the constraint we
need to create an instance passing:

- A list of the names of the columns of interest
- The strategy we want to use (``transform`` is recommended)

.. ipython:: python
    :okwarning:

    from sdv.constraints import OneHotEncoding

    one_hot_constraint = OneHotEncoding(
        columns=['full_time', 'part_time', 'contractor']
    )

Using the Constraints
---------------------

Now that we have defined the constraints needed to properly describe our
dataset, we can pass them to the Tabular Model of our choice. For
example, let us create a ``GaussianCopula`` model passing it the
constraints that we just defined as a ``list``:

.. ipython:: python
    :okwarning:

    from sdv.tabular import GaussianCopula

    constraints = [
        unique_employee_id_company_constraint,
        unique_company_department_constraint,
        age_gt_age_when_joined_constraint,
        salary_gt_30000_constraint,
        experience_years_gt_one_constraint,	
        positive_age_constraint,	
        years_in_the_company_constraint,	
        salary_rounding_constraint,	
        reasonable_age_constraint,	
        one_hot_constraint
    ]

    gc = GaussianCopula(constraints=constraints)

After creating the model, we can just fit and sample as usual:

.. ipython:: python
    :okwarning:

    gc.fit(employees)

    sampled = gc.sample(10)

And observe that the sampled rows really adjust to the constraints that
we defined:

.. ipython:: python
    :okwarning:

    sampled
