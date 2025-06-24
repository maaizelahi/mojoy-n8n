# Guide: Error Notification Workflow

**Filename (New Workflow):** `workflows/Error_Notification_Workflow.json` (You will create this file with the JSON provided by the AI)

## 1. Objective

This workflow serves as a centralized error handler for other n8n workflows, specifically the "WooCommerce Order to Shipping Slip Generator Workflow." When an unhandled error occurs in the main workflow, this error workflow is triggered to format a detailed error message and send a notification.

## 2. Workflow Structure

1.  **`Error Trigger` (Error Trigger Node):**
    *   **Trigger:** This node automatically starts the workflow when an error occurs in another workflow that has been configured to use this one as its "Error Workflow."
    *   **Output:** Provides detailed information about the error, including the error message, stack trace, details of the failed node, and data from the failed execution.

2.  **`Format Error Message` (Function Node):**
    *   **Input:** JSON data from the `Error Trigger` node.
    *   **Action:**
        *   Extracts key information from the error data:
            *   Original workflow name and ID.
            *   Execution ID.
            *   Name and type of the node that failed.
            *   Error message and stack trace.
        *   Attempts to find a WooCommerce Order ID from the input data of the failed node, if available, to provide more context.
        *   Constructs a formatted subject line and body for an email notification.
    *   **Output:** JSON object containing `emailSubject`, `emailBody`, and the original `errorDetails`.

3.  **`Send Email Notification (NEEDS CONFIGURATION)` (Send Email Node - Example):**
    *   **Input:** `emailSubject` and `emailBody` from the `Format Error Message` node.
    *   **Action:** Sends an email with the error details.
    *   **Configuration (Crucial - User Must Do):**
        *   **SMTP Server Details:** `host`, `port`, `user`, `password` (or use n8n's SMTP credentials).
        *   **Sender/Recipient:** `sender` email address, `to` email address(es).
        *   **Security:** `secure` option (TLS/SSL).
    *   **Note:** This node can be replaced with any other notification node (Slack, Pushover, etc.) based on user preference. The new node would take `emailSubject` (or a similar title field) and `emailBody` as input.

## 3. Setup Instructions

1.  **Create the Workflow File:**
    *   The AI has provided the full JSON for this workflow (`workflows/Error_Notification_Workflow.json`). Ensure this file exists with the correct content.

2.  **Import and Configure in n8n:**
    *   Open your n8n instance. It should automatically detect the new workflow. If not, import it manually via the UI ("Workflows" -> "Import from File").
    *   Open the "Error Notification Workflow."
    *   **Select and configure the notification node** (e.g., "Send Email Notification").
        *   Enter your SMTP server details, port, username, and password. **It is highly recommended to store SMTP credentials in n8n's Credentials section and select them here.**
        *   Set the "Sender Email" and "To Email(s)."
        *   Adjust security options (SSL/TLS) as required by your SMTP provider.
        *   If using a different notification service (Slack, etc.), replace the Email node and configure it.

3.  **Activate the Workflow:**
    *   Once configured, toggle the workflow to "Active."

4.  **Link from Main Workflow:**
    *   Open your main "WooCommerce Order to Shipping Slip Generator Workflow."
    *   Click the **Settings icon** (gear) at the top of the workflow editor.
    *   In the "Error Workflow" dropdown menu, select the "Error Notification Workflow" you just set up and activated.
    *   Save the main workflow.

## 5. How it Works

*   When an unhandled error occurs in the main "WooCommerce Order to Shipping Slip Generator Workflow" (or any other workflow configured to use this as its error handler):
    *   The main workflow's execution for that item stops.
    *   The "Error Trigger" node in *this* workflow activates, receiving details about the failure.
    *   The "Format Error Message" node processes these details into a human-readable message.
    *   The configured notification node (e.g., "Send Email") sends out the alert.

This setup ensures you are promptly informed of any issues in your automated processes, allowing for quicker diagnosis and resolution.
