---
simd: '0016'
title: Application Fees
authors:
  - Godmode Galactus (Mango Markets)
  - Maximilian Schneider (Mango Markets)
category: Standard
type: Core, Fees
status: Draft
created: 2022-12-23
feature:
---


## Summary

This SIMD will discuss additional fees called Application Fees. These fees are
decided and set by the dapp developer to interact with the dapp. Dapp developers
then can decide to rebate these fees if a user interacts with the dapp as
intended and disincentivize the users which do not. These fees will be applied
even if the transaction eventually fails and are collected on the writable
account. The owner of the account can do lamport transfer to recover these
fees. So instead of fees going to the validator, these fees go to the **dapp
developers**. It will be dapp developer's responsibility to advertise the
required application fees to its users.

Discussion for the issue: <https://github.com/solana-labs/solana/issues/21883>

## Motivation

Currently, there is no way for dapp developers to enforce appropriate behavior
and the way their contracts should be used. Bots spamming on dapps make them
unusable and dapps lose their users/clients because the UX is laggy and
inefficient. Unsuccessful transactions deny block space to potentially valid
transactions which reduces activity on the dapp. For dapps like mango or
openbook increasing fees without any rebates or dynamically based other proposed
mechanisms will punish potentially thousands of users because of a handful of
malicious users. Giving dapp's authority more control to incentivize proper
utilization of its contract is the primary motivation for this proposal.

During network congestion, many transactions in the network cannot be processed
because they all want to write-lock the same accounts. When a write lock on an
account is taken by a transaction batch no other batch can use this account in
parallel, so only transactions belonging to a single batch are processed
correctly and all others are retried again or forwarded to the leader of the
next slot. With all the forwarding and retrying the validator is choked by the
transactions write-locking the same accounts and effectively processing valid
transactions sequentially.

There are multiple accounts of OpenBook (formerly serum), mango markets, etc
which are used in high-frequency trading. During the event of extreme
congestion, we observe that specialized trading programs write lock these
accounts but never CPI into the respective programs unless they can extract
profit, effectively starving the actual users for access. With current low
cluster fees, it incentivizes spammers to spam the network with transactions.

Without any proper rebate mechanism for users, Solana gas fees will increase and
it will lose the edge for being low fees cluster. For entities like market
makers, Solana must remain a low-fee cluster as they have a thin profit margin
and have to quote quite often. The goal of this proposal is to reduce spamming
without really increasing fees for everyone. Encourage users to create proper
transactions. Read the cluster state before creating transactions instead of
spamming. We are motivated to improve the user experience, and user activity and
keep Solana a low gas fee blockchain. Keeping gas fees low and increasing user
activity on the chain will help all the communities based on Solana grow. This
will also help users to gain more control over accounts owned by them.

Note that this proposal does not aim to increase gas fees for all transactions
or add any kind of congestion control mechanism.

## Alternatives Considered

* Having a fixed write lock fee and a read lock fee.

Pros: Simpler to implement, Simple to calculate fees.

Cons: Will increase fees for everyone, dapp cannot decide to rebate fees, (or
not all existing apps have to develop a rebate system). Fees cannot be decided
by the dapp developer. Penalizes everyone for a few bad actors.

* Passing application fees for each account by instruction and validating them
  during the execution

Pros: High efficiency, minimal performance impact, more generic, no need to
change the account structure.

Cons: Cannot prevent program denial of service attacks, or account blocking
attacks. Cannot implement disable read locking feature.

## New Terminology

Application Fees: Fees that are decided by the dapp developer will be charged if
a user wants to use the dapp. They will be applied even if the transaction fails
contrary to lamport transfers. The program can decide to rebate these fees back
to the user if certain conditions decided by dapp developers are met.

## Other terminology

`Owner` of an account: It is the account owner as specified in the `owner`
field in the account structure. In the case of an externally owned account
(i.e, keypair) owner is a `system program` and in case of a Program derived
address owner is usually the program.

Account `Authority`: The true authority for the account. Let's take an example
of associated token account. A associated token account is a `PDA` derived
from associated token program, where the owner is token program. But the
program internally saves who has the authority over the account. Only token
program can change the data of a token account. For operations like
withdrawing tokens, the authority of the associated token account has to be
the signer for the transaction containing transfer instruction. To receive
tokens from another account, the authority's signature is not required.

`Other Fees`: Fees other than application fees
`base fees + prioritization fees`.

## Detailed Design

As a high-performance cluster, we want to incentivize players which feed the
network with responsibly behaved transactions instead of spamming the network.
The introduction of application fees would be an interesting way to penalize
the bad actors and dapps can rebate these fees to the good actors. This means
the dapp developer has to decide who to rebate and who to penalize special
instructions. In particular, it needs to be able to penalize, even if the
application is not CPI'd to. There were multiple possible approaches discussed
to provide the application with access to transfer lamports outside of the
regular cpi execution context. The following approach seems the best.

*Updating core account structure to store application fees in ledger*.

A `PayApplicationFee` instruction is will be used by Solana runtime to calculate
how much total application fees are being paid by the transaction.

A `UpdateApplicationFees` mechanism will change the application fees for an
account.

A `Rebate` mechanism will be used to rebate the application fees back to the
payer.

1. Easy to calculate total fees for a transaction message.
2. Program authority can set application fees for the program's account.
3. Program can rebate fees to the payer.
4. Checks on payment of application fees are done even before executing the
   program. In case of nonpayment or partial payment, the program is never
   executed.
5. Could make account-blocking DOS attacks less viable.
6. Could be extended to protect read locking too.

The checks on the application fees will be taken care of by Solana runtime.
Total application fees paid will be included in the transaction so that it is
easier to calculate the total amount of fees that will be paid and less scope
for fraud. The maximum amount of application fees that can be set for an account
will be limited to a predecided number of SOLs recommended (100 SOLs) so that
account does not become inaccessible.

All other programs have to implement required instructions so that the
authority of the accounts can cyclically sign to update application fees on
the accounts they own.

Application fees should be considered as set and forget kind of fees by dapp
developers. They are not aimed to control congestion over the network instead
they aim to reduce spamming.

An additional proposal will be added later to address the read-locking of
accounts.

### A new application fee program

We add a new native Solana program called the application fees program with
program id.

```
App1icationFees1111111111111111111111111111
```

This native program will implement all the features required to implement
application fees.

### PayApplicationFees Instruction

It requires:

Accounts: None

Argument: Maximum application fees to be paid in lamports as `u64`

With this instruction, the fee payer accepts to pay application fees specifying
the maximum amount. This instruction **MUST** be included in the transaction
that interacts with dapps having application fees. This instruction is like an
infallible transfer if the payer has enough funds i.e, even if the transaction
fails, the payer will end up paying the required amount of application fees.
This instruction raises the minimum required balance to `other fees` +
`application fees`

Special cases:

- If the payer does not have enough balance, the transaction is not scheduled
  and fails with an error `InsufficientFunds`. Consider this case the same as if
  the payer does not have enough funds to pay the base fees.
- If the payer did not include this instruction in the transaction that
  write-locks accounts with application fees, then the transaction fails with an
  error `ApplicationFeesNotPaid`. the payer will pay just `other fees`. The
  failure is before the execution of the transaction.
- If the payer underpays the application fees, it will be handled the same as if
  application fees were not paid.
- If the payer overpays the application fees, then the transaction will be
  executed and the payer will be reimbursed the overpaid amount.

This instruction cannot be CPI'ed into.

### UpdateApplicationFees

This mechanism will set the application fees for an account in the `Account`
structure. The account must already exist and should be rent-free to change its
application fees. Only the account authority must be allowed to update the
application fees of an account. The application fee set will permanently update
the account in the ledger. This mechanism can be reused with fees set to 0 to
remove application fees on an account.

### Rebate

This mechanism should be called by the authority program to issue a rebate.
Rebate takes amount of lamports to be rebated as input. In case of multiple
rebates by the program during the execution, only the highest amount of rebate
will be taken into account. The rebated amount is always the minimum of rebate
issued by the program and the application fees on the account. If program
rebates `U64::MAX` it means all the application fees on the account are rebated.
Rebate amount cannot be 0.

### Looking at common cases

##### No application fees involved

* A payer does not include `PayApplicationFees` in the transaction. The
  transaction does not write lock any accounts with application fees. Then the
  transaction is executed without the application fee feature. The payer ends
  up paying other fees.

* A payer includes `PayApplicationFees(app fees)` in the transaction but none of
  the accounts have any application fees. This case is considered as overpay case.
  The payer balance is checked for `other fees + app fees`.
  - The payer does not have enough balance: Transaction fails with an error
    `Insufficient Balance` and the transaction is not even scheduled for
    execution.
  - The payer has enough balance then the transaction is executed and application
    fees paid are transferred back to the payer in any case.
  
  Note in this case
    even if there are no application fees involved the payer balance is checked
    for application fees.

##### Application fees are involved

* Fees not paid case:

  A payer does not include `PayApplicationFees` in the transaction. The
  transaction includes one or more accounts with application fees. Then the
  transaction is failed with an error `ApplicationFeesNotPaid`. The program is
  not executed at all. The payer ends up paying only other fees.

* Fees paid no rebates case:

  A payer includes instruction `PayApplicationFees(200)` in the transaction.
  There are accounts (`accA`, `accB`) that are write-locked by the transaction
  and each of them has an application fees of `100` lamports. Consider that the
  program does not have any rebate mechanism. Then in any case (execution fails
  or succeeds) `accA` will receive `100` lamports, `accB` will receive `100`
  lamports. The payer will end up paying `other fees` + `200` lamports.

* Fees paid full rebates case:

  A payer includes instruction `PayApplicationFees(200)` in the transaction.
  There are accounts (`accA`, `accB`) that are write-locked by the transaction
  and each of them has an application fees of `100` lamports. Consider during
  execution the program will rebate all the fees on both accounts. Then
  payer should have a minimum balance of `other fees` + `200` lamports to execute
  the transaction. After successful execution of the transaction, the `200`
  lamports will be rebated by the program and then Solana runtime will transfer
  them back to the payer. So the payer will finally end up paying `other fees`
  only.

* Fees paid multiple partial rebates case:

  A payer includes instruction `PayApplicationFees(200)` in the transaction. The
  transaction has three instructions (`Ix1`, `Ix2`, `Ix3`). There are accounts
  (`accA`, `accB`) that are write-locked by the transaction and each of them has
  an application fees of `100` lamports. Lets consider `Ix1` rebates 25 lamports
  on both accounts, `Ix2` rebates 75 lamports on `accA` and `Ix3` rebates 10
  lamports on `accB`. In case of multiple rebates only the maximum of all the
  rebates is applied. Consider the transaction is executed successfully. The
  maximum of all the rebates for `accA` is 75 lamports and `accB` is 25
  lamports. So total of 100 lamports are rebated back to the payer, `accA` gets
  25 lamports and `accB` get 75 lamports. The payer will end up paying
  `other fees` + 100 lamports.

* Fees paid and were over rebated case:

  A payer includes instruction `PayApplicationFees(200)` in the transaction.
  There are accounts (`accA`, `accB`) that are write-locked by the transaction
  and each of them has an application fees of `100` lamports. Consider during
  execution the program will rebate `1000` lamports on both accounts. Then payer
  should have minimum balance of `other fees` + `200` lamports to execute the
  transaction. After successful execution of the transaction, the
  `min`(application fees (100), 1000) = 100` lamports totalling to 200 lamports
  will be rebated by the program and then Solana runtime will transfer them back
  to the payer. So the payer will finally end up paying `other fees` only and
  total rebates are all application fees.

* Fees paid full rebates but the execution failed case:

  A payer includes instruction `PayApplicationFees(200)` in the transaction.
  There are accounts (`accA`, `accB`) that are write-locked by the transaction
  and each of them has an application fees of `100` lamports. Consider during
  execution the program will rebate all the fees on both accounts but later
  the execution failed. Then payer should have a minimum balance of `other fees`
  + `200` lamports to execute the transaction. The program rebated application
  fees but as executing the transaction failed, no rebate will be issued. The
  application fees will be transferred to respective accounts, and the payer
  will finally end up paying `other fees` + 200 lamports as application fees.

* Fees are over paid case:

  A payer includes instruction `PayApplicationFees(1000)` in the transaction.
  There are accounts (`accA`, `accB`) that are write-locked by the transaction
  and each of them has an application fees of `100` lamports. The minimum
  balance required by payer will be `other fees` + `1000` lamports as
  application fees. So payer pays 100 lamports per account as application fees
  and 800 lamports is an overpayment. The 800 lamports will be transferred back
  to the user even if transaction succeds or fails. The 100 lamports will be
  transferred to each account in all the cases except if the transaction is
  successful and the program issued a rebate.

* Fees underpaid case:

  A payer includes instruction `PayApplicationFees(150)` in the transaction.
  There are accounts (`accA`, `accB`, `accC`) that are write-locked by the
  transaction and each of them has an application fees of `100` lamports. Each
  account is mentioned in the transaction in the same order `accA`, `accB` and
  then `accC`. The minimum balance required by payer will be `other fees` +
  `150` lamports as application fees to load accounts and schedule transactions
  for execution. Here payer has insufficiently paid the application fees paying
  150 lamports instead of 300 lamports. So before program execution, we detect
  that the application fees is not sufficiently paid and execution fails with
  error `ApplicationFeesNotPaid` and the partially paid amount is transferred
  back to the payer. So payer pays only `base fees` in the end but the
  transaction is unsuccessful.


### Changes in the core Solana code

We have to update the account structure to store the application fees. This may
involve a lot of code changes in the core code of Solana. We will also have to
store the application fees on the ledger and get them back from the ledger when
we load the accounts.

When the cluster receives a new transaction, `PayApplicationFees` instruction is
decoded to calculate the total fees required for the transaction message. Then
we verify that the fee-payer has a minimum balance of:
`per-transaction base fees` + `prioritization fees` +
`maximum application fees to be paid`

If the payer has a sufficient balance then we continue loading other accounts.
If `PayApplicationFees` is missing then the application fee is 0 and we expect
there are no application fees involved on all the accounts that we have passed.
If the payer has insufficient balance transaction fails with an error
`Insufficient Balance`.

Before processing the message, we check if any loaded account has associated
application fees. For all accounts with application fees, the fees paid should
be greater than the required fees. In case of overpayment, the difference is
stored in a variable and will be paid back to the payer in any case. In case the
application fees are insufficiently paid or not paid, then we set the
transaction status as errored.

The structure `invoke context` is passed to all the native Solana program while
execution. We create a new structure called `ApplicationFeeData` which contains
one hashmap mapping application fees (`Pubkey` -> `application fees(u64)`), and
another containing rebates (`Pubkey` -> `amount rebated (u64)`). The
`ApplicationFeeData` structure will be filled by iterating over all the accounts
and inserting accounts with application fees into the `application fees` map. On
each `Rebate` call, we find the minimum between `application_fees` and
the rebate amount for the account. If there is already a rebate in the `rebated`
map then we update it by
`max(rebate amount in map, rebate amount issued)`, if the map does not
have any value for the account, then we add the rebated amount in the map.

In verify stage, we verify that `Application Fees` >= `Rebates` +
`Overpaid amount` for each account and return `UnbalancedInstruction` on
failure.

After the execution of the program there are the following cases:

The transaction was successfully executed:
    * Rebates aggregated + overpaid fees transferred back to the payer.
    * Remaining application fees transferred to the respective account.

2. Transaction executed with an error:
    * No rebates
    * Overpaid application fees are given back to the payer.
      Application fees are transferred to the respective account.

## Impact

Overall this feature will incentivize the creation of proper transactions and
spammers would have to pay much more fees reducing congestion in the cluster.
This will add very low calculation overhead on the validators. It will also
enable users to protect their accounts against malicious read and write locks.
This feature will encourage everyone to write better-quality code to help
avoid congestion.

It is the DApp's responsibility to publish the application fee required for each
account and instruction. They should also add appropriate `PayApplicationFee`
instruction in their client library while creating transactions or provide
visible API to get these application fees. We expect these fees to be set and
forget kind of fees and do not expect them to be changed frequently. Some
changes have to be done in web3.js client library to get application fees when
we request the account. Additional instructions should be added to the known
programs like Token Program, to enable this feature on the TokenAccounts. The
DApp developer have to also take into account application fees on the programs
they are dependent on.

The cluster is currently vulnerable to a different kind of attack where an
adversary with malicious intent can block its competitors by writing or
read-locking their accounts through a transaction. This attack involves
carrying out intensive calculations that consume a large number of
computational units, thereby blocking competitors from performing MEV during
that particular block, and giving the attacker an unfair advantage. We have
identified specific transactions that perpetrate this attack and waste
valuable block space. The malicious transaction write locks multiple token
accounts and consumes 11.7 million CU i.e around 1/4 the block space. As a
result, such attacks can prevent users from using their token accounts,
vote accounts, or stake accounts, and dapps from utilizing the required
accounts. With the proposed solution, every program, such as the token
program, stake program, and vote program, can include instructions to employ
the application fees feature on their accounts and rebate the fees if the user
initiates the transaction. The attacker will find this option unfeasible as
they will consume their SOL tokens more rapidly to maintain the attack.

## Security Considerations

If the application fee for an account is set too high then we cannot ever
mutate that account anymore. Even updating the application fees for the
account will need a very high amount of balance. This issue can be easily
solved by setting a maximum limit to the application fees.

For an account that has collected application fees, to transfer these fees
collected to another account we have to pay application fees to write lock the
account, we can include a rebate in the transaction. In case of
any bugs, while transferring application fees from the account to the
authority, there can be an endless loop where the authority creates a
transaction to recover collected application fees, with an instruction to pay
application fees to modify the account and an instruction to rebate. If the
transaction fails because of the bug, the user fails to recover collected
fees, in turn increasing application fees collected on the account.

## Backwards Compatibility

This feature does not introduce any breaking changes. The transaction without
using this feature should work as it is. To use this feature supermajority of
the validators should move to a branch that implements this feature.
Validators that do not implement this feature cannot replay the blocks with
transactions using application fees they also could not validate the block
including transactions with application fees.

## Mango V4 Usecase

With this feature implemented Mango-V4 will be able to charge users who spam
risk-free arbitrage or spam liquidations by increasing application fees on
perp-markets, token banks and mango-user accounts.

#### Perp markets

Application fees on perp liquidations, perp place order, perp cancel, perp
consume, perp settle fees. Rebates on: successful liquidations, consume
events, HFT market making refresh (cancel all, N* place POST, no IOC).

#### Token vaults

Application fees on open order book liquidations, deposit, withdrawals. Rebate
on successful liquidations, place IOC & fill in isolation, HFT marketmaking
refresh (cancel all, N* place POST, no IOC).

#### Mango accounts

Application fees on all liquidity transactions, consume events, settle pnl,
all user signed transactions. Rebate on transaction signed by owner or
delegate, successful liquidations, settlements, consume events.

## Additional Notes

If the dapps want to rebate application fees they have to implement very
carefully the logic of rebate. They should be very meticulous before calling
rebate so that a malicious user could not use this feature to bypass
application fees. Dapp developers also have to implement additional
instruction to collect these fees using lamport transfers.

DApp developers have to consider the following way to bypass application fees
is possible: A Defi smart contract with two instructions IxA and IxB. Both IxA
and IxB issue a rebate. IxA is an instruction that places an order on the
market which can be used to extract profit. IxB is an instruction that just
does some bookkeeping like settling funds overall harmless instruction.
Malicious users then can create a custom smart contract to bypass the
application fees where it CPI's IxA only if they can extract profit or else
they use IxB to issue a rebate for the application fees. So DApp developers
have to be sure when to do rebates usually white listing and black listing
instruction sequence would be ideal.

Dapp should be careful before rolling out this feature. Because the
transaction would start failing if the rollout is sudden. It is preferable to
implement rebate, and add pay application fees in their APIs, so that the user
pays full application fee but is then rebated if the transaction succeeds.
Then once everyone starts using the new API they can add the check on
application fees.

Another proposal will also introduce protection against unwanted read-locking
of the accounts. Many accounts like token account rarely need to be read-locked
this proposal will force these accounts to be write-locked instead and pay
application fees if needed. This feature is out of the scope of this proposal.

### Calculating Application Fees for a dapp

Let us consider setting application fees for Openbook DEX. We can set fees
comparable to the rent of the accounts involved or something fixed. Setting
application fees too high means dapp users need more balance to interact with
the dapps and if they are too low then it won't prevent spamming or malicious
use. In case of openbook the intent is to avoid spamming.

Most of the OpenBook accounts like asks, bids and event queues are used in
write mode only we can disable read-locks on these accounts. Now we can
consider there are 48 M CU's per block and 2.5 blocks per second. Considering
each instruction takes 200 CUs so around 600 transactions per second.
Currently, base fees are around 0.00005 SOLs, with so low gas fees spammers
have the advantage to spam the cluster. A reasonable application fee could be
around 0.1 SOLs per transaction that could be distributed among different
accounts. For a user interacting with dapp with 0.1 SOLs in the account seems
reasonable assuming that the transactions are executed successfully and the
fees are rebated. This will make spammers spend their SOLs 2000 times more
rapidly than before. The thumb rule for dapps to set application fees on their
accounts is `More important the account = Higher application fees`.

Suppose a user or entity desires to safeguard their token account from
potentially malicious read/write locking that obstructs their ability to
perform MEV. In that case, they can set the highest possible application fees
on their token account, rendering it impossible to extract profit by blocking
their account. Even if the transaction fails, and they have to pay the
application fees, they can recover the fees as they own the account. A general
guideline for users is that they should possess at least N times (where N=10
is recommended) the SOLs required to pay application fees on their accounts so
that they are not locked out. The user has to pay application fees to transfer
all collected application fees.
