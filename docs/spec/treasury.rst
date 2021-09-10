.. _treasury-module:

Treasury
========

Overview
~~~~~~~~

Conceptually, the treasury serves as both the central storage for all interBTC and the interface though which to manage interBTC amount. It is implemented through the :ref:`currency` pallet.

There are three main operations on interBTC to interact with the user or the :ref:`issue-protocol` and :ref:`redeem-protocol` components. 

Step-by-step
------------

* **Transfer**: A user sends an amount of interBTC to another user by calling the :ref:`transfer` function.
* **Issue**: The issue module calls into the treasury when an issue request is completed (via :ref:`executeIssue`) and the user has provided a valid proof that the required amount of BTC was sent to the correct vault. The issue module calls the :ref:`mint` function to create interBTC.
* **Redeem**: The redeem protocol requires two calls to the treasury module. First, a user requests a redeem via the :ref:`requestRedeem` function. This invokes a call to the :ref:`lock` function that locks the requested amount of tokens for this user. Second, when a redeem request is completed (via :ref:`executeRedeem`) and the vault has provided a valid proof that it transferred the required amount of BTC to the correct user, the redeem module calls the :ref:`burn` function to destroy the previously locked interBTC.
