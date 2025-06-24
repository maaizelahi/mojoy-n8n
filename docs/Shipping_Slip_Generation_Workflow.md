# Workflow Documentation: WooCommerce Order to Shipping Slip Generation & Batch Processing

**Version:** As of last commit (`feature/shipping-slip-to-gdrive-full-error-handling`)

**Filename:** `workflows/WooCommerce_Order_to_Shipping_Slip_Generator_Workflow.json`

## 1. Objective

Automate the generation of individual A6 shipping slips from WooCommerce orders when they reach "processing" status. These slips are uploaded to Google Drive. Subsequently, a scheduled process compiles these individual slips into a daily batch PDF, also uploaded to Google Drive, suitable for printing (typically on a thermal printer). The workflow also aims to update WooCommerce orders with relevant metadata about the generated slips and includes provisions for error handling.

## 2. Workflow Structure

The workflow is divided into two main parts:

### Part 1: Individual Shipping Slip Generation & Upload (Webhook Triggered)

This part is triggered by an external event, expected to be a WooCommerce webhook.

1.  **`Webhook` (Webhook Node):**
    *   **Trigger:** Listens for POST requests.
    *   **Purpose:** Receives order status updates from WooCommerce.
    *   **Expected Data:** JSON payload containing at least `id` (WooCommerce Order ID) and `status`.
    *   **Output:** Passes the received JSON data.

2.  **`Check if Status is Processing` (IF Node):**
    *   **Input:** JSON data from `Webhook`.
    *   **Condition:** Checks if `$json.status` equals `"processing"`.
    *   **Output (True):** Passes data to `Get Order Details`.
    *   **Output (False):** Flow for this item stops (connected to `No Operation`).

3.  **`Get Order Details` (WooCommerce Node):**
    *   **Input:** JSON data (specifically `$json.id` for the Order ID).
    *   **Action:** Fetches complete order details from the configured WooCommerce store.
    *   **Credentials:** Requires WooCommerce API credentials (to be set by user).
    *   **Configuration:** The WooCommerce site URL needs to be correctly set in this node.
    *   **Output:** Detailed order object.

4.  **`Prepare Shipping Data` (Function Node):**
    *   **Input:** Full order object from `Get Order Details`.
    *   **Actions:**
        *   Formats dates, shipping address, and other order-specific information.
        *   Generates a prefixed slip Order ID (e.g., `PO` + original ID).
        *   Constructs image URLs for a QR code and a barcode (containing the slip Order ID) using `quickchart.io`.
        *   Assembles the complete HTML for an A6 shipping slip, embedding the dynamic data and image URLs. The HTML includes `@page` CSS rules for A6 size and zero margins.
    *   **Output:** JSON object containing `htmlContent`, `orderId` (the prefixed slip ID), and the original `orderData`.

5.  **`HTML to PDF Converter` (HTTP Request Node):**
    *   **Input:** `htmlContent` from `Prepare Shipping Data`.
    *   **Action:** Sends the HTML to a custom PDF conversion service (`https://tools.mojoy.in/html-to-pdf`).
    *   **Authentication:** Uses HTTP Basic Authentication (credentials to be set by user).
    *   **Output:** Binary PDF data (expects `application/pdf` response).

6.  **`Upload PDF to Google Drive` (Google Drive Node - ID `gdrive-upload-node`):**
    *   **Input:** Binary PDF data from `HTML to PDF Converter`; `orderId` (prefixed slip ID for filename) from data passed through.
    *   **Action:** Uploads the generated PDF to a specified Google Drive folder.
    *   **Credentials:** Requires Google Drive API credentials (to be set by user).
    *   **Configuration:** Destination folder path/ID on Google Drive needs to be set by user. Filename is `shipping-slip-[orderId].pdf`.
    *   **Output:** Google Drive file metadata (including `id` - GDrive file ID, and `name` - filename on GDrive).

7.  **`Add Google Drive ID to Batch File` (Function Node):**
    *   **Input:** Google Drive file metadata (`id`, `name`), `orderId` (prefixed slip ID), and `orderData` (original WooCommerce order details).
    *   **Actions:**
        *   Appends the Google Drive file ID (`$json.id`) to a daily batch control text file (e.g., `./shipping_slips/batch_gdrive_ids_YYYY-MM-DD.txt`) stored on the n8n server's local filesystem.
        *   Updates a daily count file (e.g., `./shipping_slips/count_YYYY-MM-DD.txt`) also on the local filesystem.
        *   Ensures the `./shipping_slips` directory exists.
    *   **Output:** Passes through `orderData`, `slipOrderId` (prefixed ID), `googleDriveFileName`, `processedGoogleDriveFileId` (GDrive file ID), and `batchFile` (path to the local batch control file).

8.  **`Update Order Metadata` (WooCommerce Node):**
    *   **Input:** `orderData` (for original WC Order ID), `googleDriveFileName`, `processedGoogleDriveFileId`, `batchFile`.
    *   **Action:** Updates the WooCommerce order with custom metadata fields:
        *   `shipping_slip_generated: true`
        *   `shipping_slip_filename`: (Filename on Google Drive)
        *   `shipping_slip_gdrive_id`: (Google Drive File ID)
        *   `shipping_slip_batch_file`: (Name of the batch control text file)
        *   `shipping_slip_generated_at`: (Timestamp)
    *   **Credentials:** Requires WooCommerce API credentials (to be set by user).

### Part 2: Batch PDF Compilation & Upload (Schedule Triggered)

This part runs on a schedule to process the batch of slips recorded by Part 1.

1.  **`Schedule Trigger for Batch Processing` (Schedule Node):**
    *   **Trigger:** Runs at defined intervals (e.g., hourly, daily at set times).
    *   **Output:** An empty item that starts this part of the flow.

2.  **`Read Google Drive IDs Batch File` (Function Node - ID `read-gdrive-batch-file`):**
    *   **Input:** Trigger item.
    *   **Action:**
        *   Reads the daily batch control file (e.g., `./shipping_slips/batch_gdrive_ids_YYYY-MM-DD.txt`) from the n8n server filesystem. This file contains one Google Drive file ID per line.
    *   **Output:** Outputs multiple items, each containing one `googleDriveFileId`, `today` (date string), and `batchFilePath` (path to the control file read). This allows subsequent nodes to loop through each ID.

3.  **Loop (Implicit - n8n's default behavior for multiple items):**
    The following two nodes operate for each item (i.e., each Google Drive file ID) output by the previous node.

    *   **`Download PDF from Google Drive` (Google Drive Node - ID `gdrive-download-pdf`):**
        *   **Input:** `googleDriveFileId`.
        *   **Action:** Downloads the specified PDF file from Google Drive.
        *   **Credentials:** Requires Google Drive API credentials.
        *   **Output:** Binary PDF data in the `data` property, along with other input properties passed through.

    *   **`Save Temporary PDF Locally` (Write Binary File Node - ID `save-temp-pdf`):**
        *   **Input:** Binary PDF data (`data` property), `googleDriveFileId`.
        *   **Action:** Saves the downloaded PDF to a temporary local directory on the n8n server (e.g., `./shipping_slips/temp/[googleDriveFileId].pdf`). The `./shipping_slips/temp/` directory is created if it doesn't exist.
        *   **Output:** Passes through input properties.

4.  **`Combine Downloaded PDFs` (Function Node - ID `combine-downloaded-pdfs`):**
    *   **Input:** An array of all items processed by the loop (i.e., by `Save Temporary PDF Locally`). Each item contains `googleDriveFileId`, `today`, and `batchFilePath`.
    *   **Actions:**
        *   Constructs the local paths to all downloaded temporary PDF files.
        *   Uses the `pdf-lib` library to merge all valid temporary PDFs into a single new PDF document.
        *   Saves the combined PDF to a local path on the n8n server (e.g., `./shipping_slips/compiled_slips_gdrive_YYYY-MM-DD.pdf`).
        *   Deletes the individual temporary PDF files from `./shipping_slips/temp/`.
        *   Optionally (currently commented out in code): Deletes the batch control file (`batch_gdrive_ids_...txt`).
    *   **Output:** JSON object with `status`, `outputPath` (local path to the compiled batch PDF), `processedPdfCount`.

5.  **`Upload Combined Batch PDF to Google Drive` (Google Drive Node - ID `gdrive-upload-combined-pdf`):**
    *   **Input:** `outputPath` from `Combine Downloaded PDFs`.
    *   **Action:** Uploads the final compiled batch PDF from the local server path to a specified Google Drive folder.
    *   **Credentials:** Requires Google Drive API credentials.
    *   **Configuration:** Destination folder path/ID on Google Drive needs to be set by user.

6.  **(Next Step - Manual by User) Print Compiled Batch PDF:**
    *   The local `outputPath` from `Combine Downloaded PDFs` can be used with a manually added "Execute Command" node to send the file to a thermal printer connected to the n8n server.
    *   Example command: `lp -d YOUR_PRINTER_NAME -o media=Custom.10x15cm "{{ $json.outputPath }}"`

## 3. Configuration Needed by User

*   **WooCommerce Node (`Get Order Details`, `Update Order Metadata`):** Set correct WooCommerce Site URL. Select/Create WooCommerce API credentials.
*   **HTTP Request Node (`HTML to PDF Converter`):** Select/Create HTTP Basic Auth credentials for `https://tools.mojoy.in/html-to-pdf`.
*   **Google Drive Nodes (`Upload PDF to Google Drive`, `Download PDF from Google Drive`, `Upload Combined Batch PDF to Google Drive`):** Select/Create Google Drive API credentials. Configure destination Folder IDs/Paths.
*   **Error Handling:** Create the separate "Error Notification Workflow" (see `Error_Notification_Workflow_Guide.md`). In this main workflow's settings, link it to the "Error Notification Workflow".
*   **(Optional) Printing:** Configure an "Execute Command" node if direct printing is used.
*   **Schedule Node (`Schedule Trigger for Batch Processing`):** Adjust timing for batch frequency.

## 4. Error Handling Strategy

Linked to a separate "Error Notification Workflow." Unhandled errors in this main workflow will trigger the error workflow.

## 5. File Management

*   **Individual Slips:** Google Drive.
*   **Batch Control Files (`batch_gdrive_ids_...txt`, `count_...txt`):** Local n8n server (`./shipping_slips/`).
*   **Temporary PDFs (batching):** Local n8n server (`./shipping_slips/temp/`), deleted after use.
*   **Combined Batch PDF:** Created locally, then uploaded to Google Drive. Local copy can be optionally deleted.
