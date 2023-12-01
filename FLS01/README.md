# FLS01 Fiat On-ramp 


| Name    	| `fiat_on_ramp`             |
|---------- |------------------------------	|
| Version 	| 0.1                           |
| Status    | Draft                         |

## Motivation
The goal of this specification is to provide standardized API for applications to purchase bitcoin. This will make integrations simple and providers compatible, enabling wider adoption.

## Fiat to Bitcoin

### Wallet verification (optional)
Some jurisdictions require wallet verification by users so this spec supports it within the flow. To ensure user's ownership of the withdrawing wallet user can sign a message. Service provides the user with a message and user returns that message or part of the predefined message which the user then returns signed with his public key:
Example:
```
"node_pubkey": "02765a281bd188e80a89e6ea5092dcb8ebaaa5c5da341e64327e3fadbadcbc686c",
"message": "I confirm my bitcoin wallet. [7v7t4Fmb]",
"signature": "rywfek6717yuqpfpjmghyf173obgswr9uw5wbfsfhc8exjomftm71st4cyxprzkx5juxiokhbxm8rzkxoz8e3zmmpa644tudgt119s91",
```

This can be part of another API call not a standalone step.

### Order 
Order scenarios:
#### User wants to purchase X amount of BTC

Steps:
1) service needs to provide a quote for the user
2) user confirms the order
3) service provides payment methods
4) user needs to transfer fiat to the service
5) user withdraws the proceeds (amount based on agreed quote)

#### User wants to spend X amount of fiat

Steps:
1) services provides an estimate to the user
2) user confirms the order
3) service provides payment methods
4) user needs to transfer fiat to the service 
5) user withdraws the proceeds
### Withdrawal 
Service provides [lnurlw](https://github.com/lnurl/luds/blob/luds/03.md) to the user that user can claim at their convenience. Control of the payout is enforcable so only the pubkey who previously signed the wallet ownership verification message can be the recipient of the funds. 

Using lnurlw instead of invoices provided by the users addresses multiple potential issues:
1) expired invoices and thus failed payments
2) upfront commitment to payout amounts which means long lived quotes


### Units
All units in the spec are expressed in the smallest denomination - sats and cents.

## API endpoints draft

| Name      	 | function                                        | status | type   |
|----------------|-------------------------------------------------|--------|--------|
| /verify       | get secret to verify wallet ownership            | required | GET  |
| /session       | verify wallet ownership                         | required | POST |
| /quote         | get quote/estimate                              | required | POST |
| /order         | place order                                     | required | POST |
| /order-status  | get order status                                | required | POST |
| /withdrawal    | get lnurlw                                      |required  | POST |
| /payout        | get payout options                              | optional | GET  |
| /payment-options | get supported payment options  and currencies | required | GET  |
| /features | get supported features | required | GET  |

### features 
Return a list of supported features by the provider

| feature | function | status |
|---------|----------|---------|
| quotes | provider supports binding quotes | optional 
| estimates| provider supports non-binding estimates | optional 
| on_chain_fallback | provider supports on-chain fallback | optional
| webhook | provider supports webhook notifications | optional

Request:
```
GET /features
```
Response:
```
{
  "supported_features: [
    "quotes": true,
    "estimates":  true,
    "on_chain_fallback": false,
    "webhook": true
  ]
}
```



### verify
Request a token to be signed by the reciever node as proof of ownership 

Request:
```
GET /verify
```
Response:

```
{
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "token": "yyq6qpj2a",
  "expires_on": "2023-09-20T00:25:11.123Z"
}
```
- `token` (optional) random string from the provider that needs to be signed with the node pubkey in case wallet ownership proof is required. If token is not present then AOPP is not required.
- `session_id` uuid identifiying the client session 


### session
Start a session with optional signed proof of ownership. If Proof of Ownership is not required signature can be a random alphanumeric value.

Request:
```
POST /session


{
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "app_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "signature": "rdfe8mi98o7am51jpocda1zp5d8scdu7rg65nn73fs6mb69t4byer9xned1hntkeq1pqdct9z5owx6bg58w5fmny6p5q783dce8ittjh",
  "node_pubkey": "0288037d3f0bdcfb240402b43b80cdc32e41528b3e2ebe05884aff507d71fca71a"
}
```
Response:

```
{
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "app_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "expires_on": "2023-09-20T00:25:11.123Z"
}
```
- `signature ` token from `/verify` signed by the node. In case `token` is not present in the `/verify` response signature is a random alphanumeric value. 
- `app_id` app id (optional) 
- `node_pubkey` (optional) pubkey of a node that signed the `token`. If  `token` is not present in the `/verify` response `node_pubkey` is not needed 


### quote 
Get a an quote or estimate from the provider based on amount of fiat you want to spend 



Request:
```
# client wants to spend 1000chf
POST /quote   

{ 
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "amount_fiat": 100000, # in cents 
  "currency_id":1,
  "payment_option_id":1
}

# client wants to purchase 0.005btc
POST /quote   

{ 
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "amount_btc": 5000000 # in sats
  "currency_id":1,
  "payment_option_id":1
}
```

- `amount_fiat` (optional) amount of fiat the client wants to spend (unit cents)
- `amount_sats` (optional) amount of bitcoin the client wants to purchase (unit sats)
- `currency_id` is the fiat currency the client wants to be quoted in and will be used as payment
    - must one of the supported currencies from `/payment-options`
- `payment_option_id` needs to be one of `/payment-options` and needs to be provided at this step for the fee calculation

Response:

```
{
  "quote_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "amount_fiat": 100000, # in cents
  "currency_id": 1,
  "payment_option_id":1,
  "amount_sats" : 800000, # in sats
  "is_estimate" : false,
  "btc_price": 6942000, # in cents
  "order_fee": 1234, # in cents 
  "expires_on": "2023-09-20T00:25:11.123Z"
}
```
`quote_id` id of this quote which needs to be referenced while placing the order

`amount_sats` the amount of bitcoin the client will return for the fiat amount specified in the quote (unit sats)

`is_estimate` can be `true` or `false`,at discretion of the provider if he wants to provide a short duration binding quote or estimate

`order_fee` fee taken by the provider for quotes and estimates, in fiat (unit cents)

`btc_price` quoted or estimated price  (unit cents)

`expires_on` (optional) until when the payment for order needs to arrive for the quote to be honored

### order 
Confirm an order from quote and get payment information in return


Request:
```
POST /order

{
    "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
    "quote_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
    "webhook_url": "https://webhook.example.com/"

}

```
`webhook_url` (optional) url where the provider can send notifications to the user. Triggering the webhook is done by initiating a POST request with the following json payload:

`{ "type": <hook_type>, "data": <hook_data> }`

Example:
Order filled and ready for withdrawal

```
{
  "type": "withdrawal_ready",
  "data": {
    "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
    "order_status": "filled"
  }

}

```

Response:

```
{
  "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "order_status": "placed"
  "amount_fiat": 100000, # in cents
  "currency_id": 1,
  "payment_option_id":1,
  "amount_sats" : 800000, # in sats
  "expires_on": "2023-09-20T00:25:11.123Z",
  "payment_info": {
    "
  }
}
```
`order_id` must be valid [UUID 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random))

`order_status` can be `placed`, `filled`, `finished` or `refunded`

`expires_on` until when the payment needs to arrive for the order to be honored

`payment_info` returns the payment processing details with the following with one or both of the fields below
- `payment_url` (optional) url for payment through the providers website
- `payment_details` (optional) bank transfer payment details

Examples of :
```
# SEPA
"payment_info": {
    "payment_option_id":1, 
    "payment_details": {
      "provider_iban": "string", 
      "provider_name": "string", 
      "provider_address": "string",
      "provider_bank": "string",
      "provider_country": "string",
      "provider_bic": "string"
  }

# Credit Card
"payment_info": {
    "payment_option_id":3,
    "payment_url": "url"
  }


# Bank transfer
"payment_info": {
    "payment_option_id":5, 
    "payment_url": "url"
    "payment_details": {
      "provider_accountnumber": "string", 
      "provider_name": "string", 
      "provider_address": "string",
      "provider_bank": "string",
      "provider_country": "string",
      "provider_bic": "string"
  }
```

### order-status 
Get order status 

Request:
```
POST /order-status

{
    "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
    "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6"
}

```
Response:

```
{
  "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6": {
    "amount_fiat": 100000, # in cents
    "currency_id": 1,
    "payment_option_id":1,
    "amount_sats" : 800000, # in sats
    "btc_price": 6942000, # in cents
    "order_fee": 1234, # in cents
    "order_status": "finished",
    "order_status_date": "2023-09-20T00:25:11.123Z"
  },
    "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f0333a": {
    "amount_fiat": 100000, # in cents
    "currency_id": 1,
    "payment_option_id":1,
    "amount_sats" : 800000, # in sats
    "btc_price": 6942000, # in cents
    "order_fee": 1234, # in cents
    "order_status": "finished",
    "order_status_date": "2023-09-20T00:25:11.123Z"
  },
}
```
`order_id` (optional) must be valid [UUID 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random)). If no order_id is provided all orders belonging to the user are shown

`order_status`: can be 
 1) `placed` - status upon user confirmation of the quote / pending payment
 2) `filled` - status when order is executed and user can withdraw it
 3) `finished` - status when user successfully withdrew the funds
 4) `refunded` - status when fiat payment was refunded 

`order_status_date` datetime of when the status was made 
`order_fee` fee taken by the provider on the order 

### withdrawal 
Request lnurlw from the provider. User can provide optional fallback onchain address which will be used if the withdrawal is not claimed before the expiration date.

Request:
```
POST /withdrawal

{
    "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
    "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
    "failback_onchain": "bc1qcmu7kcwrndyke09zzyl0wv3dqxwlzqkma248kj" #optional

}

```
Response:

```
{
  "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "withdrawal_expiration_date": "2023-09-20T00:25:11.123Z"
  "lnurlw": "LNURL..."
}
```
`order_id` must be valid [UUID 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random))

`failback_onchain` valid bitcoin address

`withdrawal_expiration_date` datetime when the lnurlw will expire and in case of fallback provided funds will be sent onchain 


### payment-options
Get a list of supported currencies and their payment options 

Request:
```
POST /payment-options

{
    "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
    "currency_code": "<>"  # optional  (chf,eur, usd etc)
}

```
Response:

If no currency_code is specified in request:
```
{
  "currencies": [
    {
      "eur": {
        "currency_id": 1,
        "currency_code": "EUR",
        "payment_options": [
          {
            "option": "SEPA",
            "id": 1,
            "fee_rate": 0.005,
            "min_amount": 1000, # unit cents
            "max_amount": 100000 # unit cents
          },
          {
            "option": "SEPA Instant",
            "id": 2,
            "fee_rate": 0.01,
            "min_amount": 1000, # unit cents
            "max_amount": 100000 # unit cents

          },
          {
            "option": "Credit card",
            "id": 3,
            "fee_rate": 0.05,
            "min_amount": 2500, # unit cents
            "max_amount": 100000 # unit cents 
          }
        ]
      }
    },
    {
      "chf": {
        "currency_id": 2,
        "currency_code": "CHF",
        "payment_options": [
          {
            "option": "Bank transfer",
            "id": 5,
            "fee_rate": 0.01, 
            "min_amount": 10, # unit cents
            "max_amount": 100000 # unit cents
          },
          {
            "option": "Credit card",
            "id": 6,
            "fee_rate": 0.05,
            "min_amount": 10, #unit cents
            "max_amount": 100000 # unit cents

          }
        ]
      }
    }
  ]
}

```

If currency_code (in this case EUR) is specified in request:

```
{
  "currencies": [
    {
      "eur": {
        "currency_id": 1,
        "currency_code": "EUR",
        "payment_options": [
          {
            "option": "Revolut",
            "id": 1,
            "fee_rate": 0.01,
            "min_amount": 1000, # unit cents
            "max_amount": 100000 # unit cents
          },
          {
            "option": "Sepa Instant",
            "id": 2,
            "fee_rate": 0.01,
            "min_amount": 1000, # unit cents
            "max_amount": 100000 # unit cents
          },
          {
            "option": "Credit card",
            "id": 3,
            "fee_rate": 0.01,
            "min_amount": 1000, # unit cents
            "max_amount": 100000 # unit cents
          }
        ]
      }
    }
  ]
}

```
