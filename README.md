# Cycle Online REST API instruction
![Cyclebit](/images/logo/logo-color.svg)

## 1. Necessary information for integration
To start integration, you need to get your **agent’s account email**, account **secret key**, and **server secret key**.\
All this data will be provided by your manager when you are connected to the system.

**Example values:**\
*Agent’s account email:* `agent@cyclebit.io`\
*Secret key:* `1234432690`\
*Server secret key:* `AA1353253C3B8923BFD6AAAF3A73`

You will also need the JWT method for generating Bearer authorization token.
Example debugger and the list of supported languages: https://jwt.io/

## 2. Payment link creation
- **Bearer authorization token generation**.\
The token is generated using the JWT method with desired library for your language (HS256 algorithm).

    Example (*JSON Payload to be encoded*):
    ```json
    {
        "email": "agent@cyclebit.io",
	    "secret": "1234432690"
    }
    ```
    For key `email` value is the agent’s account email as sting.\
    For key `secret` value is the secret key as sting.
    - Algorithm: `HS256`
    - Server secret key for encoding payload **(server secret key)**: `AA1353253C3B8923BFD6AAAF3A73`
    - Encoded Bearer token: `eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6ImFnZW50QGN5Y2xlYml0LmlvIiwic2VjcmV0IjoiMTIzNDQzMjY5MCJ9.Y91CiJwYR8bgNQbWWZV_VqZAweEt09fQmmsLrOa0vKY`

- **Request for transaction creation and payment link generation.**\
POST request — https://online.cyclebit.net/api/payment/createPayment
\
    **Headers:**
    ```json
    {
        "content-type": "application/json",
        "merchant": "agent@cyclebit.io",
        "authorization": "Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6ImFnZW50QGN5Y2xlYml0LmlvIiwic2VjcmV0IjoiMTIzNDQzMjY5MCJ9.Y91CiJwYR8bgNQbWWZV_VqZAweEt09fQmmsLrOa0vKY"
    }
    ```
    **Body:**\
    You can send various parameters in the request body, such as the amount, the payer's\
    email address, the payment description that will be displayed in your list of operations, and so on.
    1. Necessary parameter:\
    `"amount": "100.11"` - amount for the payment in your local fiat currency (this value can be a\
    floating-point number with up to 2 digits after point **(string or int/float)**.
    1. Additional parameters:\
    `"redirect_url": "https://cyclebit.io/"` – URL that the user will be redirected to after\
    completing the payment successfully (If this parameter is not provided, the user will be sent\
    to the page with the payment receipt **(string)**;\
    `"description": "Just test"` – custom payment description that will be linked to the payment\
    and displayed in the list of operations **(string)**.\
    ![](/images/screenshots/description.png)
    1. Payer KYC parameters (optional, but you can submit them if you have the information\
    about the payer):\
    `"fullname": "John Johnson"` – payer’s name **(string)**;\
    `"dob": "2000-10-02"` – payer’s date of birth in the YYYY-MM-DD format **(string)**;\
    `"fullname": "John Johnson"` – payer’s name **(string)**;\
    `"email": "payer@gmail.com"` – payer’s email address **(string)**;\
    `"address": "Test Address 11"` – payer’s residential address **(string)**;\
    `"city": "Miami"` – payer’s city of residence **(string)**;\
    `"state": "Florida"` – payer’s state/province of residence **(string)**;\
    `"zip": "33140"` – payer’s zip/postal code **(string)**;

- **Response.**\
The response will come in this form:
    ```json
    {
        "url": "http://online.cyclebit.net/edit?id=0252f1b1-9fc8-4617-a70e-b4d2aca3c971",
        "tranId": "0252f1b1-9fc8-4617-a70e-b4d2aca3c971",
        "success": true
    }
    ```
    Field list:
    - `"url": "http://online.cyclebit.net/edit?id=0252f1b1-9fc8-4617-a70e-b4d2aca3c971"` – URL for redirecting\
    the user to further payment **(string)**;
    - `"tranId": "0252f1b1-9fc8-4617-a70e-b4d2aca3c971"` — transaction ID to track the payment by **(string)**;
    - `"success": true` – the status of a request **(boolean)**;

    If the request is made incorrectly, an error with a description will be returned:
    ```json
    {
        "url": null,
        "tranId": null,
        "success": false,
        "description": "IDX10503: Signature validation failed."
    }
    ```
    The `description` field will display the reason for the failure.

## 3. Payment monitoring
The payment status is checked via interaction with the Cyclebit processing API. The method of checking the transaction status
by `tranId` does not require authorization and is very easy to use.

Endpoint for the request (brackets should **not** be sent and indicates a variable):\
GET request — https://prc-api-pro-eu.cyclebit.net/api/v1/payment/statusPublic/{tranId}

Response example (if the payer has not yet selected a cryptocurrency for the payment):
```json
{
    "Transaction": null,
    "ErrorCode": 0,
    "ErrorMessage": null,
    "Validations": null
}
```
Response example (if the payer has chosen a cryptocurrency to pay with):
```json
{
    "Transaction": {
        "ID": "0252F1B1-9FC8-4617-A70E-B4D2ACA3C971",
        "State": 200,
        "Substate": 201,
        "ResultCode": 0,
        "ResultMessage": null,
        "Status": null
    },
    "ErrorCode": 0,
    "ErrorMessage": null,
    "Validations": null
}
```
Monitoring must be performed by the value of the `Substate` attribute in the `Transaction` object.

If the response is received with `"Transaction": null` for more than an hour, you must generate a
new payment link for the payer and monitor a new transaction by `tranId`.

Dictionary of transaction substatuses (`Substate` value in `Transaction` object):
1. `201` — the operation is waiting for the payer to send the funds;
1. `203` — the sent funds have been detected. Waiting for the confirmation(s) in the blockchain;
1. `401` or `411` — the payment was made **successfully**, operation is finalized;
1. `402` or `412` — the payment is **failed**, operation is finalized.

If Cyclebit does not detect funds within 15 minutes after receiving the `201` status, the operation will fail and go to the `402` or `412` status.

If a Substatus `203` was received, Cyclebit detected the sent funds in the blockchain. This status is not secure, and Cyclebit is not yet responsible for these funds.

The operation should be considered successful **only** if the substatus `401` or `411` was received.
The receipt was sent to the payer’s email address specified in the KYC form. You can release good(s)/service(s) to your costumer.

If the status `402` or `412` was received, **you must stop** monitoring the transaction.
If possible, generate a new payment link for the payer.