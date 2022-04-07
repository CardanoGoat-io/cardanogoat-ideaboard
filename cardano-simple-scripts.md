# Cardano Simple Scripts

The Shelley era supports a single, simple script language, which can be
used for multi-signature addresses. The Allegra era (token locking) extends the simple script
language with a feature to make scripts conditional on time. This can be used
to make address with so-called "time locks", where the funds cannot be
withdrawn until after a certain point in time.

The Alonzo era will add support for a whole new script language:
Plutus core. (smart contracts) 

Please note that this reference covers only the existing simple script language.

## Script addresses

In general, addresses (both payment addresses and stake(reward) addresses) specify the
_authorization conditions_ that must be met for the address to be used. For
payment addresses, this means the authorization conditions for funds to be
withdrawn. For stake addresses, this means the authorization conditions for
setting a delegation choice or rewards withdrawal. (Currently around 3-5% rewards potential)

#### Two Flavours:
##### *single-key based*
or
##### *script based*.

The key-based addresses use a single cryptographic key per
address. The authorization condition to use the address is that one holds the
secret (signing) part of the cryptographic key (for that address) and thus be
able to make a cryptographic signature for that key.

The script-based addresses use a script per address. The authorization
condition to use the address is that the _evaluation_ of the script for the
address results in success. The script expresses the authorization conditions
and evaluation of the script tests if those conditions are met. 

For example, a very simple script might express the condition that one holds a particular cryptographic signing key. The script would express that condition by testing if the transaction has a cryptographic signature from the appropriate cryptographic verification key.

Such a script would, of course, be exactly equivalent to an ordinary  single-key based address: the authorization conditions would be the same!

Thus, we can see that single-key based addresses are in some sense just a (very common) special case and that script addresses are the general case.

Every transaction must contain:
Information to show authorization conditions are met. or *transaction witness*. We say that it _witnesses_ the validity of the transaction using the address. The addresses themselves have a _credential_ which is information sufficient to check that a witness is the right witness.

Specifically, there are two types of such credentials, for key and script
addresses:

+ **Key credential** - a key credential is constructed using a *verification
  key (vk)* (which has corresponding *signing key (sk)*). The credential is the
  cryptographic hash of the verification key *H(vk)*.

  The transaction witness for a key credential consists of the *verification
  key vk* and the signature of transaction body hash using the *signing key sk*.

+ **Script credential** - a script credential is the hash of the script.

  The transaction witness for a script credential is the script itself. There are no
  other inputs for the very simple script language introduced in the Shelley era.
  Scripts in the Plutus language (once that is available) will require
  additional inputs which will also form part of the witness.

## Multi-signature scripts

In Shelley and later eras, multisig scripts are used to make script addresses
where the authorization condition for a transaction to use that address is that
the transaction has signatures from multiple cryptographic keys. Examples
include M of N schemes, where a transaction can be authorized if at least *M*
distinct keys, from a set of *N* keys, sign the transaction.

As with all scripts, the transaction witness for a multisig script address
includes the script itself. The multisig language is so simple that this is
the entire witness: there are no other script data inputs. Although the script
itself is the witness, the script has _conditions_ that must be satisfied. The
conditions for multisig scripts are that the transaction has other ordinary
key witnesses. Which combination of key witnesses are acceptable is, of course,
determined by the script.

The multisig script language is an expression language. Its scripts form an
expression tree. The evaluation of the script produces either `true` or `false`.

In BNF notation, the script expressions follow the following abstract syntax:
```
<script> ::= <RequireSignature> <vkeyhash>
           | <RequireAllOf>     <script>*
           | <RequireAnyOf>     <script>*
           | <RequireMOf> <num> <script>*
```
Note that it is recursive. There are no constraints on the nesting or
size, except that imposed by the overall transaction size limit (given that
the script must be included in the transaction in a script witnesses).

In more detail, the four multisig constructors are:

+ RequireSignature: has the hash of a verification key.

  This expression evaluates to `true` if (and only if) the transaction also
  includes a valid key witness where the witness verification key hashes to the
  given hash. In other words, this checks that the transaction is signed by a
  particular key, identified by its verification key hash.

+ RequireAllOf: has a list of multisig sub-expressions.

  This expression evaluates to `true` if (and only if) _all_ the sub-expressions
  evaluate to `true`. Following standard mathematical convention, the degenerate
  case of an empty list evaluates to `true`.

+ RequireAnyOf: has a list of multisig sub-expressions.

  This expression evaluates to `true` if (and only if) _any_ the sub-expressions
  evaluate to `true`. That is, if one or more evaluate to `true`. The degenerate
  case of an empty list evaluates to `false`.

+ RequireMOf: has a number M and a list of multisig sub-expressions.

  This expression evaluates to `true` if (and only if) at least M of the
  sub-expressions evaluate to `true`.


## Time locking

In the Allegra and later eras, the simple multisig script language above is
extended with two additional terms for expressing conditions on the time.

This extension can be used to make script addresses where it is not possible
to spend from them until after a certain time. Or, in combination with the
existing features, one could make a script where up until a certain time one
group of keys (perhaps M-of-N) is allowed to spend, but after a time another
group of keys is allowed instead.

On the Cardano blockchain, time is expressed via slot numbers, since time slots
correspond to time. The extra constructors allow expressing that the current
time slot number must be before a certain slot, or after a certain slot.

The BNF notation for the abstract syntax is:
```
<script> ::= <RequireSignature>  <vkeyhash>
           | <RequireTimeBefore> <slotno>
           | <RequireTimeAfter>  <slotno>
           | <RequireAllOf>     <script>*
           | <RequireAnyOf>     <script>*
           | <RequireMOf> <num> <script>*
```

The interpretation of "before" and "after" is somewhat subtle, and interacts
with another transaction feature: transaction validity intervals.

In the Allegra and later eras transactions have validity intervals. This is the
range of slots in which they are valid; the range of slots in which they may be
included into blocks on the blockchain. The transaction can specify the first
slot in which the transaction can become valid, and the first slot in which the
transaction becomes invalid. This is a "half-open" interval: the lower bound is
inclusive and the upper bound is exclusive. Both the lower and the upper ends of
this interval are optional, in which case the interpretation is zero or positive
infinity respectively.

With this in mind, we can understand the interpretation of the new expressions:

+ RequireTimeBefore: has a slot number X.

  This expression evaluates to `true` if (and only if) the upper bound of the
  transaction validity interval is a slot number Y, and X <= Y.

  This condition guarantees that the actual slot number in which the transaction
  is included is (strictly) less than slot number X.

+ RequireTimeAfter: has a slot number X.

  This expression evaluates to `true` if (and only if) the lower bound of the
  transaction validity interval is a slot number Y, and Y <= X.

  This condition guarantees that the actual slot number in which the transaction
  is included is greater than or equal to slot number X.

One might reasonably wonder why use this two-stage check via the validity
interval rather than more straightforwardly check the actual slot time. The
reason is to give deterministic script evaluation, which becomes crucial for
Plutus scripts. In this context, by *deterministic* we mean that the result of the
script evaluation depends only on the transaction itself and not any other
context or state of the system. This property is less crucial for this simple
script language, but it is better if all the languages behave the same in this
regard. Even for the simple script language it does still have the advantage
that the script itself can be evaluated without needing to know the current slot
number.


## JSON script syntax

Multi-signature scripts can be written using JSON syntax. This is the format
that the `cardano-cli` tool accepts.

There is a JSON form corresponding to each of the six constructors described
above. Of course, for the Shelley era, only the basic four are available.

### Type "sig"

This corresponds to the "RequireSignature" expression above. It specifies the
type "sig" and the key hash in hex.

```json
{
  "type": "sig",
  "keyHash": "e09d36c79dec9bd1b3d9e152247701cd0bb860b5ebfd1de8abb6735a"
}
```

### Type "all"

This corresponds to the "RequireAllOf" expression above. It specifies the type
"all" and a list of scripts as the sub-expressions.

### Type "any"

This corresponds to the "RequireAllOf" expression above. It specifies the type
"any" and a list of scripts as the sub-expressions.

### Type "atLeast"

This corresponds to the "RequireMOf" expression above. It specifies the type
"atLeast", the required number and a list of scripts as the sub-expressions.

### Type "after"

This corresponds to the "RequireTimeAfter" expression above. It specifies the
type "after" and the slot number.

### Type "before"

This corresponds to the "RequireTimeAfter" expression above. It specifies the
type "before" and the slot number.

### Example of using a script for multi-signatures

Below is an example that shows how to use a script. This is a step-by-step
process involving:

+ the creation of a script address
+ sending ada to that address
+ gathering required witnesses in order to spend ada from the script address.

The example is based on using an `all` script.

#### Sending ada to a script address

#### Step 1 - create a script

First, generate the keys that you require witnesses from using the
`cardano-cli address key-gen` command. Then, construct a script in JSON syntax
as described above. For this example, we will describe the process using an
`all` multisig script:

#### Step 2 - create a script address

A script address is required in order to use a script. Construct this.

#### Step 3 - construct and submit a transaction (tx) to the script address

To construct and submit a tx to send ada to the script address, construct the
transaction body:

Create the transaction witness:

Assemble the transaction witness and the tx body to create the transaction:

After submitting the above tx, the inputs associated with the script address
will be "guarded" by the script.

### Sending ada from a script address

#### Step 1 - construct the tx body

#### Step 2 - construct required witnesses

To construct the script witness and three key witnesses required by the example
`all` script:

#### Step 3 - construct and submit the transaction
To construct and submit a transaction, you must assemble it with the script
witness and all the other required key witnesses.

### Example of using a script for time locking

In this example we will use a script that allows spending from an address
with a particular key, but only after a certain time.

It is important to understand the transaction validity interval and how that
relates to the script: make sure you have read the time locking section above.

Previously in Shelley we had a `--invalid-hereafter` flag in the cli which was an upper bound
on when a transaction can be valid i.e the transaction was valid up until that
slot number. This has been replaced with the `--invalid-before` and
`--invalid-hereafter` flags as described below.

If you specify only a lower bound on a transaction, with `--invalid-before=X`,
it is valid in slot X and thereafter.

If you specify an upper bound on a transaction, with `--invalid-hereafter=X`,
it is valid up to but not including slot X.

If you specify both bounds, with `--invalid-before=X --invalid-hereafter=Y`,
then the tx is valid in slot X up to but not including slot Y.

Whatever bounds you specify, the slot number in which you submit your
transaction has to be within those bounds. For example if you specify
`--invalid-before=600` but you submit your transaction in slot 500 then it will
be rejected.

This interacts with time locking scripts. If you specify a script with
`"after": 1000` then you must specify a `--invalid-before` of at least 1000
and therefore also submit that transaction in or after slot 1000.

Conversely, if you specify a time lock script with `"before": 500` you must
specify an `--invalid-hereafter` slot of at most 500 and submit the transaction
before slot 500. Remember the upper bound is exclusive: `before: 500` means the
transaction is expired in slot 500.

Once you have generated your time lock script you need to follow all the same
steps as above in the multi-signature script example but with a slight
modification 

For `after` scripts we must provide a `--invalid-before` slot that is greater
than or equal to the specified slot number in our simple script. 

Note that this is not really time locking in the normal sense, and indeed this
is a very dangerous script to use because any funds left in a script address
using this script after time slot will be locked there permanently!

For before scripts we must provide a `--invalid-hereafter` slot that is less
than or equal to the specified slot number in our simple script.
