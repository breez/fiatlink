# FLS01 Fiat On-ramp  2


| Name    	| `fiat_on_ramp`             |
|---------- |------------------------------	|
| Version 	| 0.1                           |
| Status    | Draft                         |

## Motivation
The goal of this specification is to provide standardized API for applications to purchase bitcoin. This will make integrations simple and providers compatible, enabling wider adoption.

## Fiat to Bitcoin

### Wallet verification 
To ensure user's ownership of the withdrawing wallet user must sign a message. Service provides the user with a message and user returns that message or part of the predefined message which the user then returns signed with his public key:
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
4) user needs to transfer fiat to the service (within specific amount of time?)
5) user withdraws the proceeds (amount based on agreed quote)

#### User wants to spend X amount of fiat

Steps:
1) services provides an estimate to the user
2) user confirms the order
3) service provides payment methods
4) user needs to transfer fiat to the service 
5) user withdraws the proceeds (amount based on execution at the time)
### Withdrawal 
Service provides [lnurlw](https://github.com/lnurl/luds/blob/luds/03.md) to the user that user can claim at their convenience. Control of the payout is enforcable so only the pubkey who previously signed the wallet ownership verification message can be the recipient of the funds. 

Using nurlw instead of invoices provided by the users addresses multiple potential issues:
1) expired invoices and thus failed payments
2) upfront commitment to payout amounts which means long lived quotes


Alternative options:
1) user provides an invoice for the quoted amount
2) user provides pubkey and provider opens a channel and pushes the amount (can be used as a backup for 1.)


## API endpoints draft

| Name      	 | function                                        | status | type   |
|----------------|-------------------------------------------------|--------|--------|
| /verify       | get secret to verify wallet ownership            | required | GET  |
| /auth          | verify wallet ownership                         | required | POST |
| /quote         | place order                                     | required | POST |
| /order         | place order                                     | required | POST |
| /orders        | get order status                                | required | GET  |
| /withdrawal    | get lnurlw                                      |required  | POST |
| /payout        | get payout options                              | optional | GET  |
| /payment-options | get supported payment options  and currencies | required | GET  |

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
- `token` random string from the provider that needs to be signed with the node pubkey
- `session_id` uuid identifiying the client session 


### auth
Start a session with signed proof of ownership 

Request:
```
POST /auth


{
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "signature": "rdfe8mi98o7am51jpocda1zp5d8scdu7rg65nn73fs6mb69t4byer9xned1hntkeq1pqdct9z5owx6bg58w5fmny6p5q783dce8ittjh",
}
```
Response:

```
{
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "expires_on": "2023-09-20T00:25:11.123Z"
}
```
- `signature ` token from `/verify` signed by the node 
- `id` client id (optional) 


### quote 
Get a an quote or estimate from the provider based on amount of fiat you want to spend 



Request:
```
POST /quote

{ 
  "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
  "amount_fiat": 1000,
  "currency_id":1,
  "payment_option_id":1
}

```

- `amount_fiat` what the client wants to spend
    - must be greater than 0 and less than 1000
- `currency_id` is the fiat currency the client wants to be quoted in and will be used as payment
    - must one of the supported currencies from `/payment-options`

Response:

```
{
  "quote_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "amount_fiat": "1000",
  "currency_id": 1,
  "payment_option_id":1,
  "amount_sats" : 800000 ,
  "expires_on": "2023-09-20T00:25:11.123Z"
}
```
- `amount_sats` the amount of bitcoin the client will return for the fiat amount specified in the quote
- `expires_on` until when the order needs to arrive for the quote to be honored
### order 
Confirm an order from quote and get payment information in return


Request:
```
POST /order

{
    "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
    "quote_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6"

}

```
Response:

```
{
  "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "order_status": "placed"
  "amount_fiat": "1000",
  "currency_id": 1,
  "payment_option_id":1,
  "amount_sats" : 800000 ,
  "expires_on": "2023-09-20T00:25:11.123Z",
  "payment_info": {
    "
  }
}
```
`order_id` must be valid [UUID 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random))

`order_status` can be `placed`, `filled`, `finished` or `refunded`

`payment_info` returns the payment processing details

`expires_on` until when the payment needs to arrive for the order to be honored

### orders 
Get order status 

Request:
```
POST /order

{
    "session_id": "d7ef9a88-1ca1-4ac8-bc9e-da3d9824cdc5",
    "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6"
}

```
Response:

```
{
  "order_id": "8ed13c2a-a8c6-4f0e-b43e-3fdbf1f094a6",
  "amount_fiat": "1000",
  "currency_id": 1,
  "payment_option_id":1,
  "amount_sats" : 800000 ,
  "order_status": "finished"
  "order_status_date": "2023-09-20T00:25:11.123Z"
}
```
`order_id` must be valid [UUID 4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random))

`order_status`: can be 
 1) `placed` - status upon user confirmation of the quote / pending payment
 2) `filled` - status when order is executed and user can withdraw it
 3) `finished` - status when user successfully withdrew the funds
 4) `refunded` - status when fiat payment was refunded 

`order_status_date` datetime of when the status was made 

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
            "min_amount": 10,
            "max_amount": 1000
          },
          {
            "option": "SEPA Instant",
            "id": 2,
            "fee_rate": 0.01,
            "min_amount": 10,
            "max_amount": 1000

          },
          {
            "option": "Credit card",
            "id": 3,
            "fee_rate": 0.05,
            "min_amount": 10,
            "max_amount": 1000
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
            "id": 1,
            "fee_rate": 0.01,
            "min_amount": 10,
            "max_amount": 1000
          },
          {
            "option": "Credit card",
            "id": 2,
            "fee_rate": 0.05,
            "min_amount": 10,
            "max_amount": 1000

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
            "min_amount": 10,
            "max_amount": 1000
          },
          {
            "option": "Sepa Instant",
            "id": 2,
            "fee_rate": 0.01,
            "min_amount": 10,
            "max_amount": 1000
          },
          {
            "option": "Credit card",
            "id": 3,
            "fee_rate": 0.01,
            "min_amount": 10,
            "max_amount": 1000
          }
        ]
      }
    }
  ]
}

```
