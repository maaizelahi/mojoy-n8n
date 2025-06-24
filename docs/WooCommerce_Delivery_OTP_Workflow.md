# WooCommerce Delivery OTP Workflow Documentation

## 1. Purpose

This workflow automates the generation and sending of a One-Time Password (OTP) to a customer when their WooCommerce order is marked as "Out for Delivery" (or a similar status triggering the webhook). It also logs the OTP details and SMS sending status to a Google Sheet for tracking.

## 2. Workflow Trigger

The workflow is triggered by an HTTP POST request to the N8N webhook URL: `/woo-out-for-delivery`. This webhook should be configured in WooCommerce (e.g., via an order status update hook) to send order data when an order is ready for delivery.

## 3. Expected Webhook Input

The webhook expects a JSON payload, typically originating from WooCommerce. The essential fields required from the `body` of the JSON payload are:

```json
{
  "order_id": "12345",
  "billing": {
    "first_name": "John",
    "last_name": "Doe",
    "phone": "9876543210",
    "email": "john.doe@example.com"
  },
  "shipping": {
    "address_1": "123 Main St",
    "city": "Anytown",
    "state": "CA",
    "postcode": "90210"
  },
  "status": "processing" // Or any status that indicates out for delivery
}
```

## 4. Node Breakdown

The workflow consists of the following nodes:

1.  **Webhook (`Webhook`)**
    *   **Type:** `n8n-nodes-base.webhook`
    *   **Path:** `woo-out-for-delivery`
    *   **Method:** `POST`
    *   **Function:** Receives the initial order data from WooCommerce. Expects raw body to allow for custom parsing.

2.  **Validate Webhook Data (`Validate Webhook Data`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:**
        *   Parses the incoming JSON body.
        *   Validates the presence of critical fields: `order_id`, `billing` object, `billing.phone`, `billing.email`.
        *   If validation fails, it prepares an error message and a 400 status code to be sent back by the `Respond Webhook` node.

3.  **If Valid Data (`If Valid Data`)**
    *   **Type:** `n8n-nodes-base.if`
    *   **Function:** Routes the workflow based on the output of the `Validate Webhook Data` node.
        *   **True Path:** If data is valid (no error detected), proceeds to `Generate OTP & Prepare Data`.
        *   **False Path:** If data is invalid, proceeds to `Respond Webhook` to immediately send an error response.

4.  **Generate OTP & Prepare Data (`Generate OTP & Prepare Data`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:**
        *   Generates a 6-digit random OTP.
        *   Extracts and formats customer billing and shipping information.
        *   Cleans the phone number (removes non-digits, handles Indian country code prefixes).
        *   Constructs a customer name, with a fallback if first/last names are missing.
        *   Prepares a JSON object containing `orderId`, `customerName`, `phone`, `email`, `otp`, `status`, `timestamp`, and `address` for subsequent nodes.

5.  **Send SMS via MSG91 (`Send SMS via MSG91`)**
    *   **Type:** `n8n-nodes-base.httpRequest`
    *   **Function:** Sends the generated OTP to the customer's phone number using the MSG91 API.
    *   **Configuration:**
        *   Uses an API Key for authentication (managed via n8n credentials: `MSG91Api.apiKey`).
        *   The MSG91 API URL and Flow ID are configurable via secrets: `$secrets.MSG91_API_URL` and `$secrets.MSG91_FLOW_ID`.
        *   The JSON body includes the `flow_id`, `sender` ID, customer's `mobiles` number (prefixed with 91), and variables for customer name, OTP, and order ID (`VAR1`, `VAR2`, `VAR3`).
    *   **Error Handling:** If the API call fails or returns an error, the workflow proceeds to the `Handle MSG91 Error` node.

6.  **Handle MSG91 Error (`Handle MSG91 Error`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:**
        *   Executed if the `Send SMS via MSG91` node encounters an error.
        *   Logs the error details.
        *   Prepares the data for Google Sheets, including the original prepared data and an `smsSentStatus` field indicating failure with the error message. This ensures that even failed SMS attempts are logged.

7.  **Upsert to Google Sheets (`Upsert to Google Sheets`)**
    *   **Type:** `n8n-nodes-base.googleSheets`
    *   **Function:** Logs the order and OTP details to a Google Sheet. It performs an "upsert" operation based on `orderId`.
    *   **Configuration:**
        *   Uses OAuth2 for authentication (Google Sheets account credential).
        *   The Google Sheet ID is configurable via an environment variable: `$env.GOOGLE_SHEET_ID` (fallback to `1UXjKutLclMItkfe7yxt16conqSH54NNDSXKxbHRgMr4`).
        *   The target range is `A:I`.
        *   The `key` for upserting is `orderId`.
        *   Maps fields: `orderId`, `customerName`, `phone`, `email`, `otp`, `status`, `timestamp`, `address`, and `smsSentStatus` (which indicates success/failure of the SMS from the previous HTTP request or error handler).
    *   **Error Handling:** If the Google Sheets operation fails, the workflow proceeds to the `Handle Google Sheets Error` node.

8.  **Handle Google Sheets Error (`Handle Google Sheets Error`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:**
        *   Executed if the `Upsert to Google Sheets` node fails.
        *   Logs the error details.
        *   Prepares an error JSON (further actions like notifications could be added here).
        *   *Currently, this is a terminal error logging node for this path.*

9.  **Respond Webhook (`Respond Webhook`)**
    *   **Type:** `n8n-nodes-base.respondToWebhook`
    *   **Function:** Sends a response back to the initial webhook caller.
        *   If triggered from the `If Valid Data` (FALSE path), it sends the error message and status code (e.g., 400) prepared by the `Validate Webhook Data` node.
        *   *Note: For successful initiations, this workflow primarily focuses on out-of-band OTP delivery. A success response to the webhook after all operations could be added if required, but typically a quick acknowledgment (like a 200 OK, often handled by n8n implicitly if no `Respond Webhook` node is hit early) is sufficient for such triggers.*

## 5. Key Configurations & Credentials

*   **MSG91 Credentials:**
    *   An API key for MSG91 must be stored in N8N's credentials manager (e.g., named `MSG91Api` with a field `apiKey`).
    *   `$secrets.MSG91_API_URL`: The API endpoint for MSG91 (e.g., `https://api.msg91.com/api/v5/flow/`).
    *   `$secrets.MSG91_FLOW_ID`: Your specific MSG91 Flow ID designed for sending OTPs.
    *   `$secrets.MSG91_SENDER_ID`: Your MSG91 approved Sender ID.
*   **Google Sheets Credentials:**
    *   An OAuth2 credential for Google Sheets must be configured in N8N.
    *   `$env.GOOGLE_SHEET_ID`: The ID of the Google Sheet used for logging OTPs.
*   **Webhook Configuration:**
    *   WooCommerce (or the triggering system) must be configured to send a POST request to `[N8N_INSTANCE_URL]/webhook/woo-out-for-delivery` with the order data.

## 6. Outputs

*   **Primary Output:** An SMS containing the OTP is sent to the customer's phone number.
*   **Secondary Output (Logging):**
    *   Order details, OTP, and SMS status (`Success` or `Failed: <error>`) are logged to the configured Google Sheet.
*   **Webhook Response:**
    *   If input validation fails: An HTTP 400 response with error details.
    *   If input validation succeeds: Typically an HTTP 200 OK (often implicit from N8N if no `Respond Webhook` node is hit on the success path early, or a 202 Accepted if a `Respond Webhook` node is added after initiating background tasks).

This workflow ensures that customers receive OTPs promptly for delivery verification and that all attempts are tracked for operational purposes.
