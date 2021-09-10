.. _currency:

Currency
========

Overview
~~~~~~~~

This currency pallet provides an interface for the other pallets to manage balances of different currencies. It is a wrapper around the the `orml-tokens <https://github.com/open-web3-stack/open-runtime-module-library>`_ pallet. As such, accounts have two balances per currency: they have a ``reserved`` amount and a ``free`` amount. Users are able to freely transfer ``free`` balances, but only the parachain pallets are able to operate on ``reserved`` amounts.

The external API for dispatchable and RPC functions use 'thin' amount types, meaning that the used currency depends on the context. For example, the currency used in :ref:`depositCollateral` depends on the vault's ``currencyId``. Sometimes, as is for example the case for :ref:`registerVault`, the function takes an additional ``currencyId`` argument to specify the currency to use. In contrast, internally in the parachain amounts are often represented by the ``Amount`` type defined in this pallet, which in addition to the amount, also contains the used currency. The benefit of this type is two-fold. First, we can guarantee that operations only work on compatible amounts. For example, it prevents adding DOT amounts to KSM amounts. Second, it allows for a more convenient api.

Data Model
~~~~~~~~~~

Structs
-------

Amount
......

Stores an amount and the used currency.

.. tabularcolumns:: |l|l|L|

======================  ==========  =======================================================	
Parameter               Type        Description                                            
======================  ==========  =======================================================
``amount``              Balance     The amount.
``currency``            CurrencyId  The used currency.
======================  ==========  =======================================================


Functions
~~~~~~~~~

.. _fromSignedFixedPoint:

fromSignedFixedPoint
--------------------

Constructs an ``Amount`` from a signed fixed point number and a ``currencyId``. The fixed point number is truncated. E.g., a value of 2.5 would return 2. 

Specification
.............

*Function Signature*

``fromSignedFixedPoint(amount, currencyId)``

*Parameters*

* ``amount``: The amount as fixed point.
* ``currencyId``: The currency.

*Preconditions*

* ``amount`` MUST be representable as a 128 bit unsigned number.

*Postconditions*

* An ``Amount`` MUST be returned where ``Amount.amount`` is the truncated ``amount`` argument, and ``Amount.currencyId`` is the ``currencyId`` argument.


.. _toSignedFixedPoint:

toSignedFixedPoint
------------------

Converts an ``Amount`` struct into a fixed-point number.

Specification
.............

*Function Signature*

``toSignedFixedPoint(amount)``

*Parameters*

* ``amount``: The amount struct.

*Preconditions*

* ``amount`` MUST be representable by the signed fixed point type.

*Postconditions*

* ``amount.amount`` MUST be returned as a fixed point number.


.. _convertTo:

convertTo
---------

Converts the given ``amount`` into the given currency. 

Specification
.............

*Function Signature*

``convertTo(amount, currencyId)``

*Parameters*

* ``amount``: The amount struct.
* ``currencyId``: The currency to convert to.

*Preconditions*

* :ref:`convert` when called with ``amount`` and ``currencyId`` MUST return successfully.

*Postconditions*

* :ref:`convert` MUST be called with ``amount`` and ``currencyId`` as arguments.


.. _checkedAdd:

checkedAdd
----------

Adds two amounts.

Specification
.............

*Function Signature*

``checkedAdd(amount1, amount2)``

*Parameters*

* ``amount1``: the first amount.
* ``amount2``: the second amount.

*Preconditions*

* ``amount1.currencyId`` MUST be equal to ``amount2.currencyId``

*Postconditions*

* MUST return the sum of both amounts.



.. _checkedSub:

checkedSub
----------

Subtracts two amounts.

Specification
.............

*Function Signature*

``checkedSub(amount1, amount2)``

*Parameters*

* ``amount1``: the first amount.
* ``amount2``: the second amount.

*Preconditions*

* ``amount1.currencyId`` MUST be equal to ``amount2.currencyId``

*Postconditions*

* MUST return ``amount1 - amount2``.


.. _saturatingSub:

saturatingSub
-------------

Subtracts two amounts, or zero if the result would be negative.

Specification
.............

*Function Signature*

``saturatingSub(amount1, amount2)``

*Parameters*

* ``amount1``: the first amount.
* ``amount2``: the second amount.

*Preconditions*

* ``amount1.currencyId`` MUST be equal to ``amount2.currencyId``

*Postconditions*

* if ``amount2 <= amount1``, then this function MUST return ``amount1 - amount2``.
* if ``amount2 > amount1``, then this function MUST return zero.


.. _checkedFixedPointMul:

checkedFixedPointMul
--------------------

Multiplies an amount by a fixed point scalar. The result is rounded down.

Specification
.............

*Function Signature*

``checkedFixedPointMul(amount, scalar)``

*Parameters*

* ``amount``: the Amount struct.
* ``scalar``: the fixed point scalar.

*Preconditions*

* The multiplied amount MUST be representable by a 128 bit unsigned integer.

*Postconditions*

* MUST return a copy of ``amount`` that is multiplied by the scalar. The result MUST be rounded down.


.. _checkedFixedPointMulRoundedUp:

checkedFixedPointMulRoundedUp
-----------------------------

Like :ref:`checkedFixedPointMul`, but with a rounded-up result.

Specification
.............

*Function Signature*

``checkedFixedPointMulRoundedUp(amount, scalar)``

*Parameters*

* ``amount``: the Amount struct.
* ``scalar``: the fixed point scalar.

*Preconditions*

* The multiplied amount MUST be representable by a 128 bit unsigned integer.

*Postconditions*

* MUST return a copy of ``amount`` that is multiplied by the scalar. The result MUST be rounded up.


.. _roundedMul:

roundedMul
----------

Like :ref:`checkedFixedPointMul`, but with a rounded result.

Specification
.............

*Function Signature*

``roundedMul(amount, scalar)``

*Parameters*

* ``amount``: the Amount struct.
* ``scalar``: the fixed point scalar.

*Preconditions*

* The multiplied amount MUST be representable by a 128 bit unsigned integer.

*Postconditions*

* MUST return a copy of ``amount`` that is multiplied by the scalar. The result MUST be rounded to the nearest integer.


.. _checkedDiv:

checkedDiv
----------

Divides an amount by a fixed point scalar. The result is rounded down.

Specification
.............

*Function Signature*

``checkedDiv(amount, scalar)``

*Parameters*

* ``amount``: the Amount struct.
* ``scalar``: the fixed point scalar.

*Preconditions*

* The multiplied amount MUST be representable by a 128 bit unsigned integer.

*Postconditions*

* MUST return a copy of ``amount`` that is divided by the scalar.


.. _ratio:

ratio
-----

Returns the fixed point ratio between two amounts.

Specification
.............

*Function Signature*

``ratio(amount1, amount2)``

*Parameters*

* ``amount1``: the first Amount struct.
* ``amount2``: the second Amount struct.

*Preconditions*

* ``amount1.currencyId`` MUST be equal to ``amount2.currencyId``
* The ratio MUST be representable by the fixed point type.

*Postconditions*

* MUST return the ratio between the two amounts.


.. _cmp:

Comparisons: lt, le, eq, ge, gt
-------------------------------

Compares two amounts

Specification
.............

*Function Signature*

``[lt|le|eq|ge|gt](amount1, amount2)``

*Parameters*

* ``amount1``: the first Amount struct.
* ``amount2``: the second Amount struct.

*Preconditions*

* ``amount1.currencyId`` MUST be equal to ``amount2.currencyId``

*Postconditions*

* MUST return true when the comparison holds.



.. _transfer:

transfer
--------

Transfers the amount between the given accounts.

Specification
.............

*Function Signature*

``transfer(amount, source, destination)``

*Parameters*

* ``amount``: the Amount struct.
* ``source``: the account to transfer from.
* ``destination``: the account to transfer to.

*Preconditions*

* ``source`` MUST have sufficient unlocked funds in the given currency 

*Postconditions*

* The free balance of ``source`` MUST decrease by ``amount.amount`` (in the currency determined by ``amount.currencyId)``
* The free balance of ``destination`` MUST increase by ``amount.amount`` (in the currency determined by ``amount.currencyId)``



.. _lockOn:

lockOn
------

Locks the amount on the given account.

Specification
.............

*Function Signature*

``lockOn(amount, accountId)``

*Parameters*

* ``amount``: the Amount struct.
* ``accountId``: the account to lock the amount on.

*Preconditions*

* The given account MUST have sufficient unlocked funds in the given currency.

*Postconditions*

* The free balance of ``accountId`` MUST decrease by ``amount.amount`` (in the currency determined by ``amount.currencyId)``
* The locked balance of ``accountId`` MUST increase by ``amount.amount`` (in the currency determined by ``amount.currencyId)``



.. _unlockOn:

unlockOn
--------

Unlocks the amount on the given account.

Specification
.............

*Function Signature*

``unlockOn(amount, accountId)``

*Parameters*

* ``amount``: the Amount struct.
* ``accountId``: the account to unlock the amount on.

*Preconditions*

* The given account MUST have sufficient locked funds in the given currency.

*Postconditions*

* The locked balance of ``accountId`` MUST decrease by ``amount.amount`` (in the currency determined by ``amount.currencyId)``
* The free balance of ``accountId`` MUST increase by ``amount.amount`` (in the currency determined by ``amount.currencyId)``


.. _burnFrom:

burnFrom
--------

Burns the amount on the given account.

Specification
.............

*Function Signature*

``burnFrom(amount, accountId)``

*Parameters*

* ``amount``: the Amount struct.
* ``accountId``: the account to lock the amount on.

*Preconditions*

* The given account MUST have sufficient locked funds in the given currency.

*Postconditions*

* The locked balance of ``accountId`` MUST decrease by ``amount.amount`` (in the currency determined by ``amount.currencyId)``


.. _mintTo:

mintTo
------

Mints the amount on the given account.

Specification
.............

*Function Signature*

``mintTo(amount, accountId)``

*Parameters*

* ``amount``: the Amount struct.
* ``accountId``: the account to mint the amount on.

*Postconditions*

* The unlocked balance of ``accountId`` MUST increase by ``amount.amount`` (in the currency determined by ``amount.currencyId)``