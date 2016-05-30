---
title: API Reference

language_tabs:

toc_footers:
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

includes:

search: true
---

# 1.0 Enterpay Payment Integration Guide

Enterpay's payment button enables your customers to pay for their purchases by invoice. This is especially convenient for corporate customers.

The button is very similar technically to most bank payment buttons. This document explains how to integrate the button to your site.

## 1.1.0 Tutorial

Integrating the Enterpay payment button consists of the following steps.

1. Sign a contract with Enterpay and obtain a public merchant ID and a secret API key.
2. Write code to show the payment button (an HTML form with hidden fields).
3. Write a return page that receives the user after the purchase completes or fails.

You can freely test against the dev installation at [test.laskuyritykselle.fi](https://test.laskuyritykselle.fi/).

## 1.1.1 HTML form

The first step is to generate an HTML form like the following.

```html

<form action="https://laskuyritykselle.fi/api/payment/start" method="post">
  <!-- 'version' should be 1 -->
  <input type="hidden" name="version" value="1" />

  <!-- 'merchant' should be your public merchant ID -->
  <input type="hidden" name="merchant" value="MyMerchantId123" />

  <!-- 'key_version' should be the version number of your current secret API key -->
  <input type="hidden" name="key_version" value="1" />

  <!-- 'identifier_merchant' should be your unique database identifier for this purchase event. -->
  <input type="hidden" name="identifier_merchant" value="abc123" />

  <!-- 'reference' should be the reference number with which Enterpay credits the merchant. It is NOT shown on the end user's invoice. -->
  <input type="hidden" name="reference" value="10001 10009" />

  <!-- 'locale' tells the locale that the user sees the Enterpay UI in -->
  <input type="hidden" name="locale" value="en_US" />

  <!-- 'currency' should be a three-letter currency code -->
  <input type="hidden" name="currency" value="EUR" />

  <!-- 'total_price_including_tax' should be the total price of the purchase in cents. It must match the prices of the items specified below. -->
  <input type="hidden" name="total_price_including_tax" value="119700" />

  <!-- 'url_return' tells where the user gets returned after a success/failure/cancel -->
  <input type="hidden" name="url_return" value="https://your.shop.com/valuebuy_purchase_complete" />

  <!-- 'hmac' is a signature that is explained below -->
  <input type="hidden" name="hmac" value="EBD61377C772E99BA5C9A08645854EE2A9DB7...." />


  <!-- Next we describe the items that the customer is purchasing. These will show up on the invoice. -->

  <!-- 'identifier' should be your own (preferably unique) identifier for this product type. A product code will work. -->
  <input type="hidden" name="cart_items[0][identifier]" value="ACME001" />

  <!-- 'name' should be the name of the product as shown to the user. -->
  <input type="hidden" name="cart_items[0][name]" value="Acme Supertablet 7" />

  <!-- 'quantity' should be an integer or a decimal that tells the number of items being sold. -->
  <input type="hidden" name="cart_items[0][quantity]" value="3" />

  <!-- 'unit_price_including_tax' should be the tax-inclusive price of ONE item in cents. It's also possible to specify the tax-free price. -->
  <input type="hidden" name="cart_items[0][unit_price_including_tax]" value="39900" />

  <!-- 'tax_rate' should be the VAT tax rate that applies to this product. -->
  <input type="hidden" name="cart_items[0][tax_rate]" value="0.24" />

  <!-- The next item's fields would be prefixed cart_items[1] and so on. -->

  <!-- This is the payment button that the user clicks to start the payment process through Enterpay. -->
  <input type="image"
         src="https://laskuyritykselle.fi/payment/images/button/fi-207x75-1.png"
         alt="Lasku yritykselle"
         title="Lasku yritykselle"
         name="submit" />
</form>

```

You can use use our [mock implementation](https://test.laskuyritykselle.fi/merchant-client-mock/) to experiment with different parameters.

Consult the [reference](http://www.enterpay.fi/kehittajalle/#starting_payment) for more information and for additional optional fields. The above example only shows a minimal set of fields.

The `hmac` field must contain a cryptographic signature of the other fields, computed using your secret API key. The algorithm is explained [here](https://laskuyritykselle.fi/org/login).

The `identifier_merchant` field should uniquely identify the ongoing purchase in your system. Enterpay [guarantees](http://www.enterpay.fi/kehittajalle/#identifier_merchant_reference) that at most one payment for a given `identifier_merchant` value succeeds. This way the user can start, fail and cancel the payment multiple times without you needing to create a new `identifier_merchant` value each time.


## 1.1.2 Return page

The user will be redirected to the page at `url_return`. Some [GET parameters](http://www.enterpay.fi/kehittajalle/#returned_parameters) that get added to the return URL. Here is an example:

`?version=1&status=successful&identifier_valuebuy=123456789abcdef&identifier_merchant=abc123&key_version=1&hmac=F9521CE36134BD5DBFB6D9C....`

If you want to mark the purchase as verified as soon as the user arrives to the return page, then you **must** verify the `hmac`. Otherwise an attacker can skip the payment process and go directly to the return page. Remember to also check that `status` is `successful`.

It is also recommended to make the return page *idempotent* i.e. such that it works even if the user navigates to it twice or refreshes the page.

## 1.1.3 Miscellaneous

* An explanation of how the total price is calculated and how rounding is done is [here](http://www.enterpay.fi/kehittajalle/#rounding).
* The production installation is at [laskuyritykselle.fi](https://laskuyritykselle.fi/)
* The test installation is at [test.laskuyritykselle.fi](https://test.laskuyritykselle.fi/)


# 1.2 Integration Reference

## 1.2.1 Starting a payment

This request creates a new payment and redirects the browser to Enterpay's payment UI.

* URL: `https://laskuyritykselle.fi/api/payment/start`
* Method: `POST`
* Format: Form data

Field | Req. | [Type](#data-types) | Description
----- | ---- | ------------------- | -----------
version | R | Integer | API version. Currently 1.
merchant |	R | Identifier |	Your merchant identifier (NOT your API key).
identifier_merchant |	R |	Identifier |	An unique identifier for the purchase in the merchant's database. Multiple payments may be started with the same identifier as long as no purchase with that identifier has yet succeeded.
locale |	R |	Text(5) |	The locale to show the payment UI with. Format: `lang + "_" + country`. Examples: en_US, fi_FI. Language codes: ISO 639-1. Country codes: ISO 3166-1 alpha-2.
currency |	R | Currency |	The currency used for this payment.
total_price_including_tax |	O/R |	Money |	Total price, which must match the total calculated for rows. Either this or `total_price_excluding_tax` must be specified. See [here](http://www.enterpay.fi/kehittajalle/#rounding) for details on rounding.
total_price_excluding_tax |	O/R |	Money |	Total price, which must match the total calculated for rows. Either this or `total_price_including_tax` must be specified. See here for details on rounding.
reference |	R |	Text(100) |	Reference number for crediting. Enterpay will use this when it credits the purchase to the merchant. Note: this field has nothing to do with the reference the customer sees on his invoice. Reference number should follow the specification [here](https://laskuyritykselle.fi/org/login).
invoice_reference |	O |	Text(50) |	If the user has given an invoice reference number that he wishes to see on his invoice, you should give it here. The user may also set this during the payment process. Note: this field has nothing to do with the reference number that Enterpay uses to credit the purchase to the merchant.
cost_pool |	O |	Text(70) | An optional metadata field for the user's convenience. May be filled in during the payment process.
note |	O |	Text(100) |	An optional metadata field for the user's convenience. May be filled in during the payment process.
billing_address |	O |	PostalAddress |	If the user has given the billing address that he wishes to use, you should give it here. The user may also set this during the payment process.
delivery_address | O |	PostalAddress |	If the user has given the delivery address that he wishes to use, you should give it here. This information cannot be changed during the payment process.
url_return |	R |	URL |	Enterpay sends the user to this URL after the payment concludes. See [Returned Parameters](http://www.enterpay.fi/kehittajalle/#returned_parameters) for the GET parameters that are added.
prevent_pending_status |	O |	Boolean |	Override for preventing pending status. If not specified, default merchant value will be used. This might cause that the buyer might not be able to make high purchases in the given webshop.
automatic_invoicing_off |	O |	Boolean |	Override for automatic invoicing parameter set by Enterpay. If not specified, default merchant value will be used.
invoicing_start_date |	O |	Date |	This parameter sets the invoicing date as specified. Parameter can't be in the past.
key_version |	R |	Integer |	The version number of the API key being used. See also: [switching API keys](http://www.enterpay.fi/kehittajalle/#switching_keys)
hmac |	R |	HMAC-SHA512 |	The [HMAC signature](http://www.enterpay.fi/kehittajalle/#hmac_calculation).

##1.2.2 Address fields

Billing address parameter keys are of the form `billing_address[key]` and delivery address parameter keys are of the form `delivery_address[key]`.

Both address parameters have the same set of keys.

Field |	Req. | [Type](http://www.enterpay.fi/kehittajalle/#ref_types) | Description
----- | ---- | ------------------------------------------------------ | -----------
billing_address[street] |	R |	Text(100) |	The street address of user's billing/delivery address.
billing_address[postalCode] |	R |	Text(10) |	The postal code of user's billing/delivery address.
billing_address[city] |	R |	Text(100) |	The city of user's billing/delivery address.

## 1.2.3 Cart item fields

In addition to the above, the request must contain at least one cart item.

Cart items parameter keys are of the form `cart_items[N][key]` where N is the index of the cart item, starting from 0.

Field |	Req. | [Type](http://www.enterpay.fi/kehittajalle/#ref_types) |	Description
----- | ---- | ------------------------------------------------------ | -----------
cart_items[N][identifier] |	R |	Identifier |	Identifies the product in the merchant's database.
cart_items[N][name] |	R |	Text(200) |	The name of the product as displayed on the invoice.
cart_items[N][quantity] |	R |	Decimal(10,3) |	The quantity of items on this row.
cart_items[N][unit_price_excluding_tax] |	O/R |	Money |	The cost of this item without tax. May be negative as long as the total sum is non-negative. At least one of unit_price_excluding_tax and unit_price_including_tax must be given.
cart_items[N][unit_price_including_tax] |	O/R |	Money |	The cost of this item with tax. May be negative as long as the total sum is non-negative. At least one of unit_price_excluding_tax and unit_price_including_tax must be given.
cart_items[N][tax_rate] |	R |	Decimal(10,4) |	The VAT tax rate that applies to this product. e.g. 0.24 means 24%.

## 1.2.4 Rounding and price calculation

Enterpay checks that the total price of the cart items matches the total price of the purchase. If you provide decimal quantities for some products then this section, where Enterpay's rounding scheme is explained, is probably relevant to you.

Each cart item's unit price is calculated. Then it is multiplied by the quantity and rounded according to normal rounding rules, with exactly half being rounded up. Finally, the rounded prices for the cart items are summed and added together. Adding together rounded prices cannot cause rounding errors.

>In pseudocode:

```pseudocode
for each cart item do
    if unit_price_including_tax was given then
        unit_price_excluding_tax = unit_price / tax_rate
    else
        unit_price_including_tax = unit_price * tax_rate

    total_price_including_tax = round(unit_price_including_tax * quantity)
    total_price_excluding_tax = round(unit_price_excluding_tax * quantity)

total_price_including_tax = sum each the total_price_including_tax of each cart item
total_price_excluding_tax = sum each the total_price_excluding_tax of each cart item
```
The total that you provide must match the total that Enterpay calculates.

Rounding is done at each row because that guarantees that the total shown on the invoice is the sum of each row shown on the invoice. If we didn't round when we do, the total could be a few cents greater than the sum of the rows, because the hidden extra precision would not be printed on the invoice.

## 1.2.5 HMAC calculation

To avoid spoofing or manipulation of payment requests, all parameters must be signed with the secret API key using the HMAC-SHA512 algorithm.

The string to feed the HMAC algorithm is constructed from a set of key-value pairs as follows.

1. Remove keys with an empty or missing value.
2. Sort the pairs by the key name.
3. URL-encode each key and value. Use `application/x-www-form-urlencoded` encoding (i.e. RFC 1738, where spaces are encoded as '+').
4. Convert each key-value pair into the string (`key + "=" + value`).
5. Concatenate those strings with `&` as a separator.

Feed the string to the HMAC-SHA512 algorithm along with your merchant secret, and send the result as an uppercase hexadecimal string.

>Sample code (PHP):

```PHP
<?php
$params = array('key1' => 'value1', ...);

ksort($params);

foreach ($params as $k => $v) {
  if ($v !== null && $v !== '') {
    $params[$k] = urlencode($k) . '=' . urlencode($v);
  } else {
    unset($params[$k]);
  }
}

$str = implode('&', $params);

$hmac = strtoupper(hash_hmac('sha512', $str, $secret));
?>
```
The reason for sorting the fields and dropping empty fields is forward-compatibility: new optional fields may be added to the API without changing the version number.

## 1.2.6 Returned parameters

The following values are returned by Enterpay after a payment has concluded.

They are added as GET parameters to `url_return`.

Field |	[Type](http://www.enterpay.fi/kehittajalle/#ref_types) | Description
----- | ------------------------------------------------------ | -----------
version |	Integer |	API version. Currently 1.
status | `{successful, failed, pending, canceled}` |	Whether the purchase was successful, failed for some reason, has been marked pending or was explicitly canceled.
pending_reasons |	`{credit-check, supervisor-approval}` |	Optional return parameter indicating the reason(s) of pending status. If there are several reasons, they are separated by commas. See also: [possible pending reasons](http://www.enterpay.fi/kehittajalle/#pending_reasons)
identifier_valuebuy |	Identifier | An unique identifier assigned to the purchase by Enterpay.
identifier_merchant |	Identifier | The unique identifier assigned to the purchase by the merchant.
key_version |	Integer |	The version of the secret key used for HMAC calculation. Matches the `key_version` given when the purchase was started. See also: [switching API keys](http://www.enterpay.fi/kehittajalle/#switching_keys)
hmac | HMAC-SHA512 | Calculated from the other fields as specified [here](http://www.enterpay.fi/kehittajalle/#hmac_calculation). You **MUST** verify this HMAC, or else an attacker may spoof calls to this URL.  

## 1.2.7 Possible pending reasons

If pending reasons are available, possible values are

* in_progress
* paid, unpaid, overdue (successful)
* failed
* canceled
* pending (will become paid, unpaid, overdue (successful) or canceled)

## 1.2.8 Switching API keys

The secret API key must be changed periodically in order to mitigate brute-force attacks and limit the longevity of stolen keys. Consult the admin panel to manage your API keys.

Note that, at any point in time, any number of your users may be

* viewing the payment button
* going through Enterpay's payment UI
* on their way to the return page

The simplest way to switch keys is to take down the Enterpay payment button for 30 minutes and wait for all pending payments will be either finished or expired. After 30 minutes, change the API key in your system and restore the Enterpay payment button.

Here are the exact steps you should follow:

1. Comment out the code that shows the Enterpay payment button.
2. Wait for 30 minutes.
3. Generate a new API key in the admin panel.
4. Activate the new API key in the admin panel.
5. Change the API key in your system.
6. Uncomment the code that shows the Enterpay payment button.

# 2.0 Enterpay Payment Invoices API

Enterpay offers merchants an API for retrieving, updating, cancelling and refunding invoices made through the Enterpay payment button. Every API request includes a set of parameters: required credential information of merchant and additional parameters depending on the used API functionality.

## 2.1.0 Tutorial

## 2.1.1 Retrieving invoice

A GET request to the URL `https://laskuyritykselle.fi/api/merchant/invoices` retrieves invoice information. The following is an example of the required GET parameters.

`?merchant=7d330bd2-539f-46ba-819f-5f60c6236af9&merchant_key_version=1&identifier_merchant=mid-`
`f5a0ec4d-abf9-4a3f-9b43-8c43cd4a5424&hmac=8043df52f957a50dfc3476233e9b4d4d12f2c38efef0852502a2`
`8645b45084b6253e23d68d44a822c27422adf0c9f24624468f286b0682deda59af772c76e6e8`

Consult the [reference](http://www.enterpay.fi/kehittajalle/#ref_retrieve_invoice) for more information.

>The request returns a JSON object with following information:

```json
{
  "identifier_merchant": "mid-f5a0ec4d-abf9-4a3f-9b43-8c43cd4a5424",
  "customer_org": "Company Oy",
  "customer_user": "Tommy Tester",
  "reference": "reference",
  "status": "paid",
  "is_refundable": true,
  "created_at": "2014-01-07T09:58:19Z",
  "invoicing_date": "2014-01-09",
  "due_date": "2014-01-23",
  "total_price_taxed": 215181,
  "total_price_taxed_refunded": 155420,
  "currency": "EUR",
  "cart_items": [
    {
      "num": 0,
      "identifier_merchant": "product-1",
      "name": "Test item #1",
      "quantity": "1.000",
      "unit_price_excluding_tax": 7675,
      "unit_price_including_tax": 9517,
      "total_price_taxed": 9517,
      "total_price_taxed_refunded": 8517,
      "tax_rate": "0.240"
    },
    {
      "num": 1,
      "identifier_merchant": "product-2",
      "name": "Test item #2",
      "quantity": "7.000",
      "unit_price_excluding_tax": 23694,
      "unit_price_including_tax": 29381,
      "total_price_taxed": 205664,
      "total_price_taxed_refunded": 146903,
      "tax_rate": "0.240"
    }
  ],
  "refunds": [
    {
      "invoicing_date": "2014-02-09",
      "due_date": "2014-02-23",
      "total_price_taxed": 58761,
      "refunded_items": [
        {
          "num": 0,
          "currency": "EUR",
          "refunded_amount": 1000
        },
        {
          "num": 1,
          "refunded_quantity": "2.000"
        }
      ]
    }
  ]
}
```
>The format of returned JSON data is defined in [reference](http://www.enterpay.fi/kehittajalle/#ref_retrieve_invoice).

## 2.1.2 Updating invoice

An invoice can be updated by a PUT request to the URL `https://laskuyritykselle.fi/api/merchant/invoices`.

<aside class="notice">Note: Updating an invoice is only possible before the original invoice has been sent. After the invoice has been sent, it may only be <a href="http://www.enterpay.fi/kehittajalle/#refunding_invoice">refunded</a>.</aside>

>The parameters are passed a JSON object such as the following:

```json
{
  "merchant": "7d330bd2-539f-46ba-819f-5f60c6236af9",
  "merchant_key_version": 1,
  "identifier_merchant": "mid-f5a0ec4d-abf9-4a3f-9b43-8c43cd4a5424",
  "update": {
    "reference": "newReference",
    "invoicing_date": "2014-01-09",
    "cart_items": [
      {
        "num": 0,
        "identifier_merchant": "product-1",
        "name": "Test item #1",
        "quantity": "1.000",
        "unit_price_excluding_tax": 7675,
        "currency": "EUR",
        "tax_rate": "0.240"
      },
      {
        "num": 1,
        "identifier_merchant": "product-2",
        "name": "Test item #2",
        "quantity": "5.000",
        "unit_price_excluding_tax": 23694,
        "currency": "EUR",
        "tax_rate": "0.240"
      }
    ]
  },
  "hmac": "ae8f89c22ce24107885411ba10ca7aee2f42788651514d778fac59db667f3bbb9b0e8bf8cf553764b374350a03a159bedc893e06dd2f7fee104adef7ae97414d"
}
```
<aside class="notice">Note: The JSON object must contain all cart items (also the ones you don't want to change), because the entire list of items will be replaced with the ones you provide. Consult the <a href="http://www.enterpay.fi/kehittajalle/#ref_update_invoice">reference</a> for more information about parameters.</aside>

## 2.1.3 Cancelling invoice

The invoice can be canelled through a PUT request to the URL `https://laskuyritykselle.fi/api/merchant/invoices/cancel`.

<aside class="notice">Note: Cancellation of an invoice is only possible before the invoice has been sent. After the invoice has been sent, it may only be <a href="http://www.enterpay.fi/kehittajalle/#refunding_invoice"> refunded</a>.</aside>

>The parameters are passed as a JSON object such as the following:

```json
{
  "merchant": "7d330bd2-539f-46ba-819f-5f60c6236af9",
  "merchant_key_version": 1,
  "identifier_merchant": "mid-f5a0ec4d-abf9-4a3f-9b43-8c43cd4a5424",
  "hmac": "8043df52f957a50dfc3476233e9b4d4d12f2c38efef0852502a28645b45084b6253e23d68d44a822c27422adf0c9f24624468f286b0682deda59af772c76e6e8"
}
```
Consult the [reference](http://www.enterpay.fi/kehittajalle/#ref_cancel_invoice) for more information.

## 2.1.4 Refunding invoice

Refunding an invoice is done through a POST request to URL `https://laskuyritykselle.fi/api/merchant/invoices/refund`. The request creates a new credit invoice for the purchase. Multiple credit invoices can be created for the same purchase by sending multiple requests with the same purchase identifier, as long as the refunded amounts don't exceed original amounts. The refunding can be done via quantity or amount per purchase item.

> The parameters are passed a JSON object such as the following:

```json
{
  "merchant": "7d330bd2-539f-46ba-819f-5f60c6236af9",
  "merchant_key_version": 1,
  "identifier_merchant": "mid-f5a0ec4d-abf9-4a3f-9b43-8c43cd4a5424",
  "items_to_refund": [
    {
      "num": 1,
      "refunding_type": "amount",
      "refunded_quantity": 2
    }
  ],
  "invoicing_date": "2014-01-09",
  "hmac": "f70500eb45bfbefa2740512d3b5dab27db70d3b5a4d81ec1d4f2c7da692695d344e100b953607be813de523ec631b5961f7150b98affde71156c88c1685761b0"
}
```
Consult the [reference](http://www.enterpay.fi/kehittajalle/#ref_refund_invoice) for more information about parameters.

# 2.2 Invoices Reference

## 2.2.1 Common parameters

The following parameters are used in all requests to Invoices API, so use these in addition to the parameters defined under later topics.

Field |	Req. | [Type](http://www.enterpay.fi/kehittajalle/#ref_types) |	Description
----- | ---- | ------------------------------------------------------ | -----------
merchant | R |	Identifier |	Your merchant identifier (NOT your API key).
merchant_key_version |	R |	Integer |	The version number of the merchant API key being used. See also: [switching API keys](http://www.enterpay.fi/kehittajalle/index.html#switching_keys)
identifier_merchant |	R |	Identifier |	A unique identifier of the purchase in the merchant's database (given in the start of payment process).
user |	O |	String |	Optional string parameter for custom identifier.
hmac |	R |	HMAC-SHA512 |	[The HMAC signature.](http://www.enterpay.fi/kehittajalle/#ref_hmac)

## 2.2.2 Retrieving invoice

This request retrieves the invoice information by given purchase identifier and returns it as JSON object.

* URL: https://laskuyritykselle.fi/api/merchant/invoices?(parameters)
* Method: GET
* Parameters: Common parameters

## 2.2.3 Returned data (JSON)

<aside class="notice">Note, that value type Integer is returned as JInt and all other types are returned as JString (so here for example Date is not that of javascript).</aside>

Field |	[Type](http://www.enterpay.fi/kehittajalle/#ref_types) | Description
----- | ------------------------------------------------------ | -----------
identifier_merchant |	Identifier |	A unique identifier of the purchase in the merchant's database (given in the start of payment process).
customer_org |	Text(200) |	Name of the organisation that the purchase has been made on behalf of.
customer_user |	Text(255) |	Name of the user who made the purchase.
reference |	Text(100) |	Reference number for crediting. Enterpay will use this when it credits the purchase to the merchant. Note: this field has nothing to do with the reference the customer sees on his invoice. Reference number should follow the specification here.
status |	`{successful, failed, pending, canceled}` |	The status of the purchase.
is_refundable |	`{true, false}` |	Defines if the purchase is in refundable state.
created_at |	DateTime |	The time when purchase was created in ISO8601 format (`yyyy-MM-ddTHH:mm:ss.SSSZZ`).
invoicing_date |	Date |	The date when purchase is being invoiced from the customer organisation.
due_date |	Date |	The due date of the sent invoice.
total_price_taxed |	Integer |	The total price of purchase including taxes.
total_price_taxed_refunded |	Integer |	The total price of purchase including taxes and possible refunds (credit invoices).
currency |	Currency |	The currency of prices.
cart_items[n][num] |	Integer |	The index of cart item (starts from 0).
cart_items[n][identifier_merchant] |	Identifier |	Identifies the product in the merchant's database.
cart_items[n][name] |	Text(200) |	The name of the product as displayed on the invoice.
cart_items[n][quantity] |	Decimal(10,3) |	The quantity of items on this row.
cart_items[n][unit_price_excluding_tax] |	Money |	The unit price of this item without tax.
cart_items[n][unit_price_including_tax] |	Money |	The unit price of this item with tax.
cart_items[n][total_price_taxed] |	Money |	The total price of this item with tax.
cart_items[n][total_price_taxed_refunded] |	Money |	The total price of this item with tax and possible refunds.
cart_items[n][tax_rate] |	Decimal(10,4) |	The VAT tax rate that applies to this product. e.g. 0.24 means 24%.
refunds[n][invoicing_date] |	Date |	The date when this credit invoice is being invoiced from the customer organisation.
refunds[n][due_date] |	Date |	The due date of this credit invoice.
refunds[n][total_price_taxed] |	Money |	The total price of credit invoice including taxes. Note, that the amount is given as positive, but in the invoice it will be negative.
refunds[n][refunded_items][m][num] |	Integer |	The index of original purchase item.
refunds[n][refunded_items][m][refunded_quantity] |	Decimal(10, 3) |	The quantity of refunded items on this row.
refunds[n][refunded_items][m][currency] |	Currency |	The currency of refund.
refunds[n][refunded_items][m][refunded_amount] |	Money |	The total taxed amount of refund.
refunds[n][refunded_vat_bases][m][vat_base] |	Decimal(10,4) |	The VAT base for which the refund was made.
refunds[n][refunded_vat_bases][m][currency] |	Currency |	The currency of refund.
refunds[n][refunded_vat_bases][m][refunded_amount] |	Money |	The total taxed amount of refund.

## 2.2.4 Updating invoice

This request updates the invoice of the purchase by given identifier.

* URL: `https://laskuyritykselle.fi/api/merchant/invoices`
* Method: `PUT`
* Parameters: Common parameters plus the following parameters:

Field |	Req. | [Type](http://www.enterpay.fi/kehittajalle/#ref_types) |	Description
----- | ---- | ------------------------------------------------------ | -----------
update[reference] |	R |	Text(100) |	Invoice reference string. Enterpay will use this when it credits the purchase to the merchant. Note: this field has nothing to do with the reference number the customer sees on his invoice.
update[invoicing_date] |	R |	Date |	The date when this invoice is sent.
update[cart_items][N][num] |	R |	Integer |	The index of this item (returned by the [Retrieve](http://www.enterpay.fi/kehittajalle/#retrieving_invoice) request).
update[cart_items][N][identifier_merchant] |	R |	Identifier |	Identifies the product in the merchant's database.
update[cart_items][N][name] |	R |	Text(200) |	The name of the product as displayed on the invoice.
update[cart_items][N][quantity] |	R |	Decimal(10,3) |	The quantity of items on this row.
update[cart_items][N][unit_price_excluding_tax] |	O/R |	Money |	The unit price of this item without tax. May be negative as long as the total sum is non-negative. At least one of unit_price_excluding_tax and unit_price_including_tax must be given.
update[cart_items][N][unit_price_including_tax] |	O/R |	Money |	The unit price of this item with tax. May be negative as long as the total sum is non-negative. At least one of unit_price_excluding_tax and unit_price_including_tax must be given.
update[cart_items][N][currency] |	R |	Currency |	The currency used for this product.
update[cart_items][N][tax_rate] |	R |	Decimal(10,4) |	The VAT tax rate that applies to this product. e.g. 0.24 means 24%.

<aside class="information">Note: The request must contain at least one cart item. Cart items parameter keys are of the form <code>cart_items[N][key]</code> where N is the index of the cart item, starting from 0.</aside>

## 2.2.5 Cancelling invoice

This request cancels the invoice for given purchase identifier.

* URL: `https://laskuyritykselle.fi/api/merchant/invoices/cancel`
* Method: `PUT`
* Parameters: Common parameters

## 2.2.6 Refunding invoice

This request creates a credit invoice for the purchase by given identifier.

* URL: `https://laskuyritykselle.fi/api/merchant/invoices/refund`
* Method: `POST`
* Parameters: Common parameters plus the following parameters:

Field |	Req. | [Type](http://www.enterpay.fi/kehittajalle/#ref_types) |	Description
----- | ---- | ------------------------------------------------------ | -----------
refund[items_to_refund][N][num] |	R |	Integer |	The index of the original item to be refunded (returned by the [Retrieve](http://www.enterpay.fi/kehittajalle/#retrieving_invoice) request).
refund[items_to_refund][N][refunding_type] |	R |	`{quantity, amount}` |	The refunding type of this item.
refund[items_to_refund][N][refunded_quantity] |	O/R |	Decimal(10,3) |	The quantity refunded of this item. Either this or currency and refundedAmount must be specified, depending on chosen refunding type.
refund[items_to_refund][N][currency] |	O/R |	Currency |	The currency of refunded amount. Either this and refundedAmount, or refundedQuantity must be specified, depending on chosen refunding type.
refund[items_to_refund][N][refunded_amount] |	O/R |	Integer |	The amount to refund of this purchase item. Either this and currency, or refundedQuantity must be specified, depending on chosen refunding type.
refund[vat_bases_to_refund][N][vatBase] |	O/R |	Decimal(10,4) |	The VAT base that is refunded.
refund[vat_bases_to_refund][N][currency] |	O/R |	Currency |	The currency of refund.
refund[vat_bases_to_refund][N][refundedAmount] |	O/R |	Money |	The total taxed amount of refund.
refund[invoicing_date] |	R |	Date |	The date when this credit invoice is sent.

<aside class="information">Note: The request must contain at least one refund either per an item or a VAT base. Item refund parameter keys in the list are presented in form <code>items_to_refund[N][key]</code>, where N is the index of the refunded item, starting from 0. VAT base refund parameter keys are similarly presented in form <code>vat_bases_to_refund[N][key]</code>. The parameters should still be sent to the server as JSON.</aside>

## 2.2.7 HMAC calculation

To avoid spoofing or manipulation of API requests, all parameter values must be signed with the secret API key using the HMAC-SHA512 algorithm. Signing is with merchant secret key.

The string to feed the HMAC algorithm is constructed from the values of parameters as follows.

1. Sort request parameters by key.
2. Remove empty or missing values.
3. URL-encode each value. Use application/x-www-form-urlencoded encoding (i.e. RFC 1738, where spaces are encoded as '+').
4. Concatenate values with & as a separator.

>Example of HMAC calculation in the request for updating invoice (request shown in tutorial above):

```PHP
<?php

function calc_hmac($merchant_secret, $params) {
  $keys = array_keys($params);
  sort($keys);

  $values = array();
  foreach ($keys as $key) {
    $value = $params[$key];
    if ($value !== null && $value !== "") {
      $encoded_value = urlencode($value);
      array_push($values, $encoded_value);
    }
  }

  $str = implode("&", $values);
  return hash_hmac("sha512", $str, $merchant_secret);
}

function flatten_array($params, $base_key = '') {
  $result = array();
  foreach ($params as $key => $value) {
    $new_key = $base_key . $key;
    if (is_array($value)) {
      $result = array_merge($result, flatten_array($value, $new_key));
    }
    else {
      $result[$new_key] = $value;
    }
  }

  return $result;
}

$params = array(
  "merchant"              => "7d330bd2-539f-46ba-819f-5f60c6236af9",
  "merchant_key_version"  => 1,
  "identifier_merchant"   => "mid-f5a0ec4d-abf9-4a3f-9b43-8c43cd4a5424",

  "update" => array(
    "reference"           => "newReference",
    "invoicing_date"      => "2014-01-09",

    "cart_items" => array(
      array(
            "num"                       => 0,
            "identifier_merchant"       => "product-1",
            "name"                      => "Test item #1",
            "quantity"                  => "1.000",
            "unit_price_excluding_tax"  => 7675,
            "currency"                  => "EUR",
            "tax_rate"                  => "0.240"
        ),
        array(
            "num"                       => 1,
            "identifier_merchant"       => "product-2",
            "name"                      => "Test item #2",
            "quantity"                  => "5.000",
            "unit_price_excluding_tax"  => 23694,
            "currency"                  => "EUR",
            "tax_rate"                  => "0.240"
        )
    )
  )
);

$hmac = calc_hmac($MERCHANT_SECRET_KEY, flatten_array($params));
$params["hmac"] = $hmac;
$json = json_encode($params);

?>
```
With merchant secret key `"AtSwv0AtTBd504p6iXB4JE1O"`, the HMAC value should be
`"ae8f89c22ce24107885411ba10ca7aee2f42788651514d778fac59db667f3bbb9b0e8bf8cf553`
`764b374350a03a159bedc893e06dd2f7fee104adef7ae97414d".`

# 3.0 Data Types

Here are the data types used in the other tables in this document.

Type | Description
---- | -----------
Boolean |	`1` means true, anything else means false. Note that optional boolean fields always default to false.
Date | Allowed formats are yyyy-MM-dd and dd.MM.yyyy
Integer |	Any signed 32-bit integer.
Text(`N`) |	An UTF-8 string of up to N characters.
Money |	An integer denoting an amount of money in the fractional unit of the currency (cents, pennies etc.).
Decimal(`M`,`D`) |	A string containing a decimal number, using the decimal point `"."` (not comma). `M` is the maximum number of digits in total. `D` is the maximum number of digits after the decimal point. For instance, the largest possible Decimal(10,3) value is `"9999999.999"`.
Currency |	An [ISO 4217](http://en.wikipedia.org/wiki/ISO_4217) three-letter currency code.
Identifier |	Alphanumeric ASCII string of up to 40 characters.
URL |	A valid URL, no more than 1000 characters long.
HMAC-SHA512 |	Exactly 40 upper case hexadecimal digits (0-9, A-F).
PostalAddress |	A custom type consisting of three `Text(N)` fields, which contain street address, postal code and city. Consult the [Address fields section](http://www.enterpay.fi/kehittajalle/#address_fields) for more information.
Date |	A date in format "yyyy-mm-dd".
