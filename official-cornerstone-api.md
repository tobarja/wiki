The Cornerstone Dashboard API
=============================

Introduction:

- [Endpoints](#endpoints)
- [Parameters](#parameters)
- [Authenication](#authentication)
- [HTTP Verbs](#http-verbs)
- [Errors](#errors)

Usage:

- [API Version](#api-version)
- [Transactions](#transactions)
- [Fetch Transactions](#fetch-transactions)
- [Update Schedules](#update-schedule)
- [Payment Information Vault](#payment-information-vault)
- [Merchant Application Status](#merchant-applications-status)
- [Tenants (Customers)](#tenants)

# Introduction 

The Cornerstone API is a [REST](http://en.wikipedia.org/wiki/REST) API secured with SSL that accepts parameters as HTTP GET and POST fields, and returns information as [JSON](http://en.wikipedia.org/wiki/JSON). 

<!--
Inspiration for API:
[Apache CouchDB](http://en.wikipedia.org/wiki/CouchDB#Accessing_data_via_HTTP),
[Google Fusion Tables](https://developers.google.com/fusiontables/docs/v1/using),
[Github API](http://developer.github.com/).
-->

## Endpoints

The main endpoint for the API is

    https://api.cornerstone.cc/v1

In the event of a network fault, it is also possible to try the alternative endpoint. (This endpoint routes into the API from a different IP address, registrar, DNS provider, and SSL authority.)

    https://cornerstone2.cc/api/v1

There is also a legacy sandbox endpoint, which will be completely removed within the next couple of years. Please note that you will need separate credentials to access this endpoint. It also also not recommended to send any live or confidential information, only testing data. 

    http://api.cornerstone.cc:8080/v1


## Authentication

Authentication is provided by [HTTP Basic Authentication](http://tools.ietf.org/html/rfc2617#section-2), using a provided API client ID and client key. With the [curl](http://curl.haxx.se/) utility, it looks something line this:

    $ curl -i -u <client_id>:<client_key> <resource_url>

## Parameters

For GET requests, all parameters not included in the resource path can be accessed as an HTTP query string parameter:

    $ curl -i -u client_id:client_key "https://api.cornerstone.cc/v1/transactions?range=*&show_test"

Other requests are placed in the body of the request as POST parameters:

    $ curl -i -u client_id:client_key -X POST https://api.cornerstone.cc/v1/transactions -d \
      "amount=15&card[number]=4444333322221111&card[expmonth]=12&card[expyear]=23\
       &customer[firstname]=Robert&customer[lastname]=Parr&customer[email]=robertp@example.com"

## HTTP Verbs

- **GET** - Retrieve a resource
- **POST** - Create a new resource
- **PATCH** - Partially update an existing resource
- **PUT** - Replace a resource with new data
- **COPY** - Duplicate a resource, with any changes included as parameters
- **DELETE** - Delete a resource

## Errors

There are three types of errors our API will give you:

- `401``auth_error`: Thrown in two circumstances. When client ID and key are missing or unmatched, and when the client doesn't have permissions to perform certain API actions.
- `400``bad_request`: When arguments are missing or do not match a specific pattern.
- `404``not_found`: If a resource referred to by its identifier is missing, or a client lacks permission to view it.
- `500``internal`: Any internal problem. Please [contact us](mailto:rm@cornerstone.cc) right away if you experience one of these. They are very rare and can be dangerous if left in the wild.

All errors return as a Json object, and contain at least two properties: `error` and `reason`. Often a `try` property is also given, to hint at possible next steps. When applicable, a `user_message` is also sent, which is always a safe message to display directly to users.

### Example Errors

```json
{
	"error": "auth_error",
	"reason": "Could not find a matching key and ID",
	"try": "Double-checking your credentials"
}
```

```json
{
	"error": "not_found",
	"reason": "No application found by ID: xyz"
}
```

```json
{
	"error": "not_found",
	"reason": "No resource URI: xyz"
}
```

# API Version

    GET https://api.cornerstone.cc/v1/

```json
HTTP/1.1 200 OK

{
	"cornerstone": "Howdy there!",
	"version": "1.0",
	"documentation": "https://github.com/cpscc/wiki/blob/master/official-cornerstone-api.md"
}
```


# Transactions

    POST https://api.cornerstone.cc/v1/transactions
    GET https://api.cornerstone.cc/v1/transactions
    GET https://api.cornerstone.cc/v1/transactions/<transaction_id>

## Create (charge) a transaction

    POST https://api.cornerstone.cc/v1/transactions

### Testing Credentials
Card | Securenet | Sage
---- | ----- | -----
number | 4444333322221111 | 4111111111111111
expmo | 12 | 12
expyr | 24 | 24
cvv | 123 | 123

eCheck | Securenet | Sage
---- | ----- | -----
account | 9999999999 | 056008849
routing | 031100393 | 12345678901234

### Parameters

Name | Usage
---- | -----
request_id | (optional, recommended) A unique id string (max. length 255) sent along with the transaction request. If the transaction request is re-sent, the `request_id` will be checked for uniqueness, and if it is found, a `request_id_conflict` error will be sent with a `400` response code. Please see the notes and examples on this below.
amount | Amount in US dollars. We try to determine what you mean automatically, so `13`, `13.00`, `$13`, and `13 dollars` all register as $13.00 USD.
merchant | (optional) If the transaction is being charged to another merchant or a sub-account, it is specified here.
recurring | (optional) Allows you to specify a recurring cycle. Values available: `once` (default), `weekly`, `monthly`, `quarterly`, or `yearly`. Without `startdate` set, all monthly transactions made on the 13th will fall on the 13th of the next month, and so on.
start-date | (optional) Used to schedule a transaction in the future. Must be formatted: `mm/dd/yyyy`, e.g. `12/31/1999`. If the day of the month is above 30 (as it is in our example), it is silently shifted down to 30.
memo | (optional) Can contain any string of text.
vault | Vault the payment info -- this results in a 0 transaction record, where no authorization or capture has been made on for the payment
card[] | contains: `card[number]`, `card[expmonth]`, `card[expyear]` and `card[cvv]`
check[] | only required if `card[]` is missing, contains: `check[aba]`, `check[account]` and `check[type]`. `type` can be one of `savings`, `checking`, `bsave`or `bcheck`.
customer[] | Customer billing information. `customer[firstname]`, `customer[lastname]`, and `customer[email]` are required.

For more details, see "Parameter Details" below.

### Examples

Approved CC transaction:

    POST https://api.cornerstone.cc/v1/transactions

```yaml
request_id: abc123
amount: 15
customer[firstname]: Bob
customer[lastname]: Parr
customer[email]: robertp@example.com
card[number]: 4444333322221111
card[expmonth]: 12
card[expyear]: 21
card[cvv]: 123
```

```json
HTTP/1.1 200 OK

{
    "approved": [
        {
            "id": 8984,
            "merchant": "Test Ministry",
            "reason": "Accepted",
            "amount": "15",
            "frequency": "once",
            "startdate": false,
            "test": false,
            "token": "visa.1111.1221.NTAwMzc1MC0yMw=="
        }
    ]
}
```

`request_id` conflict:

    POST https://api.cornerstone.cc/v1/transactions

```yaml
request_id: abc123
amount: 15
customer[firstname]: Bob
customer[lastname]: Parr
customer[email]: robertp@example.com
card[number]: 4444333322221111
card[expmonth]: 12
card[expyear]: 21
card[cvv]: 123
```

```json
HTTP/1.1 400 Bad Request

{
    "error": "bad_request",
    "reason": "request_id already exists: abc123",
    "request_id_conflict": "abc123",
    "transaction_ids": [
        "8984"
    ]
}
```

Declined transaction:

    POST https://api.cornerstone.cc/v1/transactions

```yaml
amount: 15
customer[firstname]: Bob
customer[lastname]: Parr
customer[email]: robertp@example.com
card[number]: 1234123412341234
card[expmonth]: 12
card[expyear]: 21
card[cvv]: 123
```

```json
HTTP/1.1 200 OK

{
    "declined": [
        {
            "reason": "CARD TYPE COULD NOT BE IDENTIFIED"
        }
    ]
}
```

Approved EFT Transaction:


    POST https://api.cornerstone.cc/v1/transactions

```yaml
amount: 15
customer[firstname]: bob
customer[lastname]: parr
customer[email]: robertp@insuricare.com
check[aba]: 031100393
check[account]: 9999999999
check[type]: checking
```

```json
HTTP/1.1 200 OK

{
    "approved": [
        {
            "amount": 15,
            "frequency": "once",
            "id": 1234
        }
    ]
}
```

EFT not enabled:

    POST https://api.cornerstone.cc/v1/transactions

```yaml
amount: 15
customer[firstname]: bob
customer[lastname]: parr
customer[email]: robertp@insuricare.com
check[aba]: 031100393
check[account]: 9999999999
check[type]: checking
```

```json
HTTP/1.1 200 OK

{
    "declined": [
        {
            "reason": "MERCHANT IS NOT ENABLED FOR ACH TRANSACTIONS"
        }
    ]
}
```

No fields given:

    POST https://api.cornerstone.cc/v1/transactions

```json
HTTP/1.1 400 Bad Request

{
    "error": "bad_request",
    "reason": "Missing required field: amount"
}
```

Missing fields:

    POST https://api.cornerstone.cc/v1/transactions

```yaml
amount: 15
customer[firstname]: bob
customer[lastname]: parr
customer[email]: robertp@insuricare.com
```

```json
HTTP/1.1 400 Bad Request

{
    "error": "bad_request",
    "reason": "Missing required fields: a card or check is required."
}
```

### Parameter Details

#### card[]

(required if `check` is missing)

* `card[number]` Credit card number. Must contain 15-16 digits.
* `card[expmonth]` Credit card expiration month. Must contain 2 digits between 1 and 12.
* `card[expyear]` Credit card expiration year. Must contain 2 digits, later than the current year.
* `card[cvv]` Card security code (CVV/CVC). Must contain 3-4 digits.

#### check[] - EFT / E-check

(required if `card` is missing)

* `check[aba]` 9-digit bank routing number.
* `check[account]` Bank account number.
* `check[type]` Bank account type. Can be one of: `savings`, `checking`, `bsave` (business savings) or `bcheck` (business checking).

#### customer[]

The following parameters are required and must contain at least two charactars: `customer[firstname]`, `customer[lastname]`, and `customer[email]`. Note: A valid email address for the cardholder. This will be used for receipts, and possibly identification, later.

The following parameters make up the billing address and may or may not be required depending on the setting for the merchant.

* `customer[address]`: Address line 1
* `customer[company]`: Address line 2
* `customer[city]`:
* `customer[state]`
* `customer[zip]`
* `customer[country]` (optional)
* `customer[phone]` (optional)


#### customer[comment]

`customer[comment]` is used for notes or comments from the customer. This is different than the memo.

#### memo[]
The memo parameter can be used to send various information.
i.e. if a merchant wanted to post an invoice number they would pass it as `memo[Invoice]` with the attached value.


### Request ID

The `request_id` parameter is provided to allow re-sending a transaction request in the case of network faults or 
timeouts, without the risk of a duplicate transaction processing. A `request_id` must be unique within the system,
and it is recommended that you use the full 255 characters of space to avoid any "false positives" with the conflicts. An example of one way you may construct a `request_id`
(although the format is completely up to you):

    <ORGANIZATION NAME>.<TIMESTAMP>.<HASH OF REQUEST>.<RANDOM DATA>
    
So, for example, given your organization is called "Orient Missions", you could have a `request_id` that looks like the following:

    ORIENT MISSIONS.1454961474.21002edf4b6627c85aece64f669b63a905819b43.R|yw*ZOmqaZ42(%z^n8= ... (truncated)

Or, it is possible to simply used an entirely random string:

    tg7cVcPg[u5a+AX/rG33uGEGzff6LdPyoFbOeTn+%d2+pBGy0E@mdEtf]%zidZzV7di6_3|mM3MJ!jlwn7-4NMCA ... (truncated)


## Update-Schedule

    PATCH https://api.cornerstone.cc/v1/transactions
    
Update scheduled transaction information to be processed at a future date.

To update the transaction you must refer to a certain transaction ID.  These ID's are returned when a schedule is made or you can fetch the transaction ID's using the [Fetch Transactions](#fetch-transactions) functionality.  If card of check information is passed it will return the updated token related to the transaction.  You made do a simple update without any card or check information such as adjusting the amount, token, cycle, startdate, or nextdate of a transaction.

### Parameters

Name | Usage
---- | -----
id | The ID of the transaction you would like to update. If the transaction cannot be found, or you do not have access, a "not_found" error will be returned.
amount | Amount in US dollars. We try to determine what you mean automatically, so `13`, `13.00`, `$13`, and `13 dollars` all register as $13.00 USD.
cycle | (optional) Allows you to specify a recurring cycle. Values available: `once` (default), `weekly`, `monthly`, `quarterly`, or `yearly`.
nextdate | (optional) Used to schedule the next date a transaction will occur. Must be formatted: `mm/dd/yyyy`
startdate | (optional) Used to schedule first occurence of transaction if none have occured. Must be formatted: `mm/dd/yyyy`
token | (optional) Used to update the payment token for a transaction.  Payment tokens are returned when a recurring transaction is created or future payment scheduled.
card[] | (optional) Used to update card information for a transaction.
check[] | (optional) Used to update check information for a transaction.

#### card[] - Credit Card
* `card[number]` Credit card number. Must contain 15-16 digits.
* `card[expmonth]` Credit card expiration month. Must contain 2 digits between 1 and 12.
* `card[expyear]` Credit card expiration year. Must contain 2 digits, later than the current year.
* `card[cvv]` Card security code (CVV/CVC). Must contain 3-4 digits.

#### check[] - EFT / E-check
* `check[aba]` 9-digit bank routing number.
* `check[account]` Bank account number.
* `check[type]` Bank account type. Can be one of: `savings`, `checking`, `bsave` (business savings) or `bcheck` (business checking).


## Payment Information Vault

    POST https://api.cornerstone.cc/v1/transactions

Store payment information securely in a vault without making a charge or authorization to the payment method.

To use the vault, add the “vault” field to a normal transaction request. When “vault” is present, the amount is no longer required, and if included, will be ignored. As long as the payment information is valid, you will receive an “approved” or “accepted” status.

### Examples 

Valid Vaulted Payment:

    POST https://api.cornerstone.cc/v1/transactions

```yaml
vault: 1
customer[email]: angusm@example.com
customer[firstname]: Angus
customer[lastname]: MacGyver
card[number]: 370000000000002
card[expmonth]: 12
card[expyear]: 23
card[cvv]: 1114
```

```json
HTTP/1.1 200 OK

{
	"approved": [
		{
			"id": 99999,
			"merchant": "Thorton Towing",
			"reason": "Accepted",
			"amount": 0,
			"frequency": "once",
			"startdate": false
		}
	]
}
```


## Fetch-Transactions

    GET https://api.cornerstone.cc/v1/transactions

Fetch a list of transactions, according to a given filter.

### Parameters

Name | Usage
----:| -----
range          | Filters by date or date range. Format: `12/31/1999` or `01/31/1999-12/31/1999` (optional). If left empty, this turns into today's date. To get all transactions, enter `*`. Note that this is slow and it's recommended not to use in a production environment, and instead a range should be used.
amount         | Dollar amount. We try to determine what you mean automatically, so `13`, `13.00`, `$13`, and `13 dollars` all register as $13.00 USD. (optional)
firstname      | Filter by customer's first name (optional)
lastname       | Or customer's last name (optional)
email 	       | Or customer's email (optional)
customerid     | Or customer ID (optional) 
payment_type[] | Can be a combination of `amex`, `discover`, `visa`, `mastercard`, and `check` (optional)
merchant       | Filter by sub-merchant name (optional)
failed         | Display declined transactions instead of approved transactions (optional)
scheduled      | Display scheduled transactions instead of approved or declined (optional)
show_test      | Display test transactions (optional)
trans_id       | Fetch a singe transaction by ID. (optional)
custom[]       | Filter by any custom fields that may be present on the transaction (optional)
include_gid    | Include the transaction ID in the backend gateway with the response (optional)


# Merchant Applications Status

    GET https://api.cornerstone.cc/v1/applications/<application_id>

These are applications that are fed into Cornerstone's internal Sales program for approval as merchants. For Cornerstone partners, this allows them to check on applications that have been submitted through them.

## Examples

Fetching an approved application:

    GET https://api.cornerstone.cc/v1/applications/rVNw9CUcerzJ6rfwlIE0

```json
HTTP/1.1 200 OK

{
	"id": "rVNw9CUcerzJ6rfwlIE0",
	"partner": "Realm of Creativity",
	"dba": "Joe's Trucking",
	"approved": true,
	"merchant_id": "12345",
	"gateway_id": "9001",
	"gateway_key": "xyz123"
}
```

An application not approved:

    GET https://api.cornerstone.cc/v1/applications/LQr7oUAYJZLQsh27i1kP

```json
HTTP/1.1 200 OK

{
	"id": "GV5ScI689yJ07ezSi3kd4HIPi",
	"partner": "Realm of Creativity",
	"dba": "Laurence Chinese Palace",
	"approved": false
}
```

A pending application:

    GET https://api.cornerstone.cc/v1/applications/GV5ScI689yJ07ezSi3kd4HIPi

```json
HTTP/1.1 200 OK

{
	"id": "LQr7oUAYJZLQsh27i1kP",
	"partner": "Realm of Creativity",
	"dba": "James Smith Pizzeria",
	"pending": true
}
```

Finally, if an application by the ID does not exist, or if your client id doesn't have permission to view an application, you will receive a not found error:

    GET https://api.cornerstone.cc/v1/applications/xyz

```json
HTTP/1.1 404 Not Found

{
	"error": "not_found",
	"reason": "No application found by ID: xyz"
}
```
# Tenants

## Create a Tenant

    POST https://api.cornerstone.cc/v1/tenants
    
### Parameters

Name | Description
---- | -----------
merchant | (required) Name of the page or site the client has access to.
login | (required) Email to be associated the user account.
password | (required) Password associated with the account.
firstname | (optional) First name of the user.
lastname | (optional) Last name of the user.

```json
HTTP/1.1 200 OK

{
        "success": true,
        "tenant_id": 757928
}
```
## Fetch a Tenant

    GET https://api.cornerstone.cc/v1/tenants
    
### Parameters

Name | Description
---- | -----------
id | (required) Customer ID used to fetch customer account.

```json
HTTP/1.1 200 OK

{
        "success": true,
        "tenant": {
                "id": "757928",
                "merchant": "oneitem",
                "firstname": "bruce",
                "lastname": "wayne",
                "login": "bwayne@yahoo.com",
                "password": "stuff"
        }
}
```
## Update a Tenant

    PATCH https://api.cornerstone.cc/v1/tenants
    
### Parameters

Name | Description
---- | -----------
id | (required) Customer ID used to update.
logins | (optional) Update user login email.
firstname | (optional) Update user first name.
lastname | (optional) Update user last name.
```json
HTTP/1.1 200 OK

{
        "success": true,
        "tenant": {
                "id": "757928",
                "merchant": "oneitem",
                "firstname": "spider",
                "lastname": "man",
                "login": "spiderman@aol.com",
                "password": "stuff"
        },
        "message": "ID 757928 updated"
}
```
## Delete a Tenant

    DELETE https://api.cornerstone.cc/v1/tenants/<customerid>

```
HTTP/1.1 200 OK

{
        "success": true,
        "tenant_id": 757928,
        "message": "ID 757928 removed."
}
```
