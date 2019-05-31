---
group: graphql
title: Paypal endpoint
---

The `Paypal` endpoint provides support for the following PayPal payment methods:

* PayPal Express Checkout
* PayPal Payflow Pro and PayPal Payflow Link

The following steps describe the flow of calls required to complete a typical PayPal checkout transaction. A successful purchase requires that you send three mutations to PayPal, and the buyer must approve the purchase by logging in to PayPal.

1. **Send a token request.** When the buyer clicks a PayPal button, the frontend executes the `createPaypalExpressToken` mutation. The Magento `PaypalGraphQl` module gathers information in the specified cart and sends this information to PayPal.

2. **PayPal returns a token.** If the token request succeeds, PayPal returns a token and a payer ID. PayPal also sends payment-related data that is outside the scope of GraphQL. You must include this token in subsequent steps. The buyer is redirected to the payment confirmation page, which was specified in the token request.

3. **Redirect the customer to PayPal for approval.** Depending on your implementation, the buyer is either redirected to the PayPal login screen, or the buyer enters their credentials in-context. 

4. **PayPal redirects the customer back to your site.** If the customer approves the payment, PayPal directs the customer back to the payment confirmation page.

5. **Set the payment method.** The frontend runs the `setPaymentMethodOnCart` mutation. The payload includes the PayPal token and the payer ID. The cart may have been updated since the token was requested with shipping costs, taxes, and other adjustments. Magento submits the updated cart to PayPal.

6. **Complete the transaction.** Place the order with the `placeOrder` mutation. PayPal captures the payment by transferring the funds from the customer account to the appropriate merchant account. Magento creates an order, ready for fulfillment.

## Mutation

The `createPaypalExpressToken` mutation initiates a PayPal checkout transaction and receives a token. You must specify this token when setting a PayPal payment method.

## Syntax

`createPaypalExpressToken(input: PaypalExpressTokenInput): PaypalExpressToken`

## Examples

These examples show all the mutations required to complete a PayPal purchase.

### Request a PayPal token

The PayPal `token` will be used in the other mutations. The raw response from PayPal also includes the payer ID in the URL. Extract the payer ID so that it used in the mutation that sets the payment method.

**Request**

```text
mutation {
    createPaypalExpressToken(
        input: {
            cart_id: "rMQdWEecBZr4SVWZwj2AF6y0dNCKQ8uH"
            code: "paypal_express"
            express_button: true
            urls: {
                return_url: "http://magento.test/paypal/express/return/"
                cancel_url: "http://magento.test/paypal/express/cancel/"
            }
        }
    )
    {
        token
        paypal_urls{
            start
            edit
        }
    }
}
```

**Response**

```json
{
  "data": {
    "createPaypalExpressToken": {
      "token": "<PayPal_Token>"
      "paypal_urls": {
        "start": "https://www.sandbox.paypal.com/checkoutnow?token=<PayPal_Token>"
        "edit": "https://www.sandbox.paypal.com/cgi-bin/webscr?cmd=_express-checkout&useraction=continue&token=<PayPal_Token>"
      }
    }
  }
}
```

### Set the payment method

Magento GraphQL supports the `paypal_express` and `paypal_payflow` payment methods.

**Request**

```text
mutation {
  setPaymentMethodOnCart(input: {
    cart_id: "rMQdWEecBZr4SVWZwj2AF6y0dNCKQ8uH"
    payment_method: {
        code: "paypal_express"
        additional_data: {
            paypal_express: {
                payer_id: "<PayPal_PayerID>"
                token: "<PayPal_Token>"
            }
        }
      }
  }) {
    cart {
      selected_payment_method {
        code
        title
      }
    }
  }
}
```

**Response**

```json
{
  "data": {
    "setPaymentMethodOnCart": {
      "cart": {
        "selected_payment_method": {
          "code": "paypal_express",
          "title": "PayPal Express Checkout"
        }
      }
    }
  }
}
```

### Place the order

**Request**

``` text
mutation {
  placeOrder(
    input: {
      cart_id: "rMQdWEecBZr4SVWZwj2AF6y0dNCKQ8uH"
    }
  ) {
    order {
      order_id
    }
  }
}
```

**Response**

```json
{
  "data": {
    "placeOrder": {
      "order": {
        "order_id": "000000006"
      }
    }
  }
}
```

## Input attributes

### PaypalExpressTokenInput {#PaypalExpressTokenInput}

The `PaypalExpressTokenInput` object defines the attributes required to receive a payment token from PayPal.

Attribute |  Data Type | Description
--- | --- | ---
`cart_id` | String! | The unique ID that identifies the customer's cart
`code` | String! | Payment method code
`express_button`: | Boolean | Indicates whether the buyer selected the PayPal Express Checkout button. The default value is `false`
`urls` | [`PaypalExpressUrlsInput`](#PaypalExpressUrlsInput) | Defines a set of URLs to redirect to in response to the token request
`use_paypal_credit` | Boolean | Indicates whether the buyer clicked the Paypal credit button. The default value is `false`

### PaypalExpressUrlsInput {#PaypalExpressUrlsInput}

The `PaypalExpressUrlsInput` object contains a set of URLs that PayPal uses to respond to a token request.

Attribute |  Data Type | Description
--- | --- | ---
`cancel_url` | String! | The redirect URL when the buyer cancels the transaction. This should be the page on your website where the buyer initially chose PayPal as the payment type
`pending_url` | String! | The URL to redirect for a pending transactions. Not applicable to most PayPal solutions
`return_url` | String! | The URL of the final review page on your website where the buyer confirms the order and payment
`success_url` | String! | The URL to redirect upon success. Not applicable to most PayPal solutions

### PaymentMethodAdditionalDataInput {#PaymentMethodAdditionalDataInput}

The `PaymentMethodAdditionalDataInput` data type attributes are used when setting a PayPal payment method. [setPaymentMethodOnCart mutation]({{ page.baseurl}}/graphql/reference/quote-payment-method.html) provides more context.

Attribute |  Data Type | Description
--- | --- | ---
`payflow_express` | [PayflowExpressInput](#PayflowExpressInput) | Required input for PayPal Payflow Express Checkout payments
`paypal_express` | [PaypalExpressInput](#PaypalExpressInput) | Required input for PayPal Express Checkout payments

### PaypalExpressInput {#PaypalExpressInput}

The `PaypalExpressInput` object contains the required input for PayPal Express Checkout payments.

Attribute |  Data Type | Description
--- | --- | ---
`payer_id` | String! | The unique ID of the PayPal customer
`token` | String! | The token returned by the createPaypalExpressToken mutation

### PayflowExpressInput {#PayflowExpressInput}

The `PayflowExpressInput` object contains the required input for PayPal Payflow Express Checkout payments

Attribute |  Data Type | Description
--- | --- | ---
`payer_id` | String! | The unique ID of the PayPal user
`token` |  String! | The token returned by the createPaypalExpressToken mutation

## Output attributes

### PaypalExpressToken {#PaypalExpressToken}

The `PaypalExpressToken` object contains a token returned by PayPal and a set of URLs that allow the buyer to authorize payment and adjust checkout details.

Attribute |  Data Type | Description
--- | --- | ---
`paypal_urls` | [PaypalExpressUrlList](#PaypalExpressUrlList) | A set of URLs that allow the buyer to authorize payment and adjust checkout details
`token` | String | The token returned by PayPal

### PaypalExpressUrlList {#PaypalExpressUrlList}

The `PaypalExpressUrlList` object defines a set of URLs that allow the buyer to authorize payment and adjust checkout details.

Attribute |  Data Type | Description
--- | --- | ---
`edit` | String | The PayPal URL that allows the buyer to edit their checkout details
`start` | String | The URL to the PayPal login page
