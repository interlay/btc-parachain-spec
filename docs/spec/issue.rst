.. _issue-protocol:

Issue
=====

Overview
~~~~~~~~

The Issue module allows as user to create new interBTC tokens. The user needs to request interBTC through the :ref:`requestIssue` function, then send BTC to a Vault, and finally complete the issuing of interBTC by calling the :ref:`executeIssue` function. If the user does not complete the process in time, the Vault can cancel the issue request and receive a griefing collateral from the user by invoking the :ref:`cancelIssue` function. Below is a high-level step-by-step description of the protocol.

Step-by-step
------------

The nominal control flow is as follows:

1. Precondition: a Vault has locked collateral as described in the :ref:`Vault-registry`.
2. A user executes the :ref:`requestIssue` function to open an issue request. The issue request includes the amount of interBTC the user wants to issue, the selected Vault, and a small collateral reserve to prevent :ref:`griefing`.
3. A user sends the equivalent amount of BTC to issue as interBTC to the Vault on the Bitcoin blockchain. 
4. The user or Vault acting on behalf of the user extracts a transaction inclusion proof of that locking transaction on the Bitcoin blockchain. The user or a Vault acting on behalf of the user executes the :ref:`executeIssue` function on the BTC Parachain. The issue function requires a reference to the issue request and the transaction inclusion proof of the Bitcoin locking transaction. If the function completes successfully, the user receives the requested amount of interBTC into his account.
5. Optional: If the user is not able to complete the issue request within the predetermined time frame (``IssuePeriod``), the Vault is able to call the :ref:`cancelIssue` function to cancel the issue request adn will receive the griefing collateral locked by the user.

User Failsafe
.............

To accommodate for user error, the bridge allows the execution of issue requests even when the user sends an incorrect BTC amount. Specifically, we distinguish the following cases:

* The user sends less than the expected amount. The user has the option to execute the issue with this amount. However, it will lose part of its griefing collateral. If it sends e.g. 10% of the expected amount, it loses 90% of the griefing collateral. It will also receive 10% of the wrapped tokens. Because there is a cost associated with this choice, automatic execution of this issue request by Vaults is disallowed. The alternative for the user is to make another Bitcoin transfer, and to execute the issue with that transaction. In this case, however, it loses the Bitcoin sent in the first transaction.
* The user sends more than the expected amount.

  * If the Vault has sufficient collateral to issue wrapped tokens for the sent amount, the size of the issue request is automatically increased and more collateral of the Vault is reserved. The user receives the amount corresponding to the received amount of Bitcoin. The issue fee is deducted from the updated (increased) amount.
  * If the Vault does not have sufficient collateral to issue the additional amount, only the amount that was originally requested is issued. A refund request is sent to the Vault to return the surplus Bitcoin (excluding a fee). Note, however, that there is no penalty for the Vault if it does not return the surplus Bitcoin since this is a user fault rather than a Vault fault.

Security
--------

- Unique identification of Bitcoin payments: :ref:`okd`

Vault Registry
--------------

The data access and state changes to the Vault registry are documented in :numref:`fig-vault-registry-issue` below.

.. _fig-vault-registry-issue:
.. figure:: ../figures/spec/VaultRegistry-Issue.png
    :alt: Vault-Registry Issue

    The issue protocol interacts with three functions in the :ref:`vault-registry` that handle updating the different token balances.

Fee Model
---------

- Issue fees are paid by users in interBTC when executing the request. The fees are transferred to the Parachain Fee Pool.
- If an issue request is executed, the user’s griefing collateral is returned.
- If an issue request is canceled, the Vault assigned to this issue request receives the griefing collateral.

Data Model
~~~~~~~~~~

Scalars
-------

.. _issuePeriod:

IssuePeriod
............

The time difference between when an issue request is created and required completion time by a user.
Concretely, this period is the amount by which :ref:`activeBlockCount` is allowed to increase before the issue is considered to be expired.
The period has an upper limit to prevent griefing of Vault collateral.

.. _issueBtcDustValue:

IssueBtcDustValue
.................

The minimum amount of BTC that is required for issue requests; lower values would risk the rejection of payment on Bitcoin.

Maps
----

.. _issueRequests:

IssueRequests
.............

Users create issue requests to issue interBTC. This mapping provides access from a unique hash ``IssueId`` to a ``Issue`` struct. ``<IssueId, IssueRequest>``.

Structs
-------

IssueRequest
............

Stores the status and information about a single issue request.

.. tabularcolumns:: |l|l|L|

======================  ============  =======================================================	
Parameter               Type          Description                                            
======================  ============  =======================================================
``vault``               AccountId     The address of the Vault responsible for this issue request.
``opentime``            BlockNumber   The :ref:`activeBlockCount` when the issue request was created.
``period``              BlockNumber   Value of the :ref:`issuePeriod` when the request was made.
``griefingCollateral``  DOT           Security deposit provided by a user.
``amount``              interBTC      Amount of interBTC to be issued.
``fee``                 interBTC      Fee charged to the user for issuing.
``requester``           AccountId     User account receiving interBTC upon successful issuing.
``btcAddress``          BtcAddress    Vault's P2WPKH Bitcoin deposit address.
``btcPublicKey``        BtcPublicKey  Vault's Bitcoin public key used to generate the deposit address.
``btcHeight``           u32           The highest recorded height of the relay at time of opening.
``status``              Enum          Status of the request: Pending, Completed or Cancelled.
======================  ============  =======================================================

Functions
~~~~~~~~~

.. _requestIssue:

requestIssue
------------

A user opens an issue request to create a specific amount of interBTC. 
When calling this function, a user provides their parachain account identifier, the to be issued amount of interBTC, and the Vault to use in this process (account identifier). Further, they provide some (small) amount of DOT collateral (``griefingCollateral``) to prevent griefing.

Specification
.............

*Function Signature*

``requestIssue(requester, amount, vault, griefingCollateral)``

*Parameters*

* ``requester``: The user's account identifier.
* ``amount``: The amount of interBTC to be issued.
* ``vault``: The address of the Vault involved in this issue request.
* ``griefingCollateral``: The collateral amount provided by the user as griefing protection.

*Events*

* :ref:`requestIssueEvent`

*Preconditions*

* The function call MUST be signed by ``requester``.
* The BTC Parachain status in the :ref:`security` component MUST NOT be ``SHUTDOWN:2``.
* The :ref:`btc_relay` MUST be initialized.
* The Vault MUST be registered and active.
* The Vault MUST NOT be banned.
* The ``amount`` MUST be greater than or equal to :ref:`issueBtcDustValue`.
* The ``griefingCollateral`` MUST exceed or equal the value of request ``amount`` at the current exchange-rate, multiplied by :ref:`issueGriefingCollateral`.
* The ``griefingCollateral`` MUST be equal or less than the requester's free balance in the :ref:`griefingCurrency`.
* The :ref:`tryIncreaseToBeIssuedTokens` function MUST return a new BTC deposit address for the Vault ensuring that the Vault's free collateral is above the :ref:`SecureCollateralThreshold` for the requested ``amount`` and that a unique BTC address is used for depositing BTC.
* A new unique ``issuedId`` MUST be generated via the :ref:`generateSecureId` function.

*Postconditions*

* The Vault's ``toBeIssuedTokens`` MUST increase by ``amount``.
* The requester's free balance in the :ref:`griefingCurrency` MUST decrease by ``griefingCollateral``.
* The requester's locked balance in the :ref:`griefingCurrency` MUST increase by ``griefingCollateral``.
* A new BTC deposit address for the Vault MUST be generated by the :ref:`tryIncreaseToBeIssuedTokens`.
* The new issue request MUST be created as follows:

    * ``issue.vault``: MUST be the ``vault``.
    * ``issue.opentime``: MUST be the :ref:`activeBlockCount` of the current block of this transaction.
    * ``issue.period``: MUST be the current :ref:`issuePeriod`.
    * ``issue.griefingCollateral``: MUST be the ``griefingCollateral`` amount passed to the function.
    * ``issue.amount``: MUST be ``amount`` minus ``issue.fee``.
    * ``issue.fee``: MUST equal ``amount`` multiplied by :ref:`issueFee`.
    * ``issue.requester``: MUST be the ``requester``
    * ``issue.btcAddress``: MUST be the BTC address returned from the :ref:`tryIncreaseToBeIssuedTokens`
    * ``issue.btcPublicKey``: MUST be the BTC public key returned from the :ref:`tryIncreaseToBeIssuedTokens`
    * ``issue.btcHeight``: MUST be the current Bitcoin height as stored in the BTC-Relay.
    * ``issue.status``: MUST be ``Pending``.

* The new issue request MUST be inserted into :ref:`issueRequests` using the generated ``issueId`` as the key.

.. _executeIssue:

executeIssue
------------

An executor completes the issue request by sending a proof of transferring the defined amount of BTC to the vault's address.

Specification
.............

*Function Signature*

``executeIssue(executorId, issueId, rawMerkleProof, rawTx)``

*Parameters*

* ``executor``: the account of the user.
* ``issueId``: the unique hash created during the ``requestIssue`` function.
* ``rawMerkleProof``: Raw Merkle tree path (concatenated LE SHA256 hashes).
* ``rawTx``: Raw Bitcoin transaction including the transaction inputs and outputs.

*Events*

* :ref:`executeIssueEvent`
* If the amount transferred IS not equal to the ``issue.amount + issue.fee``, the :ref:`issueAmountChangeEvent` MUST be emitted

*Preconditions*

* The function call MUST be signed by ``executor``.
* The BTC Parachain status in the :ref:`security` component MUST NOT be ``SHUTDOWN:2``.
* The issue request for ``issueId`` MUST exist in :ref:`issueRequests`.
* The issue request for ``issueId`` MUST NOT have expired.
* The ``rawTx`` MUST be valid and contain a payment to the Vault.
* The ``rawMerkleProof`` MUST be valid and prove inclusion to the main chain.
* If the amount transferred is less than ``issue.amount + issue.fee``, then the ``executor`` MUST be the account that made the issue request.

*Postconditions*

* If the amount transferred IS less than the ``issue.amount + issue.fee``:

    * The Vault's ``toBeIssuedTokens`` MUST decrease by the deficit (``issue.amount - amountTransferred``).
    * The Vault's free balance in the :ref:`griefingCurrency` MUST increase by the ``griefingCollateral * (1 - amountTransferred / (issue.amount + issue.fee))``.
    * The requester's free balance in the :ref:`griefingCurrency` MUST increase by the ``griefingCollateral * amountTransferred / (issue.amount + issue.fee)``.
    * The ``issue.fee`` MUST be updated to the amount transferred multiplied by the :ref:`issueFee`.
    * The ``issue.amount`` MUST be set to the amount transferred minus the updated ``issue.fee``.

* If the amount transferred IS NOT less than the expected amount:

    * The requester's free balance in the :ref:`griefingCurrency` MUST increase by the ``griefingCollateral``.
    * If the amount transferred IS greater than the expected amount:

        * If the Vault IS NOT liquidated and has sufficient collateral:

            * The Vault's ``toBeIssuedTokens`` MUST increase by the surplus (``amountTransferred - issue.amount``).
            * The ``issue.fee`` MUST be updated to the amount transferred multiplied by the :ref:`issueFee`.
            * The ``issue.amount`` MUST be set to the amount transferred minus the updated ``issue.fee``.

        * If the Vault IS NOT liquidated and does not have sufficient collateral:

            * There MUST exist a :ref:`refund-protocol` request which references ``issueId``.

* The requester's locked balance in the :ref:`griefingCurrency` MUST decrease by ``issue.griefingCollateral``.
* The ``issue.status`` MUST be set to ``Completed``.
* The Vault's ``toBeIssuedTokens`` MUST decrease by ``issue.amount + issue.fee``.
* The Vault's ``issuedTokens`` MUST increase by ``issue.amount + issue.fee``.
* The user MUST receive ``issue.amount`` interBTC in its free balance.
* Function :ref:`reward_distributeReward` MUST complete successfully  - parameterized by ``issue.fee``.

.. _cancelIssue:

cancelIssue
-----------

If an issue request is not completed on time, the issue request can be cancelled.

Specification
.............

*Function Signature*

``cancelIssue(requester, issueId)``

*Parameters*

* ``requester``: The sender of the cancel transaction.
* ``issueId``: the unique hash of the issue request.

*Events*

* :ref:`cancelIssueEvent`

*Preconditions*

* The function call MUST be signed by ``requester``.
* The BTC Parachain status in the :ref:`security` component MUST NOT be ``SHUTDOWN:2``.
* The issue request for ``issueId`` MUST exist in :ref:`issueRequests`.
* The issue request MUST have expired.

*Postconditions*

* If the vault IS liquidated:

    * The requester's free balance oinf the :ref:`griefingCurrency` MUST increase by the ``griefingCollateral``.

* If the Vault IS NOT liquidated:

    * The vault's free balance in the :ref:`griefingCurrency` MUST increase by the ``griefingCollateral``.

* The requester's locked balance in the :ref:`griefingCurrency` MUST decrease by the ``griefingCollateral``.
* The vault's ``toBeIssuedTokens`` MUST decrease by ``issue.amount + issue.fee``.
* The issue status MUST be set to ``Cancelled``.


Events
~~~~~~

.. _requestIssueEvent:

RequestIssue
------------

Emit an event if a user successfully open a issue request.

*Event Signature*

``RequestIssue(issueId, requester, amount, fee, griefingCollateral, vault, btcAddress, btcPublicKey)``

*Parameters*

* ``issueId``: A unique hash identifying the issue request. 
* ``requester``: The user's account identifier.
* ``amount``: The amount of interBTC requested.
* ``fee``: The amount of interBTC to mint as fees.
* ``griefingCollateral``: The security deposit provided by the user.
* ``vault``: The address of the Vault involved in this issue request.
* ``btcAddress``: The Bitcoin address of the Vault.
* ``btcPublicKey``: The Bitcoin public key of the Vault.

*Functions*

* :ref:`requestIssue`

.. _issueAmountChangeEvent:

IssueAmountChange
-----------------

Emit an event if the issue amount changed for any reason.

*Event Signature*

``IssueAmountChange(issueId, amount, fee, griefingCollateral)``

*Parameters*

* ``issueId``: A unique hash identifying the issue request. 
* ``amount``: The amount of interBTC requested.
* ``fee``: The amount of interBTC to mint as fees.
* ``griefingCollateral``: Confiscated griefing collateral.

*Functions*

* :ref:`executeIssue`

.. _executeIssueEvent:

ExecuteIssue
------------

*Event Signature*

``ExecuteIssue(issueId, requester, amount, vault, fee)``

*Parameters*

* ``issueId``: A unique hash identifying the issue request. 
* ``requester``: The user's account identifier.
* ``amount``: The amount of interBTC issued to the user.
* ``vault``: The address of the Vault involved in this issue request.
* ``fee``: The amount of interBTC minted as fees.

*Functions*

* :ref:`executeIssue`

.. _cancelIssueEvent:

CancelIssue
-----------

*Event Signature*

``CancelIssue(issueId, requester, griefingCollateral)``

*Parameters*

* ``issueId``: the unique hash of the issue request.
* ``requester``: The sender of the cancel transaction.
* ``griefingCollateral``: The released griefing collateral.

*Functions*

* :ref:`cancelIssue`

Error Codes
~~~~~~~~~~~

``ERR_VAULT_NOT_FOUND``

* **Message**: "There exists no Vault with the given account id."
* **Function**: :ref:`requestIssue`
* **Cause**: The specified Vault does not exist.

``ERR_VAULT_BANNED``

* **Message**: "The selected Vault has been temporarily banned."
* **Function**: :ref:`requestIssue`
* **Cause**:  Issue requests are not possible with temporarily banned Vaults

``ERR_INSUFFICIENT_COLLATERAL``

* **Message**: "User provided collateral below limit."
* **Function**: :ref:`requestIssue`
* **Cause**: User provided griefingCollateral below :ref:`issueGriefingCollateral`.

``ERR_UNAUTHORIZED_USER``

* **Message**: "Unauthorized: Caller must be associated user"
* **Function**: :ref:`executeIssue`
* **Cause**: The caller of this function is not the associated user, and hence not authorized to take this action.

``ERR_ISSUE_ID_NOT_FOUND``

* **Message**: "Requested issue id not found."
* **Function**: :ref:`executeIssue`
* **Cause**: Issue id not found in the ``IssueRequests`` mapping.

``ERR_COMMIT_PERIOD_EXPIRED``

* **Message**: "Time to issue interBTC expired."
* **Function**: :ref:`executeIssue`
* **Cause**: The user did not complete the issue request within the block time limit defined by the ``IssuePeriod``.

``ERR_TIME_NOT_EXPIRED``

* **Message**: "Time to issue interBTC not yet expired."
* **Function**: :ref:`cancelIssue`
* **Cause**: Raises an error if the time limit to call ``executeIssue`` has not yet passed.

``ERR_ISSUE_COMPLETED``

* **Message**: "Issue completed and cannot be cancelled."
* **Function**: :ref:`cancelIssue`
* **Cause**: Raises an error if the issue is already completed.
