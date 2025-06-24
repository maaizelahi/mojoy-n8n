# Delivery Confirmation Workflow Documentation

## 1. Purpose

This workflow handles the delivery confirmation process. It is triggered when a delivery person submits OTP confirmation details, typically via a Google Form, a custom app, or a Google Apps Script connected to a spreadsheet that then calls this workflow's webhook. The workflow validates the submitted OTP against the one stored for the order, updates Google Sheets for tracking and logging, and updates the order status in WooCommerce.

## 2. Workflow Trigger

The workflow is triggered by an HTTP POST request to the N8N webhook URL: `/delivery-confirmation-sheet`. This webhook is intended to be called by the system used by the delivery personnel for submitting OTP and delivery status (e.g., a Google Form submission handler).

## 3. Expected Webhook Input

The webhook expects a JSON payload. The essential fields in the `body` of the JSON (or directly in the JSON if not nested) are:

```json
{
  "orderId": "12345", // or "Order ID"
  "deliveryPersonName": "John Rider", // or "Delivery Person Name"
  "otp": "654321", // or "OTP"
  "deliveryStatus": "Delivered", // Optional, defaults to "Delivered"
  "deliveryNotes": "Customer received package.", // Optional
  "rowId": "101", // Optional, if coming from a specific row update in a sheet
  "spreadsheetId": "sheet_id_if_any" // Optional, for context
}
```
Field names like "Order ID" or "OTP" (with spaces or different casing) are also handled by the initial extraction logic.

## 4. Node Breakdown

The workflow consists of the following nodes:

1.  **Webhook: Delivery Confirmation (`Webhook: Delivery Confirmation`)**
    *   **Type:** `n8n-nodes-base.webhook`
    *   **Path:** `delivery-confirmation-sheet`
    *   **Method:** `POST`
    *   **Function:** Receives the delivery confirmation data. Expects raw body for custom parsing.

2.  **Extract & Validate Form Data (`Extract & Validate Form Data`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:**
        *   Parses the incoming JSON payload (handles if data is in `input.body` or at the root).
        *   Extracts `orderId`, `deliveryPersonName`, and `otp`, trimming whitespace. It also looks for alternative field names like `Order ID`.
        *   Validates that these three fields are present and non-empty.
        *   If validation fails, it prepares an error message with `success: false` and `errorType: "INVALID_INPUT"`.
        *   Extracts optional fields: `deliveryStatus` (defaults to "Delivered"), `deliveryNotes`, `rowId`, `spreadsheetId`.
        *   Returns a JSON object with the extracted data and validation status (`validInputs: true/false`).

3.  **Valid Inputs? (`Valid Inputs?`)**
    *   **Type:** `n8n-nodes-base.if`
    *   **Function:** Routes based on the `validInputs` flag from the previous node.
        *   **True Path:** Proceeds to `GSheets: Look Up Order`.
        *   **False Path:** Proceeds to `Invalid Input Response` to generate an error message for the webhook response.

4.  **GSheets: Look Up Order (`GSheets: Look Up Order`)**
    *   **Type:** `n8n-nodes-base.googleSheets` (Version 4)
    *   **Function:** Looks up the order details (especially the stored OTP) in the main OTP tracking Google Sheet using the provided `orderId`.
    *   **Configuration:**
        *   Uses OAuth2.
        *   Sheet ID: `{{ $env.OTP_TRACKING_SHEET_ID || '1UXjKutLclMItkfe7yxt16conqSH54NNDSXKxbHRgMr4' }}`
        *   Sheet Name: `{{ $env.OTP_TRACKING_SHEET_NAME || 'Sheet1' }}`
        *   Operation: `lookup`, `lookupColumn: "orderId"`.
    *   **Error Handling:** If the lookup fails, proceeds to `Handle GSheets Lookup Error`.

5.  **Handle GSheets Lookup Error (`Handle GSheets Lookup Error`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:** Executed if `GSheets: Look Up Order` fails. Logs the error and prepares a generic error response (`errorType: 'GSHEETS_LOOKUP_FAILED'`).

6.  **Process Lookup & Verify OTP (`Process Lookup & Verify OTP`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:**
        *   Processes the result from `GSheets: Look Up Order`.
        *   If the order is not found (lookup returns empty or no data), it sets `orderExists: false` and prepares an "Order Not Found" message.
        *   If found, it extracts the `storedOtp`.
        *   Compares the `storedOtp` with the `providedOtp` from the input.
        *   Sets `otpMatches: true/false`.
        *   Merges input data with sheet data and returns a comprehensive JSON object including `orderExists` and `otpMatches` flags.

7.  **Order Exists? (`Order Exists?`)**
    *   **Type:** `n8n-nodes-base.if`
    *   **Function:** Routes based on the `orderExists` flag from `Process Lookup & Verify OTP`.
        *   **True Path:** Proceeds to `OTP Matches?`.
        *   **False Path:** Proceeds to `Order Not Found Response`.

8.  **OTP Matches? (`OTP Matches?`)**
    *   **Type:** `n8n-nodes-base.if`
    *   **Function:** Routes based on the `otpMatches` flag.
        *   **True Path (OTP Matches):** Proceeds to `GSheets: Update Main Tracking (Success)`.
        *   **False Path (OTP Mismatch):** Proceeds to `OTP Mismatch Response`.

9.  **GSheets: Update Main Tracking (Success) (`GSheets: Update Main Tracking (Success)`)**
    *   **Type:** `n8n-nodes-base.googleSheets` (Version 4)
    *   **Function:** If OTP matches, this node updates the status and other delivery details in the main OTP tracking Google Sheet.
    *   **Configuration:** Same Sheet ID/Name as `GSheets: Look Up Order`. Updates columns like `status`, `deliveryPersonName`, `deliveryNotes`, `otpVerifiedTimestamp`, `otpVerificationStatus`.
    *   **Error Handling:** If update fails, proceeds to `Handle Main Tracking Update Error (Success Path)`.

10. **Handle Main Tracking Update Error (Success Path) (`Handle Main Tracking Update Error (Success Path)`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:** Logs the error. Critical operations (logging to form response, updating WooCommerce) still proceed. Passes through the data from `Process Lookup & Verify OTP`.

11. **GSheets: Update Form Response Log (`GSheets: Update Form Response Log`)**
    *   **Type:** `n8n-nodes-base.googleSheets` (Version 4)
    *   **Function:** Logs the outcome of the OTP verification attempt (Verified/Mismatch) along with other delivery details to a separate "Form Responses" or audit log Google Sheet. This runs after the main tracking sheet update attempt (on success path of OTP match).
    *   **Configuration:**
        *   Sheet ID: `{{ $env.DELIVERY_FORM_RESPONSES_SHEET_ID || '1Wqoo6mylPVXO1yV0vgb3hVAdEfq6l9uLj_nZ0Q-1FNo' }}`
        *   Sheet Name: `{{ $env.DELIVERY_FORM_RESPONSES_SHEET_NAME || 'Form Responses 1' }}`
        *   Updates columns like `Status Message`, `Delivery Confirmation Timestamp`, `Delivery Notes`, `Delivery Person`.
    *   **Error Handling:** If update fails, proceeds to `Handle GSheets Log Update Error`.

12. **Handle GSheets Log Update Error (`Handle GSheets Log Update Error`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:** Logs the error. Prepares a message indicating delivery was confirmed but logging failed. Tries to proceed with WooCommerce update.

13. **WooCommerce: Update Order Status (`WooCommerce: Update Order Status`)**
    *   **Type:** `n8n-nodes-base.wooCommerce` (Version 1.2 example)
    *   **Function:** If OTP verification was successful (and previous GSheet updates attempted), this node updates the order status in WooCommerce to "completed" and adds a note about the delivery confirmation.
    *   **Configuration:** Uses WooCommerce API credentials.
    *   **Error Handling:** If update fails, proceeds to `Handle WooCommerce Update Error`.

14. **Handle WooCommerce Update Error (`Handle WooCommerce Update Error`)**
    *   **Type:** `n8n-nodes-base.code`
    *   **Function:** Logs the error. Prepares a message indicating delivery was confirmed but WooCommerce update failed.

15. **Response Nodes (Code type):**
    *   **Success Response:** Prepares a JSON success message for the webhook if all primary operations (OTP match, WooCommerce update) are successful.
    *   **OTP Mismatch Response:** Prepares a JSON error message for `errorType: "OTP_MISMATCH"`.
    *   **Order Not Found Response:** Prepares a JSON error message for `errorType: "ORDER_NOT_FOUND"`. (Message generated in `Process Lookup & Verify OTP`).
    *   **Invalid Input Response:** Prepares a JSON error message for `errorType: "INVALID_INPUT"`. (Message generated in `Extract & Validate Form Data`).

16. **Respond to Webhook (`Respond to Webhook`)**
    *   **Type:** `n8n-nodes-base.respondToWebhook`
    *   **Function:** Sends the final JSON response (prepared by one of the preceding success/error handler nodes) back to the webhook caller.

## 5. Key Configurations & Credentials

*   **Google Sheets Credentials:**
    *   An OAuth2 credential for Google Sheets.
    *   `$env.OTP_TRACKING_SHEET_ID`: ID of the main sheet where OTPs are stored and looked up.
    *   `$env.OTP_TRACKING_SHEET_NAME`: Name of the sheet (e.g., "Sheet1") within the OTP tracking spreadsheet.
    *   `$env.DELIVERY_FORM_RESPONSES_SHEET_ID`: ID of the sheet used for logging form responses/delivery attempts.
    *   `$env.DELIVERY_FORM_RESPONSES_SHEET_NAME`: Name of the sheet (e.g., "Form Responses 1") within the form responses spreadsheet.
*   **WooCommerce Credentials:**
    *   WooCommerce API credentials stored in N8N.
*   **Webhook Configuration:**
    *   The system used for delivery confirmation (e.g., Google Apps Script handling a Form submission) must be configured to send a POST request to `[N8N_INSTANCE_URL]/webhook/delivery-confirmation-sheet` with the payload.

## 6. Outputs

*   **Primary System Updates (on successful OTP match):**
    *   Order status updated to "completed" in WooCommerce with a confirmation note.
    *   Main OTP tracking Google Sheet updated with delivery details and "Verified" status.
    *   Delivery attempt/confirmation logged in the "Form Responses" Google Sheet.
*   **Webhook Response:**
    *   A JSON response indicating success or failure, with appropriate messages and error types:
        *   `success: true` with delivery details on full success.
        *   `success: false` with messages for `INVALID_INPUT`, `GSHEETS_LOOKUP_FAILED`, `ORDER_NOT_FOUND`, `OTP_MISMATCH`, `GSHEETS_LOG_UPDATE_FAILED`, `WOOCOMMERCE_UPDATE_FAILED`.

This workflow provides a comprehensive way to manage delivery confirmations, ensuring data consistency across different platforms.
