# Setup Guide

This guide configures the sanitized invoice-validation workflow in this repository. The public export is disconnected from the original accounts and resources.

## 1. Requirements

Create or obtain access to:

- n8n;
- Gmail;
- Google Sheets;
- Slack; and
- Groq, or another chat-model provider that you deliberately substitute.

Use test invoices that contain no real customer or vendor data while you configure the workflow.

## 2. Import the Workflow

Import:

`workflows/invoice-validation-duplicate-detection.json`

Keep the workflow inactive during setup.

**Expected result:** n8n shows the full workflow and asks you to configure credential-dependent nodes.

## 3. Create a Dedicated Invoice Inbox or Filter

The original build used the Gmail Trigger with attachments enabled.

For a new deployment, do not process every message in a personal inbox. Use one of these controls:

- a dedicated accounts-payable mailbox;
- a Gmail label for invoices; or
- a narrow Gmail search/filter that selects the approved senders or subject pattern.

Configure the Gmail Trigger so that it downloads attachments.

**Expected result:** a test email with one PDF attachment reaches the workflow and exposes the attachment as binary data.

## 4. Configure PDF Extraction

Open `Extract from File`.

The workflow expects a PDF attachment in the binary property used by the node. The original export used `attachment_0`.

If your Gmail Trigger gives the file a different binary property name, update `Binary Property Name` in the extraction node.

**Expected result:** the node returns readable invoice text.

## 5. Configure the AI Extraction Step

Connect a supported chat-model credential to the model node.

The public prompt requires these fields:

- `vendor_name`
- `vendor_email`
- `invoice_number`
- `issue_date`
- `due_date`
- `customer_name`
- `line_items`
- `subtotal`
- `tax`
- `total_amount`
- `classification`

The agent must return valid JSON for an accepted invoice. It must not invent missing values.

The Code node parses the `output` string into a JSON object for later nodes.

**Expected result:** a complete sample invoice produces parseable JSON with the required keys.

## 6. Configure the Finance Review Slack Channel

Connect Slack to n8n and select a channel that finance staff can review.

The workflow uses Slack in two places:

1. as an AI tool for incomplete invoices and basic anomalies; and
2. as a normal Slack node for a matching invoice number.

Use a finance channel, not a general sales channel.

**Expected result:** a controlled test can post an exception message to the selected channel.

## 7. Create the Invoice Register

Create a Google Sheet with these columns:

- `Invoice Number`
- `Vendor Name`
- `Vendor Email`
- `Issue Date`
- `Due Date`
- `Customer Name`
- `Subtotal`
- `Tax`
- `Total Amount`
- `Classification`
- `Date Processed`

Connect Google Sheets credentials to both sheet nodes.

In `Get row(s) in sheet`, select the new spreadsheet and sheet. Keep the lookup column as `Invoice Number`.

In `Append row in sheet`, select the same spreadsheet and map each incoming field to the correct column.

**Expected result:** the lookup node can search the invoice register and the append node can add a clean test record.

## 8. Verify the Duplicate Branch

The original workflow checks whether the lookup returns an existing row.

Test two cases:

### New invoice number

Use an invoice number that is not in the sheet.

Expected path:

`Get row(s) in sheet` → `If` → `Append row in sheet`

### Existing invoice number

Run the same invoice again after the first record exists.

Expected path:

`Get row(s) in sheet` → `If` → Slack duplicate-review message

Do not automatically delete, reject, or pay an invoice only because one key matches. A duplicate candidate should be reviewed against vendor, amount, dates, purchase order, and payment history when those data sources exist.

## 9. Configure the Vendor Acknowledgement

Connect Gmail credentials to the final Gmail node.

Review the subject and body. The public copy says only that the invoice was recorded for review. It does not tell the vendor that payment is approved.

**Expected result:** a test invoice on the accepted path sends an acknowledgement to the sample vendor address after the Wait node.

## 10. Test the Minimum Scenario Set

Use synthetic invoices and test at least these cases:

1. complete new invoice;
2. same invoice number submitted twice;
3. missing vendor email;
4. missing invoice number;
5. subtotal plus tax does not equal total;
6. due date earlier than issue date;
7. non-invoice PDF;
8. malformed model output; and
9. Gmail, Sheets, Slack, or model-service failure.

Record the expected branch for every case before you activate the workflow.

## 11. Production Hardening

The portfolio workflow demonstrates the architecture. For production, strengthen the controls.

### Move critical validation out of the model

Use deterministic Code or If nodes for arithmetic, date comparison, required-field checks, and strict schema validation. Let AI help with document interpretation and classification, not final financial control.

### Use stronger duplicate detection

Invoice number alone is not enough for every supplier. Consider a composite key or review rule that includes:

- normalized vendor identity;
- normalized invoice number;
- total amount;
- invoice date;
- purchase-order reference;
- currency; and
- attachment hash.

Near-duplicate detection should create a review candidate, not an automatic rejection.

### Add a system of record

Replace Google Sheets when the business needs stronger concurrency, permissions, audit history, approval states, or integration with accounting software.

### Add error handling

Add an n8n Error Workflow or equivalent monitoring path. Decide how to handle retries, malformed files, unavailable services, and repeated trigger events.

### Add human approval before financial action

Do not connect this portfolio pattern directly to payment release without an explicit, auditable approval design and the controls required by the organization.

## 12. Activate the Workflow

Activate only after the test cases pass with synthetic data.

Monitor the first production executions and keep a clear exception queue for finance staff.
