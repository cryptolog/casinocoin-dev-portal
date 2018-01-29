## Payment
[[Source]<br>](https://github.com/casinocoin/casinocoind/blob/4.0.1/src/casinocoin/app/transactors/Payment.cpp "Source")

A Payment transaction represents a transfer of value from one account to another. (Depending on the path taken, this can involve additional exchanges of value, which occur atomically.)

Payments are also the only way to [create accounts](#creating-accounts).

Example payment:

```
{
  "TransactionType" : "Payment",
  "Account" : "cf1BiGeXwwQoi8Z2ueFYTEXSwuJYfV2Jpn",
  "Destination" : "ra5nK24KXen9AHvsdFTKHSANinZseWnPcX",
  "Amount" : {
     "currency" : "USD",
     "value" : "1",
     "issuer" : "cf1BiGeXwwQoi8Z2ueFYTEXSwuJYfV2Jpn"
  },
  "Fee": "12",
  "Flags": 2147483648,
  "Sequence": 2,
}
```

| Field          | JSON Type            | [Internal Type][] | Description      |
|:---------------|:---------------------|:------------------|:-----------------|
| Amount         | [Currency Amount][]  | Amount            | The amount of currency to deliver. For non-CSC amounts, the nested field names MUST be lower-case. If the [**tfPartialPayment** flag](#payment-flags) is set, deliver _up to_ this amount instead. |
| Destination    | String               | Account           | The unique address of the account receiving the payment. |
| DestinationTag | Unsigned Integer     | UInt32            | _(Optional)_ Arbitrary tag that identifies the reason for the payment to the destination, or a hosted recipient to pay. |
| InvoiceID      | String               | Hash256           | _(Optional)_ Arbitrary 256-bit hash representing a specific reason or identifier for this payment. |
| Paths          | Array of path arrays | PathSet           | (Optional, auto-fillable) Array of [payment paths](concept-paths.html) to be used for this transaction. Must be omitted for CSC-to-CSC transactions. |
| SendMax        | [Currency Amount][]  | Amount            | _(Optional)_ Highest amount of source currency this transaction is allowed to cost, including [transfer fees](concept-transfer-fees.html), exchange rates, and [slippage](http://en.wikipedia.org/wiki/Slippage_%28finance%29). Does not include the [CSC destroyed as a cost for submitting the transaction](#transaction-cost). For non-CSC amounts, the nested field names MUST be lower-case. Must be supplied for cross-currency/cross-issue payments. Must be omitted for CSC-to-CSC payments. |
| DeliverMin     | [Currency Amount][]  | Amount            | _(Optional)_ Minimum amount of destination currency this transaction should deliver. Only valid if this is a [partial payment](#partial-payments). For non-CSC amounts, the nested field names are lower-case. |

### Special issuer Values for SendMax and Amount

Most of the time, the `issuer` field of a non-CSC [Currency Amount][] indicates a financial institution's [issuing address](concept-issuing-and-operational-addresses.html). However, when describing payments, there are special rules for the `issuer` field in the `Amount` and `SendMax` fields of a payment.

* There is only ever one balance for the same currency between two addresses. This means that, sometimes, the `issuer` field of an amount actually refers to a counterparty that is redeeming issuances, instead of the address that created the issuances.
* When the `issuer` field of the destination `Amount` field matches the `Destination` address, it is treated as a special case meaning "any issuer that the destination accepts." This includes all addresses to which the destination has extended trust lines, as well as issuances created by the destination which are held on other trust lines.
* When the `issuer` field of the `SendMax` field matches the source account's address, it is treated as a special case meaning "any issuer that the source can use." This includes creating new issuances on trust lines that other accounts have extended to the source account, and sending issuances the source account holds from other issuers.

### Creating Accounts

The Payment transaction type is the only way to create new accounts in the CSC Ledger. To do so, send an amount of CSC that is equal or greater than the [account reserve](concept-reserves.html) to a mathematically-valid account address that does not exist yet. When the Payment is processed, a new [AccountRoot object](reference-ledger-format.html#accountroot) is added to the ledger.

If you send an insufficient amount of CSC, or any other currency, the Payment fails.

### Paths

If present, the `Paths` field must contain a _path set_ - an array of path arrays. Each individual path represents one way value can flow from the sender to receiver through various intermediary accounts and order books. A single transaction can potentially use multiple paths, for example if the transaction exchanges currency using several different order books to achieve the best rate.

You must omit the `Paths` field for direct payments, including:

* An CSC-to-CSC transfer.
* A direct transfer on a trust line that connects the sender and receiver.

If the `Paths` field is provided, the server decides at transaction processing time which paths to use, from the provided set plus a _default path_ (the most direct way possible to connect the specified accounts). This decision is deterministic and attempts to minimize costs, but it is not guaranteed to be perfect.

The `Paths` field must not be an empty array, nor an array whose members are all empty arrays.

For more information, see [Paths](concept-paths.html).

### Payment Flags

Transactions of the Payment type support additional values in the [`Flags` field](#flags), as follows:

| Flag Name        | Hex Value  | Decimal Value | Description                  |
|:-----------------|:-----------|:--------------|:-----------------------------|
| tfNoDirectCasinocoin | 0x00010000 | 65536         | Do not use the default path; only use paths included in the `Paths` field. This is intended to force the transaction to take arbitrage opportunities. Most clients do not need this. |
| tfPartialPayment | 0x00020000 | 131072        | If the specified `Amount` cannot be sent without spending more than `SendMax`, reduce the received amount instead of failing outright. See [Partial Payments](#partial-payments) for more details. |
| tfLimitQuality   | 0x00040000 | 262144        | Only take paths where all the conversions have an input:output ratio that is equal or better than the ratio of `Amount`:`SendMax`. See [Limit Quality](#limit-quality) for details. |

### Partial Payments

A partial payment allows a payment to succeed by reducing the amount received. Partial payments are useful for [returning payments](tutorial-gateway-guide.html#bouncing-payments) without incurring additional costs to oneself. However, partial payments can also be used to exploit integrations that naively assume the `Amount` field of a successful transaction always describes the exact amount delivered.

A partial payment is any [Payment transaction][] with the **tfPartialPayment** flag enabled. A partial payment can be successful if it delivers any positive amount greater than or equal to its `DeliverMin` field (or any positive amount at all if `DeliverMin` is not specified) without sending more than the `SendMax` value.

The [`delivered_amount`](#delivered-amount) field of a payment's metadata indicates the amount of currency actually received by the destination account.

For more information, see the full article on [Partial Payments](concept-partial-payments.html).


### Limit Quality

The CSC Ledger defines the "quality" of a currency exchange as the ratio of the numeric amount in to the numeric amount out. For example, if you spend $2 USD to receive £1 GBP, then the "quality" of that exchange is `0.5`.

The [*tfLimitQuality* flag](#payment-flags) allows you to set a minimum quality of conversions that you are willing to take. This limit quality is defined as the destination `Amount` divided by the `SendMax` amount (the numeric amounts only, regardless of currency). When set, the payment processing engine avoids using any paths whose quality (conversion rate) is worse (numerically lower) than the limit quality.

By itself, the tfLimitQuality flag reduces the number of situations in which a transaction can succeed. Specifically, it rejects payments where some part of the payment uses an unfavorable conversion, even if the overall average *average* quality of conversions in the payment is equal or better than the limit quality. If a payment is rejected in this way, the [transaction result](#transaction-results) is `tecPATH_DRY`.

Consider the following example. If I am trying to send you 100 Chinese Yuan (`Amount` = 100 CNY) for 20 United States dollars (`SendMax` = 20 USD) or less, then the limit quality is `5`. Imagine one trader is offering ¥95 for $15 (a ratio of about `6.3` CNY per USD), but the next best offer in the market is ¥5 for $2 (a ratio of `2.5` CNY per USD). If I were to take both offers to send you 100 CNY, then it would cost me 17 USD, for an average quality of about `5.9`.

Without the tfLimitQuality flag set, this transaction would succeed, because the $17 it costs me is within my specified `SendMax`. However, with the tfLimitQuality flag enabled, the transaction would fail instead, because the path to take the second offer has a quality of `2.5`, which is worse than the limit quality of `5`.

The tfLimitQuality flag is most useful when combined with [partial payments](#partial-payments). When both *tfPartialPayment* and *tfLimitQuality* are set on a transaction, then the transaction delivers as much of the destination `Amount` as it can, without using any conversions that are worse than the limit quality.

In the above example with a ¥95/$15 offer and a ¥5/$2 offer, the situation is different if my transaction has both tfPartialPayment and tfLimitQuality enabled. If we keep my `SendMax` of 20 USD and a destination `Amount` of 100 CNY, then the limit quality is still `5`. However, because I am doing a partial payment, the transaction sends as much as it can instead of failing if the full destination amount cannot be sent. This means that my transaction consumes the ¥95/$15 offer, whose quality is about `6.3`, but it rejects the ¥5/$2 offer because that offer's quality of `2.5` is worse than the quality limit of `5`. In the end, my transaction only delivers ¥95 instead of the full ¥100, but it avoids wasting money on poor exchange rates.