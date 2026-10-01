Launch and sell ebooks with Google Gemini, PDFShift, Google Drive and email

https://n8nworkflows.xyz/workflows/launch-and-sell-ebooks-with-google-gemini--pdfshift--google-drive-and-email-19817


# Launch and sell ebooks with Google Gemini, PDFShift, Google Drive and email

### 1. Workflow Overview

This workflow automates the complete lifecycle of creating, publishing, and marketing a digital e-book product based on an incoming topic idea, alongside a secondary branch that handles customer order fulfillment and post-purchase engagement. 

The process is divided into two primary execution flows:
1. **Product Generation and Launch Pipeline:** Receives a product idea via webhook, uses AI to research and structure the book, writes the manuscript, converts it to PDF, uploads it to cloud storage, deploys a sales page and product listing on an external storefront API, and generates marketing copy.
2. **Order Fulfillment and Follow-Up Pipeline:** Listens for a payment success webhook, emails the digital download link to the customer, pauses for three days, and sends a follow-up email requesting feedback or a testimonial.

---

### 2. Block-by-Block Analysis

#### 2.1 Product Creation & Content Generation
- **Overview:** Receives the raw product concept, performs structured market research, outlines the e-book, and drafts the full manuscript in Markdown format.
- **Nodes Involved:** 
  - `POST Launch Product Webhook`
  - `Research & Outline Agent`
  - `Content Writing Agent`

- **Node Details:**
  - **POST Launch Product Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Version 2) – Acts as the primary entry point for the product creation flow.
    - *Configuration Choices:* Listens for incoming HTTP `POST` requests on the path `launch-product`.
    - *Key Expressions:* Accesses incoming data using `$json.body.topic` and `$json.body.product_url`.
    - *Input/Output:* No inputs; outputs to `Research & Outline Agent`.
    - *Edge Cases / Failure Types:* Webhook timeout, missing expected payload properties (`topic` or `product_url`).

  - **Research & Outline Agent**
    - *Type and Technical Role:* `n8n-nodes-base.googleGemini` (Version 1) – Acts as a Product Manager AI agent to structure the e-book concept.
    - *Configuration Choices:* Uses model `models/gemini-1.5-pro` with a localized Indonesian prompt demanding a strict, raw JSON response containing `title`, `target_audience`, `price`, and a chapter `outline`.
    - *Key Expressions:* `={{ 'Anda adalah Product Manager. Berdasarkan ide berikut: "' + $json.body.topic + '", buat riset produk ringkas...' }}`
    - *Input/Output:* Input from `POST Launch Product Webhook`; output to `Content Writing Agent`.
    - *Credentials:* `Google Gemini API Account` (`googleGeminiApi`).
    - *Edge Cases / Failure Types:* API rate limits, invalid JSON returned by the model breaking downstream property references.

  - **Content Writing Agent**
    - *Type and Technical Role:* `n8n-nodes-base.googleGemini` (Version 1) – Acts as a professional Copywriter & Author to generate the manuscript.
    - *Configuration Choices:* Uses model `models/gemini-1.5-pro`. It consumes the chapter outline array produced by the previous agent.
    - *Key Expressions:* `={{ 'Anda adalah profesional Copywriter & Author. Berdasarkan outline berikut:\n' + JSON.stringify($json.outline) + '\n\nBuatkan isi naskah lengkap e-book...' }}`
    - *Input/Output:* Input from `Research & Outline Agent`; output to `Convert to PDF API Request`.
    - *Credentials:* `Google Gemini API Account` (`googleGeminiApi`).
    - *Edge Cases / Failure Types:* Token limit truncation for long books, generation timeouts.

---

#### 2.2 Document Compilation & Storage
- **Overview:** Converts the Markdown manuscript into a formatted PDF file and stores it securely in cloud storage for customer delivery.
- **Nodes Involved:** 
  - `Convert to PDF API Request`
  - `Upload PDF to Drive`

- **Node Details:**
  - **Convert to PDF API Request**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Version 4.2) – Communicates with the PDFShift API to render HTML/Markdown into a downloadable PDF.
    - *Configuration Choices:* Executes a `POST` request to `https://api.pdfshift.io/v3/convert/pdf` using JSON payload configuration.
    - *Key Expressions:* `={{ JSON.stringify({ "source": $json.text }) }}`
    - *Input/Output:* Input from `Content Writing Agent`; output to `Upload PDF to Drive`.
    - *Credentials:* `PDF API Key` (`httpHeaderAuth`).
    - *Edge Cases / Failure Types:* External API downtime, invalid markup format causing rendering failure, authentication rejection.

  - **Upload PDF to Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Version 3) – Uploads binary file outputs to user storage.
    - *Configuration Choices:* Set to `upload` operation, utilizing binary data from the upstream HTTP request. Dynamically names the file using the book title from the initial AI agent.
    - *Key Expressions:* File name: `={{ $('Research & Outline Agent').item.json.title + '.pdf' }}`
    - *Input/Output:* Input from `Convert to PDF API Request`; output to `Sales Copy Generation Agent`.
    - *Credentials:* `Google Drive Account` (`googleDriveOAuth2Api`).
    - *Edge Cases / Failure Types:* Insufficient storage space, permission scopes missing on the Google account.

---

#### 2.3 Store Deployment & Marketing
- **Overview:** Generates conversion-focused sales copy, registers the product on an external digital storefront, and creates promotional content for social channels.
- **Nodes Involved:** 
  - `Sales Copy Generation Agent`
  - `Deploy Product to Store API`
  - `Marketing Content Agent`

- **Node Details:**
  - **Sales Copy Generation Agent**
    - *Type and Technical Role:* `n8n-nodes-base.googleGemini` (Version 1) – Generates sales copy using the AIDA framework.
    - *Configuration Choices:* Uses model `models/gemini-1.5-pro` to output structured HTML sales copy based on the book title.
    - *Key Expressions:* `={{ 'Buatkan deskripsi penjualan (Sales Copy) menggunakan metode AIDA...' }}`
    - *Input/Output:* Input from `Upload PDF to Drive`; output to `Deploy Product to Store API`.
    - *Credentials:* `Google Gemini API Account` (`googleGeminiApi`).
    - *Edge Cases / Failure Types:* Model generation failures.

  - **Deploy Product to Store API**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Version 4.2) – Integrates with the Lynk.id API to publish the digital product.
    - *Configuration Choices:* Executes a `POST` request to `https://api.lynk.id/v1/products` mapping price, title, HTML description, and Google Drive download links.
    - *Key Expressions:* Constructs payload referencing data from multiple upstream nodes (`Research & Outline Agent`, `Sales Copy Generation Agent`, and `Upload PDF to Drive`).
    - *Input/Output:* Input from `Sales Copy Generation Agent`; output to `Marketing Content Agent`.
    - *Credentials:* `Store API Auth` (`httpHeaderAuth`).
    - *Edge Cases / Failure Types:* API schema mismatches, rejected pricing thresholds, expired authentication tokens.

  - **Marketing Content Agent**
    - *Type and Technical Role:* `n8n-nodes-base.googleGemini` (Version 1) – Generates promotional posts for social platforms (Threads/Instagram).
    - *Configuration Choices:* Uses model `models/gemini-1.5-pro` to generate storytelling-driven marketing posts with embedded Call-to-Action links.
    - *Key Expressions:* `={{ 'Buatkan 3 konten promosi Threads/Instagram... Sertakan Call to Action ke link: ' + $json.body.product_url }}`
    - *Input/Output:* Input from `Deploy Product to Store API`; terminal node for this branch.
    - *Credentials:* `Google Gemini API Account` (`googleGeminiApi`).
    - *Edge Cases / Failure Types:* Missing URL parameters from the initial webhook.

---

#### 2.4 Fulfillment & Follow-Up Pipeline
- **Overview:** Listens for successful payment events, delivers purchased files directly to the customer via email, waits three days, and requests a testimonial.
- **Nodes Involved:** 
  - `POST Payment Success Webhook`
  - `Send Product Files Email`
  - `Wait 3 Days`
  - `Send Testimonial Request Email`

- **Node Details:**
  - **POST Payment Success Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Version 2) – Entry point for transaction notifications.
    - *Configuration Choices:* Listens for incoming HTTP `POST` requests on the path `payment-success-webhook`.
    - *Key Expressions:* Accesses customer fields such as `$json.body.customer_name`, `$json.body.customer_email`, `$json.body.download_url`, and `$json.body.product_name`.
    - *Input/Output:* No inputs; outputs to `Send Product Files Email`.
    - *Edge Cases / Failure Types:* Malformed JSON payloads from the payment gateway.

  - **Send Product Files Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Version 2.1) – Sends the digital product delivery email.
    - *Configuration Choices:* Sends plain text emails using configured SMTP/Email settings. Sender address defaults to `user@example.com` (requires customization).
    - *Key Expressions:* Dynamic recipient (`$json.body.customer_email`), subject (`Akses Download: ...`), and message body.
    - *Input/Output:* Input from `POST Payment Success Webhook`; output to `Wait 3 Days`.
    - *Credentials:* SMTP / Email Send credentials.
    - *Edge Cases / Failure Types:* Invalid recipient email addresses, SMTP authentication failure, rate limiting by the mail server.

  - **Wait 3 Days**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Version 1.1) – Pauses workflow execution for a defined duration.
    - *Configuration Choices:* Configured to wait for an amount of `3` days.
    - *Input/Output:* Input from `Send Product Files Email`; output to `Send Testimonial Request Email`.
    - *Edge Cases / Failure Types:* Server reboots or workflow deactivations during the wait period causing delays or cleared execution states (depending on n8n persistence settings).

  - **Send Testimonial Request Email**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Version 2.1) – Sends a follow-up email asking for feedback.
    - *Configuration Choices:* Sends plain text emails using configured SMTP/Email settings.
    - *Key Expressions:* References original customer details using node-resolving syntax: `{{ $('POST Payment Success Webhook').item.json.body.customer_name }}`.
    - *Input/Output:* Input from `Wait 3 Days`; terminal node for this branch.
    - *Credentials:* SMTP / Email Send credentials.
    - *Edge Cases / Failure Types:* Expired node references if execution caching behaves unexpectedly.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| POST Launch Product Webhook | n8n-nodes-base.webhook | Entry point for product creation idea | None | Research & Outline Agent | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Capture and draft product |
| Research & Outline Agent | n8n-nodes-base.googleGemini | Product Manager agent to generate outline and metadata | POST Launch Product Webhook | Content Writing Agent | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Capture and draft product |
| Content Writing Agent | n8n-nodes-base.googleGemini | Author agent to write full Markdown manuscript | Research & Outline Agent | Convert to PDF API Request | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Capture and draft product |
| Convert to PDF API Request | n8n-nodes-base.httpRequest | Converts Markdown into PDF via PDFShift API | Content Writing Agent | Upload PDF to Drive | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Create and store PDF |
| Upload PDF to Drive | n8n-nodes-base.googleDrive | Uploads rendered PDF to Google Drive | Convert to PDF API Request | Sales Copy Generation Agent | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Create and store PDF |
| Sales Copy Generation Agent | n8n-nodes-base.googleGemini | Generates AIDA-format sales copy in HTML | Upload PDF to Drive | Deploy Product to Store API | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Launch and promote store |
| Deploy Product to Store API | n8n-nodes-base.httpRequest | Publishes product to store via Lynk.id API | Sales Copy Generation Agent | Marketing Content Agent | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Launch and promote store |
| Marketing Content Agent | n8n-nodes-base.googleGemini | Creates social media promotion posts | Deploy Product to Store API | None | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Launch and promote store |
| POST Payment Success Webhook | n8n-nodes-base.webhook | Entry point for successful order fulfillment | None | Send Product Files Email | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Deliver purchase follow-up |
| Send Product Files Email | n8n-nodes-base.emailSend | Emails download links to the customer | POST Payment Success Webhook | Wait 3 Days | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Deliver purchase follow-up |
| Wait 3 Days | n8n-nodes-base.wait | Pauses execution for 3 days | Send Product Files Email | Send Testimonial Request Email | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Deliver purchase follow-up |
| Send Testimonial Request Email | n8n-nodes-base.emailSend | Requests feedback/testimonials post-purchase | Wait 3 Days | None | Agentic AI - Digital Product Launchpad (Step 1-6)<br>Deliver purchase follow-up |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps sequentially to rebuild the entire workflow inside an n8n instance:

1. **Create the Product Webhook Trigger:**
   - Add a **Webhook** node. Set the HTTP Method to `POST` and path to `launch-product`. Name it `POST Launch Product Webhook`.
2. **Add the Research Agent:**
   - Add a **Google Gemini** node connected to the output of the webhook. Set the model to `models/gemini-1.5-pro`. Configure the prompt to return structured JSON containing the title, target audience, price, and outline based on `$json.body.topic`. Attach a Google Gemini API credential. Name it `Research & Outline Agent`.
3. **Add the Writing Agent:**
   - Add a second **Google Gemini** node. Connect it to the output of `Research & Outline Agent`. Set the model to `models/gemini-1.5-pro`. Set the prompt to consume the outline JSON and return full Markdown content (`$json.outline`). Name it `Content Writing Agent`.
4. **Configure PDF Conversion:**
   - Add an **HTTP Request** node connected to `Content Writing Agent`. Set the method to `POST`, URL to `https://api.pdfshift.io/v3/convert/pdf`, and body content type to JSON with expression `={{ JSON.stringify({ "source": $json.text }) }}`. Configure generic HTTP Header Authentication using your PDFShift API key credential. Name it `Convert to PDF API Request`.
5. **Set Up Google Drive Upload:**
   - Add a **Google Drive** node connected to the PDF request node. Set the operation to `upload`. Configure the file name expression using `={{ $('Research & Outline Agent').item.json.title + '.pdf' }}` and map the file content from binary data (`data`). Connect your Google Drive OAuth2 account. Name it `Upload PDF to Drive`.
6. **Generate Sales Copy:**
   - Add a **Google Gemini** node connected to the Google Drive node. Use model `models/gemini-1.5-pro`. Prompt the model to write AIDA HTML sales copy using the title from the research step (`={{ $('Research & Outline Agent').item.json.title }}`). Name it `Sales Copy Generation Agent`.
7. **Deploy Product to Store:**
   - Add an **HTTP Request** node connected to the sales copy agent. Set the method to `POST`, URL to `https://api.lynk.id/v1/products`, and specify a JSON body containing `name`, `price`, `description` (mapped to `$json.text`), and `file_url` (mapped to Google Drive's web view link). Add HTTP authentication credentials. Name it `Deploy Product to Store API`.
8. **Generate Marketing Content:**
   - Add a final **Google Gemini** node in the primary sequence, connected to the store deployment node. Use model `models/gemini-1.5-pro` to create 3 promotional posts referencing `$json.body.product_url`. Name it `Marketing Content Agent`.
9. **Create the Fulfillment Webhook Trigger:**
   - Add a separate **Webhook** node. Set the method to `POST` and path to `payment-success-webhook`. Name it `POST Payment Success Webhook`.
10. **Configure Email Delivery:**
    - Add an **Email Send** node connected to the payment webhook. Configure recipient email (`$json.body.customer_email`), subject, and body text containing the download link (`$json.body.download_url`). Provide verified SMTP/Email credentials and replace placeholder sender emails with a verified address. Name it `Send Product Files Email`.
11. **Set Up the Wait Period:**
    - Add a **Wait** node connected to the delivery email node. Set the amount to `3` and unit to `days`. Name it `Wait 3 Days`.
12. **Configure Follow-Up Email:**
    - Add a second **Email Send** node connected to the wait node. Configure recipient email using an expression pointing back to the payment webhook payload (`{{ $('POST Payment Success Webhook').item.json.body.customer_email }}`). Set the message to request a testimonial. Use the same SMTP credentials. Name it `Send Testimonial Request Email`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Project Title & Scope | Launch and sell ebooks with Google Gemini, PDFShift, Google Drive, and email. |
| Workflow Origin | Designed as an agentic AI digital product launchpad handling end-to-end automation from raw idea to deployment and CRM follow-up. |