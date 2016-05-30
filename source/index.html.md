---
title: API Reference

language_tabs:
  - shell
  - ruby
  - python
  - html

toc_footers:
  - <a href='#'>Sign Up for a Developer Key</a>
  - <a href='https://github.com/tripit/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Enterpay Payment Integration Guide

Enterpay's payment button enables your customers to pay for their purchases by invoice. This is especially convenient for corporate customers.

The button is very similar technically to most bank payment buttons. This document explains how to integrate the button to your site.

##Tutorial

Integrating the Enterpay payment button consists of the following steps.

1. Sign a contract with Enterpay and obtain a public merchant ID and a secret API key.
2. Write code to show the payment button (an HTML form with hidden fields).
3. Write a return page that receives the user after the purchase completes or fails.

You can freely test against the dev installation at [test.laskuyritykselle.fi](https://test.laskuyritykselle.fi/).

##HTML form

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


## Return page

The user will be redirected to the page at `url_return`. Some [GET parameters](http://www.enterpay.fi/kehittajalle/#returned_parameters) that get added to the return URL. Here is an example:

`?version=1&status=successful&identifier_valuebuy=123456789abcdef&identifier_merchant=abc123&key_version=1&hmac=F9521CE36134BD5DBFB6D9C....`

If you want to mark the purchase as verified as soon as the user arrives to the return page, then you **must** verify the `hmac`. Otherwise an attacker can skip the payment process and go directly to the return page. Remember to also check that `status` is `successful`.

It is also recommended to make the return page *idempotent* i.e. such that it works even if the user navigates to it twice or refreshes the page.

## Miscellaneous

* An explanation of how the total price is calculated and how rounding is done is [here](http://www.enterpay.fi/kehittajalle/#rounding).
* The production installation is at [laskuyritykselle.fi](https://laskuyritykselle.fi/)
* The test installation is at [test.laskuyritykselle.fi](https://test.laskuyritykselle.fi/)


# Reference

## Starting a payment

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

# Kittens

## Get All Kittens

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get()
```

```shell
curl "http://example.com/api/kittens"
  -H "Authorization: meowmeowmeow"
```

> The above command returns JSON structured like this:

```json
[
  {
    "id": 1,
    "name": "Fluffums",
    "breed": "calico",
    "fluffiness": 6,
    "cuteness": 7
  },
  {
    "id": 2,
    "name": "Max",
    "breed": "unknown",
    "fluffiness": 5,
    "cuteness": 10
  }
]
```

This endpoint retrieves all kittens.

### HTTP Request

`GET http://example.com/api/kittens`

### Query Parameters

Parameter | Default | Description
--------- | ------- | -----------
include_cats | false | If set to true, the result will also include cats.
available | true | If set to false, the result will include kittens that have already been adopted.

<aside class="success">
Remember — a happy kitten is an authenticated kitten!
</aside>

## Get a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.get(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.get(2)
```

```shell
curl "http://example.com/api/kittens/2"
  -H "Authorization: meowmeowmeow"
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "name": "Max",
  "breed": "unknown",
  "fluffiness": 5,
  "cuteness": 10
}
```

This endpoint retrieves a specific kitten.

<aside class="warning">Inside HTML code blocks like this one, you can't use Markdown, so use <code>&lt;code&gt;</code> blocks to denote code.</aside>

### HTTP Request

`GET http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to retrieve
