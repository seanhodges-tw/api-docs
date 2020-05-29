# Webhook events

Events are messages that will be sent to your server as HTTP `POST` requests if you create a webhook subscription for
the type of event and a relevant resource you have access to is updated. 
Events will not contain any personally identifiable information.

For example, an event may describe a change in the status of a transfer you have made.

To acknowledge that you have successfully processed an event, make sure your server answers with a `2xx`-series HTTP status
code within 5 seconds. Otherwise, we will consider the delivery attempt as having failed and will later try to resend the
message.

We will attempt to redeliver messages at increasing intervals over a two week period. We will try at most 25 times to do this.


## Event HTTP requests

Event HTTP request bodies have a type-specific structure.
Events using version 2 of our type schema will contain a common base structure with additional event-specific details.
Each event type is described in detail later in this section.

Event HTTP requests also contain the following custom headers:


### Signature header `X-Signature`

> TransferWise public key for production environment:

```
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvO8vXV+JksBzZAY6GhSO
XdoTCfhXaaiZ+qAbtaDBiu2AGkGVpmEygFmWP4Li9m5+Ni85BhVvZOodM9epgW3F
bA5Q1SexvAF1PPjX4JpMstak/QhAgl1qMSqEevL8cmUeTgcMuVWCJmlge9h7B1CS
D4rtlimGZozG39rUBDg6Qt2K+P4wBfLblL0k4C4YUdLnpGYEDIth+i8XsRpFlogx
CAFyH9+knYsDbR43UJ9shtc42Ybd40Afihj8KnYKXzchyQ42aC8aZ/h5hyZ28yVy
Oj3Vos0VdBIs/gAyJ/4yyQFCXYte64I7ssrlbGRaco4nKF3HmaNhxwyKyJafz19e
HwIDAQAB
```
> TransferWise public key for sandbox environment:

```
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAwpb91cEYuyJNQepZAVfP
ZIlPZfNUefH+n6w9SW3fykqKu938cR7WadQv87oF2VuT+fDt7kqeRziTmPSUhqPU
ys/V2Q1rlfJuXbE+Gga37t7zwd0egQ+KyOEHQOpcTwKmtZ81ieGHynAQzsn1We3j
wt760MsCPJ7GMT141ByQM+yW1Bx+4SG3IGjXWyqOWrcXsxAvIXkpUD/jK/L958Cg
nZEgz0BSEh0QxYLITnW1lLokSx/dTianWPFEhMC9BgijempgNXHNfcVirg1lPSyg
z7KqoKUN0oHqWLr2U1A+7kqrl6O2nx3CKs1bj1hToT1+p4kcMoHXA7kA+VBLUpEs
VwIDAQAB
```

> How to verify signatures (in Java):

```java
public boolean verifySignature(String encodedPublicKey, String signature, String payload) {
  X509EncodedKeySpec publicKeySpec = new X509EncodedKeySpec(Base64.getMimeDecoder().decode(encodedPublicKey));
  KeyFactory keyFactory = KeyFactory.getInstance("RSA");
  PublicKey publicKey = keyFactory.generatePublic(publicKeySpec);
  
  Signature sign = Signature.getInstance("SHA1WithRSA");
  sign.initVerify(publicKey);
  sign.update(payload.getBytes());
  
  byte[] signatureBytes = Base64.getDecoder().decode(signature);
  
  return sign.verify(signatureBytes);
}
```

Each outgoing webhook request is signed.
Although event payloads do not contain any sensitive information, you may want to
verify if the request is coming from TransferWise (however this is optional).
We advise you not to process any requests where signature appears to be forged.

Each `POST` request includes the `X-Signature` header, which contains the message-specific signature.


### Delivery ID header `X-Delivery-Id`

Each outgoing notification is assigned a unique delivery UUID.


### Test notification header `X-Test-Notification`

This header is present with the value `true` if the notification is a test message.

Test messages can be sent to verify callback URLs when subscriptions are being set up.


## Event payload

All event notification payloads have the same high-level structure.
Top-level properties are common to all events.
The `data` property is an object that can contain various properties.
The exact properties that the `data` object contains depends on the event type and schema version of the event.  


> Basic event payload:

```json
{
  "data": {},
  "subscription_id": "01234567-89ab-cdef-0123-456789abcdef",
  "event_type": "event#type",
  "schema_version": "2.0.0",
  "sent_at": "2020-01-01T12:34:56Z"
}
```

### Common properties

Field           | Description                                                                   | Format
---------       | -------                                                                       | -----------
data            | Event type- and schema version-specific details                               | Object
subscription_id | ID of the webhook subscription that triggered the event notification          | UUID
event_type      | Event type (what event happened in our system)                                | String
schema_version  | Schema version (what notification structure is being used to model the event) | String
sent_at         | When the event notification was sent from our system                          | Datetime


## Transfer status change event

Event type: `transfers#state-change`

This event will be triggered every time a transfer's status is updated. Each event contains a timestamp.
As we do not guarantee the order of events, please use `data.occurred_at` to reconcile the order. 

If you would like to subscribe to transfer state change events, please use `transfers#state-change`
when creating your subscription.


> Example v2.0.0 `transfers#state-change` event:

```json
{
  "data": {
    "resource": {
      "type": "transfer",
      "id": 111,
      "profile_id": 222,
      "account_id": 333
    },
    "current_state": "processing",
    "previous_state": "incoming_payment_waiting",
    "occurred_at": "2020-01-01T12:34:56Z"
  },
  "subscription_id": "01234567-89ab-cdef-0123-456789abcdef",
  "event_type": "transfers#state-change",
  "schema_version": "2.0.0",
  "sent_at": "2020-01-01T12:34:56Z"
}
```

### Schema version `2.0.0` (latest)

Field                       | Description                                   | Format
---------                   | -------                                       | -----------
data.resource.type          | Transfer resource type (always `transfer`)    | String
data.resource.id            | ID of the transfer                            | Integer
data.resource.profile_id    | ID of the profile that owns the transfer      | Integer
data.resource.account_id    | ID of transfer's recipient account            | Integer
data.current_state          | Current transfer state (see [transfer statuses](#payouts-guide-track-transfer-status)) | String
data.previous_state         | Previous transfer state (see [transfer statuses](#payouts-guide-track-transfer-status)) | String
data.occurred_at            | When the state change occurred                | Datetime


## Transfer active cases event

Event type: `transfers#active-cases`

This event will be triggered every time a transfer's list of active cases is updated.
Active cases indicate potential problems with transfer processing. 

If you would like to subscribe to transfer active cases events, please use `transfers#active-cases` when creating
your subscription.


> Example v2.0.0 `transfers#active-cases` event:

```json
{
  "data": {
    "resource": {
      "type": "transfer",
      "id": 111,
      "profile_id": 222,
      "account_id": 333
    },
    "active_cases": ["deposit_amount_less_invoice"]
  },
  "subscription_id": "01234567-89ab-cdef-0123-456789abcdef",
  "event_type": "transfers#active-cases",
  "schema_version": "2.0.0",
  "sent_at": "2020-01-01T12:34:56Z"
}
```

### Schema version `2.0.0` (latest)

Field                       | Description                                   | Format
---------                   | -------                                       | -----------
data.resource.type          | Transfer resource type (always `transfer`)    | String
data.resource.id            | ID of the transfer                            | Integer
data.resource.profile_id    | ID of the profile that owns the transfer      | Integer
data.resource.account_id    | ID of transfer's recipient account            | Integer
data.active_cases           | Ongoing issues related to the transfer        | List of strings


## Balance credit event

Event type: `balances#credit`

This event will be triggered every time a balance account is credited.

If you would like to subscribe to balance credit events, please use `balances#credit` when creating your subscription.

Please note: This event is not currently delivered to application subscriptions.


> Example v2.0.0 `balances#credit` event:

```json
{
  "data": {
    "resource": {
      "type": "balance-account",
      "id": 111,
      "profile_id": 222
    },
    "transaction_type": "credit",
    "amount": 1.23,
    "currency": "EUR",
    "post_transaction_balance_amount": 2.34,
    "occurred_at": "2020-01-01T12:34:56Z"
  },
  "subscription_id": "01234567-89ab-cdef-0123-456789abcdef",
  "event_type": "balances#credit",
  "schema_version": "2.0.0",
  "sent_at": "2020-01-01T12:34:56Z"
}
```

**Schema version `2.0.0` (latest)**

Field                       | Description                                                   | Format
---------                   | -------                                                       | -----------
data.resource.type          | Balance account resource type (always `balance-account`)      | String
data.resource.id            | ID of the balance account                                     | Integer
data.resource.profile_id    | ID of the profile that owns the balance account               | Integer
data.transaction_type       | Always `credit`                                               | String
data.amount                 | Deposited amount                                              | Decimal
data.currency               | Currency code                                                 | String
data.post_transaction_balance_amount | Balance amount after the credit was applied          | Decimal
data.occurred_at            | When the balance credit occurred                              | Datetime
