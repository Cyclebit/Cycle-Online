# Cycle Online REST API instruction
![Cyclebit](/images/logo/logo-color.svg)

## 1. Necessary information for integration
To start integration, you need to get your **agent’s account email**, account **secret key**, and **server secret key**.\
All this data will be provided by your manager when you connect to the system.

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
    For key `email` value is the agent’s account email as string.\
    For key `secret` value is the secret key as string.
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
    `"amount": "100.11"` - amount for the payment in your local fiat currency\
    (this value can be a floating-point number with up to 2 digits after point) **(string or int/float)**;
    1. Additional parameters:\
    `"redirect_url": "https://cyclebit.io/"` – URL that the user will be redirected to after\
    completing the payment successfully (If this parameter is not provided, the user will be sent\
    to the page with the payment receipt) **(string)**;\
    `"description": "Just test"` – custom payment description that will be associated with the payment\
    and displayed in the list of operations in the Cyclebit web UI **(string)**;\
    ![](/images/screenshots/description.png) \
    \
    `"callback_url": "https://callbackserver.com"` – URL to which the callback notification message\
    will be sent after the payment is successfully confirmed **(string)**.\
    Callbacks will be described [below](#callbacks).
    1. Payer KYC parameters (optional, but you can submit them if you have the information\
    about the payer):\
    `"fullname": "John Johnson"` – payer’s name **(string)**;\
    `"dob": "2000-10-02"` – payer’s date of birth in the YYYY-MM-DD format **(string)**;\
    `"fullname": "John Johnson"` – payer’s name **(string)**;\
    `"email": "payer@gmail.com"` – payer’s email address **(string)**;\
    `"address": "Test Address 11"` – payer’s residential address **(string)**;\
    `"city": "Miami"` – payer’s city of residence **(string)**;\
    `"state": "Florida"` – payer’s state/province of residence **(string)**;\
    `"zip": "33140"` – payer’s zip/postal code **(string)**.

- **Response**\
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
    - `"success": true` – status of the request **(boolean)**.

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

### Callbacks

To simplify the monitoring of Cycle Online transactions, it makes sense to use callbacks.\
This will allow you to not create a separate queue system for checking payments statuses, but simply receive notifications about successful payments.\
To do this, you need to deploy a simple server on your side, which will listen for incoming POST requests from the Cyclebit server.

Callbacks workflow:
1. Create a payment with the `callback_url` parameter, which will contain the address of your notification server.\
Example: `"callback_url": "https://myfavouriteshop.com/notifications"`\
**Important**: your website must support the HTTPS protocol (URL must start with https://);
1. Listen for HTTPS POST requests that come to this URL;
1. As soon as you receive a notification with a similar body, respond to it with any message with the HTTP code **200 OK**, otherwise the Cyclebit server will resend you a notification for a specific payment again.
Notification request body model:
    ```json
    {
        "TransactionId": "e3cc51a0-4542-4433-af5e-28f2dea99e7e",
        "State": "completed",
        "Success": true
    }
    ```
1. Make a payment verification [status request to the public Cycle Online API](#status-request) to make sure that the transaction is completed successful.


In any case, do not rely on the response from the callback notification to directly confirm the payment in your system, as the address of the Cycle Online server may change.\
**Be sure to perform the fourth step** of workflow.

The Cyclebit server will send you notification callbacks for each transaction until it receives an HTTP 200 OK response from your server, or until 8 callbacks with responses other than 200 OK have passed.

Retry intervals for notifications:
1. Immediately after payment confirmation
1. 30 seconds after payment confirmation
1. 5 minutes after payment confirmation
1. 30 minutes after payment confirmation
1. 1 hour after payment confirmation
1. 4 hours after payment confirmation
1. 12 hours after payment confirmation
1. 24 hours after payment confirmation



### Status request

The payment status is checked via interaction with the Cyclebit public API.\
The method of checking the payment status by `tranId` does not require authorization and is very easy to use.

Endpoint for the request (braces should **not** be sent and indicates a variable):\
GET request — https://online.cyclebit.net/publicApi/payment/statusPublic/{tranId}

Response example (successfully completed payment):
```json
{
    "state": "completed",
    "success": "true",
    "description": null
}
```
Response example (no transaction was found with the provided `tranId`):
```json
{
    "state": null,
    "success": false,
    "description": "Transaction guid not found"
}
```
Monitoring must be performed by the value of the `state` attribute in the returned json object.

Dictionary of payment statuses (`state` value):
1. `null` — transaction id not found;
1. `created` — payment was created, the payer has not yet selected the currency for payment;
1. `pending` — payer chose the currency to pay the invoice, but incoming transaction in the blockchain has not yet been detected;
1. `paid` — incoming transaction in blockchain was found, but did not reach the required amount of confirmations;
1. `completed` — payment is successfully completed, you can release good(s)/service(s) to the payer, the receipt has been sent to the payer's email address specified in the KYC form;
1. `failed` — payment failed for some reason (verification error, funds were not sent by the payer, etc.).


If the `paid` status was received, Cyclebit detected the sent funds in the blockchain.\
This status is **not secure**, and Cyclebit is not yet responsible for these funds.\
\
The payment should be considered successful **only** if the status `completed` was received.