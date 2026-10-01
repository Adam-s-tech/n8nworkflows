Create and send quote PDFs with OpenAI, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/create-and-send-quote-pdfs-with-openai--gmail-and-google-sheets-20028


# Create and send quote PDFs with OpenAI, Gmail and Google Sheets

### 1. Workflow Overview

This workflow functions as an automated quote and supplier management assistant tailored for German craft and trade businesses. It orchestrates communication between customers, suppliers, and internal databases to ingest quote requests, generate documents, and handle supplier price comparisons.

The logic is divided into six functional blocks:
- **1.1 Approved Customer Quote Generation:** Monitors Google Sheets for approved quote rows (`FREIGEGEBEN`), calculates financials, generates professional German quote text via OpenAI, builds a Google Docs document, converts it to PDF, emails it to the customer, and updates the status to `VERSENDET`.
- **1.2 Inbound Request Processing:** Triggers on new Gmail messages, parses message text and optional PDF attachments, extracts structured project details using OpenAI, and saves a new quote draft (`ENTWURF`) into Google Sheets.
- **1.3 Supplier Price Inquiry Creation:** Evaluates whether an inbound customer request includes a bill of quantities (`Leistungsverzeichnis`) and automatically generates a supplier price inquiry draft in a dedicated sheet tab.
- **1.4 Outbound Supplier Inquiry Dispatch:** Watches the supplier inquiries tab, sends approved inquiries (`FREIGEGEBEN`) to wholesalers via Gmail, and updates their status to `VERSENDET`.
- **1.5 Supplier Response Extraction:** Captures wholesaler email replies, parses commercial terms and pricing details using OpenAI, and records each offer in a supplier offers tab.
- **1.6 Multi-Offer Supplier Comparison:** Aggregates supplier offers for a specific inquiry, uses OpenAI to evaluate and compare them once at least two offers are present, and updates both individual offer rows and a central comparison record.

---

### 2. Block-by-Block Analysis

#### 2.1 Approved Customer Quote Generation
- **Overview:** This block processes approved customer quote records from Google Sheets, generates the formal offer text, converts it into a Google Drive document and PDF, sends the final file to the customer via email, and logs completion.
- **Nodes Involved:** `Angebote überwachen`, `Freigabe prüfen`, `Angebotsdaten berechnen`, `Angebotstext erstellen`, `OpenAI Chat Model`, `Angebot zusammenfuehren`, `Angebot in Tabelle aktualisieren`, `Angebotsdokument erstellen`, `Angebot als PDF exportieren`, `Angebots-PDF speichern`, `PDF-Link zusammenführen`, `PDF-Link in Tabelle speichern`, `PDF für E-Mail laden`, `Angebot per E-Mail senden`, `Status auf VERSENDET setzen`.

- **Node Details:**
  - `Angebote überwachen`
    - *Type:* `n8n-nodes-base.googleSheetsTrigger`
    - *Technical Role:* Polls a Google Sheets document every minute to detect changes in the main quote sheet.
    - *Configuration:* Document ID `18HTzdkvuVmXZbiT--jIjdJ8ykB7-5W2qNxpyMMVqdUg`, Sheet Name `Tabellenblatt1`.
    - *Input/Output:* Triggers execution; outputs row data including the `Status` column.
    - *Credentials:* `Google Sheets Trigger account` (`g68efoQqHcAKaTGM`).
    - *Edge Cases:* API rate limits or network dropouts during polling; invalid sheet IDs.
  - `Freigabe prüfen`
    - *Type:* `n8n-nodes-base.if`
    - *Technical Role:* Filters incoming rows to proceed only if `Status` equals `FREIGEGEBEN`.
    - *Configuration:* Condition checks `{{ $json.Status }} equals FREIGEGEBEN`.
    - *Input/Output:* Input from `Angebote überwachen`; output routes to `Angebotsdaten berechnen`.
    - *Edge Cases:* Typos in sheet status strings cause silent halts.
  - `Angebotsdaten berechnen`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Validates mandatory fields (`Kundenname`, `Kunden_E-Mail`, `Leistungsbeschreibung`, `Gesamt_Netto`), parses German/numeric financial values, computes gross totals based on a 19% VAT default, generates sequential quote numbers (`ANG-YYYY-XXXX`), and sets expiration dates.
    - *Configuration:* Runs custom JavaScript (`runOnceForEachItem`).
    - *Input/Output:* Input from `Freigabe prüfen`; output provides normalized financial and structural variables.
    - *Edge Cases:* Missing mandatory fields throw explicit errors; malformed numeric strings fail parsing.
  - `Angebotstext erstellen`
    - *Type:* `@n8n/n8n-nodes-langchain.agent`
    - *Technical Role:* Acts as an AI LangChain agent using the connected LLM to draft professional German trade quote text strictly adhering to provided parameters.
    - *Configuration:* Prompt uses mustache templates referencing calculated properties (`{{ $json.Angebotsnummer }}`, etc.).
    - *Input/Output:* Input from `Angebotsdaten berechnen`; output connects to `Angebot zusammenfuehren` and requires `OpenAI Chat Model`.
    - *Credentials:* Requires linked OpenAI chat model.
    - *Edge Cases:* LLM hallucination prevented via strict prompt constraints.
  - `OpenAI Chat Model`
    - *Type:* `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - *Technical Role:* Language model backend provider for the LangChain agent.
    - *Configuration:* Model set to `gpt-5-mini`.
    - *Input/Output:* Connects via `ai_languageModel` to `Angebotstext erstellen`.
    - *Credentials:* `OpenAI account` (`NhJg3mNoMQ0Z7X8y`).
    - *Edge Cases:* API quotas, rate limits, or transient OpenAI service outages.
  - `Angebot zusammenfuehren`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Merges base calculation data with the AI-generated quote text and updates internal status to `ERSTELLT`.
    - *Configuration:* Custom JavaScript using expression reference `$('Angebotsdaten berechnen').item.json`.
    - *Input/Output:* Input from `Angebotstext erstellen`; output routes to `Angebot in Tabelle aktualisieren`.
  - `Angebot in Tabelle aktualisieren`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Updates the Google Sheets row with the generated quote text and status.
    - *Configuration:* Operation `update`, matching column `Angebotsnummer`.
    - *Input/Output:* Input from `Angebot zusammenfuehren`; output routes to `Angebotsdokument erstellen`.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `Angebotsdokument erstellen`
    - *Type:* `n8n-nodes-base.googleDrive`
    - *Technical Role:* Creates a Google Docs file in Drive using the generated quote text.
    - *Configuration:* Operation `createFromText`, option `convertToGoogleDocument: true`, naming template `=Angebot_{{ $json.Angebotsnummer }}_{{ $json.Kundenname }}`.
    - *Input/Output:* Input from `Angebot in Tabelle aktualisieren`; output routes to `Angebot als PDF exportieren`.
    - *Credentials:* `Google Drive account v2` (`wVDiVJYAw395kB4a`).
  - `Angebot als PDF exportieren`
    - *Type:* `n8n-nodes-base.googleDrive`
    - *Technical Role:* Exports the newly created Google Doc as an application/pdf binary stream.
    - *Configuration:* Operation `download`, conversion option `docsToFormat: application/pdf`.
    - *Input/Output:* Input from `Angebotsdokument erstellen`; output routes to `Angebots-PDF speichern`.
    - *Credentials:* `Google Drive account v2` (`wVDiVJYAw395kB4a`).
  - `Angebots-PDF speichern`
    - *Type:* `n8n-nodes-base.googleDrive`
    - *Technical Role:* Uploads the exported PDF file to the root Google Drive folder.
    - *Configuration:* Operation `create` (implicit via name/binary upload), name `={{ $binary.data.fileName }}.pdf`, folder ID `root`.
    - *Input/Output:* Input from `Angebot als PDF exportieren`; output routes to `PDF-Link zusammenführen`.
    - *Credentials:* `Google Drive account v2` (`wVDiVJYAw395kB4a`).
  - `PDF-Link zusammenführen`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Re-combines base quote parameters with the public Google Drive `webViewLink` of the saved PDF.
    - *Configuration:* Custom JavaScript reading `$json.webViewLink`.
    - *Input/Output:* Input from `Angebots-PDF speichern`; output routes to `PDF-Link in Tabelle speichern`.
  - `PDF-Link in Tabelle speichern`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Writes the generated `PDF_Link` back into the corresponding Google Sheets row.
    - *Configuration:* Operation `update`, matching column `Angebotsnummer`.
    - *Input/Output:* Input from `PDF-Link zusammenführen`; output routes to `PDF für E-Mail laden`.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `PDF für E-Mail laden`
    - *Type:* `n8n-nodes-base.googleDrive`
    - *Technical Role:* Downloads the stored PDF file from Google Drive to attach it to the outgoing customer email.
    - *Configuration:* Operation `download`, File ID `={{ $('Angebots-PDF speichern').item.json.id }}`.
    - *Input/Output:* Input from `PDF-Link in Tabelle speichern`; output routes to `Angebot per E-Mail senden`.
    - *Credentials:* `Google Drive account v2` (`wVDiVJYAw395kB4a`).
  - `Angebot per E-Mail senden`
    - *Type:* `n8n-nodes-base.gmail`
    - *Technical Role:* Sends an HTML email containing the quote PDF attachment to the customer.
    - *Configuration:* Operation `send`, recipient `={{ $('PDF-Link zusammenführen').item.json['Kunden_E-Mail'] }}`, subject `=Ihr Angebot {{ ... }}`. Includes binary attachments.
    - *Input/Output:* Input from `PDF für E-Mail laden`; output routes to `Status auf VERSENDET setzen`.
    - *Credentials:* `Gmail OAuth2 API` (`Skhf9xJctEiDhKzm`).
  - `Status auf VERSENDET setzen`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Finalizes the quote lifecycle in Google Sheets by updating its status to `VERSENDET`.
    - *Configuration:* Operation `update`, mapping `Status: VERSENDET`, matching column `Angebotsnummer`.
    - *Input/Output:* Input from `Angebot per E-Mail senden`; terminal node for this branch.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).

---

#### 2.2 Inbound Request Processing
- **Overview:** Polls Gmail for incoming customer quote requests, downloads and parses attached PDF documents (such as bills of quantities), uses OpenAI information extraction to structure project data, and creates an initial quote draft row in Google Sheets.
- **Nodes Involved:** `Angebotsanfragen empfangen`, `Anfrage vollständig laden`, `E-Mail-Daten vorbereiten`, `PDF-Anhang vorbereiten`, `PDF-Anhang prüfen`, `PDF-Text auslesen`, `PDF-Inhalt vorbereiten`, `E-Mail und PDF zusammenführen`, `Angebotsdaten aus E-Mail extrahieren`, `OpenAI-Chatmodell Angebotsdaten`, `Angebotsanfrage prüfen`, `Bestehende Angebote abrufen`, `Angebotsentwurf vorbereiten`, `Angebotsentwurf in Tabelle speichern`, `Anfrage als gelesen markieren`.

- **Node Details:**
  - `Angebotsanfragen empfangen`
    - *Type:* `n8n-nodes-base.gmailTrigger`
    - *Technical Role:* Polls Gmail every minute for incoming messages with attachments excluding those with "Preisanfrage" in the subject.
    - *Configuration:* Query `has:attachment -subject:Preisanfrage`, poll interval 1 minute.
    - *Input/Output:* Triggers execution; outputs message metadata.
    - *Credentials:* `Gmail OAuth2 API` (`Skhf9xJctEiDhKzm`).
  - `Anfrage vollständig laden`
    - *Type:* `n8n-nodes-base.gmail`
    - *Technical Role:* Retrieves full message payload and downloads attachments.
    - *Configuration:* Operation `get`, `downloadAttachments: true`, message ID `={{ $json.id }}`.
    - *Input/Output:* Input from trigger; branches to `E-Mail-Daten vorbereiten` and `PDF-Anhang vorbereiten`.
    - *Credentials:* `Gmail OAuth2 API` (`Skhf9xJctEiDhKzm`).
  - `E-Mail-Daten vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Decodes HTML entities and sanitizes email body text, extracting clean sender details and message text.
    - *Configuration:* Custom JavaScript (`runOnceForEachItem`).
    - *Input/Output:* Input from `Anfrage vollständig laden`; output connects to merge node.
  - `PDF-Anhang vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Identifies and filters PDF files from binary attachments.
    - *Configuration:* Custom JavaScript checking MIME types and file extensions.
    - *Input/Output:* Input from `Anfrage vollständig laden`; output routes to `PDF-Anhang prüfen`.
  - `PDF-Anhang prüfen`
    - *Type:* `n8n-nodes-base.if`
    - *Technical Role:* Conditional branch routing based on whether a valid PDF attachment is present.
    - *Configuration:* Checks `{{ $json.Hat_PDF_Anhang }}`.
    - *Input/Output:* Input from `PDF-Anhang vorbereiten`; routes to `PDF-Text auslesen` (true) or `PDF-Inhalt vorbereiten` (false).
  - `PDF-Text auslesen`
    - *Type:* `n8n-nodes-base.extractFromFile`
    - *Technical Role:* Extracts raw text content from the PDF binary file.
    - *Configuration:* Operation `pdf`.
    - *Input/Output:* Input from `PDF-Anhang prüfen` (true branch); output routes to `PDF-Inhalt vorbereiten`.
  - `PDF-Inhalt vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Formats and sanitizes extracted PDF text and metadata.
    - *Configuration:* Custom JavaScript.
    - *Input/Output:* Input from `PDF-Text auslesen` or `PDF-Anhang prüfen` (false branch); output routes to `E-Mail und PDF zusammenführen`.
  - `E-Mail-Daten vorbereiten` (Second reference in flow) & `PDF-Inhalt vorbereiten` merge via:
  - `E-Mail und PDF zusammenführen`
    - *Type:* `n8n-nodes-base.merge`
    - *Technical Role:* Combines email body data and parsed PDF text into a single unified item.
    - *Configuration:* Mode `combine`, combination method `combineByPosition`.
    - *Input/Output:* Inputs from `E-Mail-Daten vorbereiten` and `PDF-Inhalt vorbereiten`; output routes to `Angebotsdaten aus E-Mail extrahieren`.
  - `Angebotsdaten aus E-Mail extrahieren`
    - *Type:* `@n8n/n8n-nodes-langchain.informationExtractor`
    - *Technical Role:* Uses OpenAI information extraction to pull structured fields (customer name, address, project type, scope of work, bill of quantities flags) from the combined email and PDF text.
    - *Configuration:* Schema defined with required attributes (`Kundenname`, `Strasse_Hausnummer`, `Ort`, `Projektart`, `Leistungsbeschreibung`, `Ist_Angebotsanfrage`, `PLZ`, `PDF_Ist_Leistungsverzeichnis`, `LV_Leistungen`, `LV_Materialien`, `LV_Mengen`).
    - *Input/Output:* Input from `E-Mail und PDF zusammenführen`; output connects to `Angebotsanfrage prüfen` and requires `OpenAI-Chatmodell Angebotsdaten`.
    - *Credentials:* Requires linked LLM.
  - `OpenAI-Chatmodell Angebotsdaten`
    - *Type:* `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - *Technical Role:* Language model backend for information extraction.
    - *Configuration:* Model set to `gpt-5-mini`.
    - *Input/Output:* Connects via `ai_languageModel` to `Angebotsdaten aus E-Mail extrahieren`.
    - *Credentials:* `OpenAI account` (`NhJg3mNoMQ0Z7X8y`).
  - `Angebotsanfrage prüfen`
    - *Type:* `n8n-nodes-base.if`
    - *Technical Role:* Ensures the email is genuinely a quote request before proceeding.
    - *Configuration:* Checks `{{ $json.output.Ist_Angebotsanfrage }} equals true`.
    - *Input/Output:* Input from extractor; output routes to `Bestehende Angebote abrufen`.
  - `Bestehende Angebote abrufen`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Fetches existing quotes from Google Sheets to determine the next sequential quote number.
    - *Configuration:* Operation `get` (all rows), sheet `Tabellenblatt1`.
    - *Input/Output:* Input from `Angebotsanfrage prüfen`; output routes to `Angebotsentwurf vorbereiten`.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `Angebotsentwurf vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Calculates the next sequential quote number (`ANG-YYYY-XXXX`) and structures the data payload for the new draft row.
    - *Configuration:* Custom JavaScript.
    - *Input/Output:* Input from `Bestehende Angebote abrufen`; output routes to `Angebotsentwurf in Tabelle speichern`.
  - `Angebotsentwurf in Tabelle speichern`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Appends the new quote draft row (`ENTWURF`) into Google Sheets.
    - *Configuration:* Operation `append`, sheet `Tabellenblatt1`.
    - *Input/Output:* Input from `Angebotsentwurf vorbereiten`; output branches to `Anfrage als gelesen markieren` and `Leistungsverzeichnis prüfen`.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `Anfrage als gelesen markieren`
    - *Type:* `n8n-nodes-base.gmail`
    - *Technical Role:* Marks the processed incoming Gmail message as read.
    - *Configuration:* Operation `markAsRead`, message ID from `E-Mail-Daten vorbereiten`.
    - *Input/Output:* Input from `Angebotsentwurf in Tabelle speichern`; terminal node for email management.
    - *Credentials:* `Gmail OAuth2 API` (`Skhf9xJctEiDhKzm`).

---

#### 2.3 Supplier Price Inquiry Creation
- **Overview:** Evaluates if an incoming request contains a bill of quantities (`Leistungsverzeichnis`) and compiles a structured supplier price inquiry draft in the `Haendleranfragen` sheet tab.
- **Nodes Involved:** `Leistungsverzeichnis prüfen`, `Preisanfrage vorbereiten`, `Preisanfrage in Tabelle speichern`.

- **Node Details:**
  - `Leistungsverzeichnis prüfen`
    - *Type:* `n8n-nodes-base.if`
    - *Technical Role:* Checks if the extracted data contains a valid bill of quantities flag (`PDF_Ist_Leistungsverzeichnis`).
    - *Configuration:* Condition checks `{{ $('Angebotsdaten aus E-Mail extrahieren').item.json.output.PDF_Ist_Leistungsverzeichnis }} equals true`.
    - *Input/Output:* Input from `Angebotsentwurf in Tabelle speichern`; output routes to `Preisanfrage vorbereiten`.
  - `Preisanfrage vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Generates a unique inquiry ID (`PA-<Angebotsnummer>`) and compiles a detailed German price inquiry text template incorporating extracted scope, materials, and quantities.
    - *Configuration:* Custom JavaScript.
    - *Input/Output:* Input from `Leistungsverzeichnis prüfen`; output routes to `Preisanfrage in Tabelle speichern`.
  - `Preisanfrage in Tabelle speichern`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Appends the new price inquiry record with status `ENTWURF` into the `Haendleranfragen` sheet tab.
    - *Configuration:* Operation `append`, Document ID `18HTzdkvuVmXZbiT--jIjdJ8ykB7-5W2qNxpyMMVqdUg`, Sheet Name `Haendleranfragen` (`gid=998073869`).
    - *Input/Output:* Input from `Preisanfrage vorbereiten`; terminal node for inquiry creation.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).

---

#### 2.4 Outbound Supplier Inquiry Dispatch
- **Overview:** Monitors the `Haendleranfragen` sheet tab for approved rows (`FREIGEGEBEN`), sends the compiled price inquiry emails to wholesalers via Gmail, and updates their status to `VERSENDET`.
- **Nodes Involved:** `Preisanfragen überwachen`, `Freigabe der Preisanfrage prüfen`, `Preisanfrage an Grosshaendler senden`, `Preisanfrage auf VERSENDET setzen`.

- **Node Details:**
  - `Preisanfragen überwachen`
    - *Type:* `n8n-nodes-base.googleSheetsTrigger`
    - *Technical Role:* Polls the `Haendleranfragen` sheet every minute for changes.
    - *Configuration:* Poll interval 1 minute, Sheet Name `Haendleranfragen`.
    - *Input/Output:* Triggers execution; outputs row data.
    - *Credentials:* `Google Sheets Trigger account` (`g68efoQqHcAKaTGM`).
  - `Freigabe der Preisanfrage prüfen`
    - *Type:* `n8n-nodes-base.if`
    - *Technical Role:* Filters rows to proceed only when `Status` equals `FREIGEGEBEN`.
    - *Configuration:* Checks `{{ $json.Status }} equals FREIGEGEBEN`.
    - *Input/Output:* Input from trigger; output routes to `Preisanfrage an Grosshaendler senden`.
  - `Preisanfrage an Grosshaendler senden`
    - *Type:* `n8n-nodes-base.gmail`
    - *Technical Role:* Emails the structured price inquiry text to the wholesaler's email address.
    - *Configuration:* Operation `send`, recipient `={{ $json['Grosshaendler_E-Mail'] }}`, subject `=Preisanfrage {{ $json.Preisanfrage_ID }} – {{ $json.Projektart }}`, email type `text`.
    - *Input/Output:* Input from `Freigabe der Preisanfrage prüfen`; output routes to `Preisanfrage auf VERSENDET setzen`.
    - *Credentials:* `Gmail OAuth2 API` (`Skhf9xJctEiDhKzm`).
  - `Preisanfrage auf VERSENDET setzen`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Updates the inquiry row status to `VERSENDET`.
    - *Configuration:* Operation `update`, mapping `Status: VERSENDET`, matching column `Preisanfrage_ID`.
    - *Input/Output:* Input from `Preisanfrage an Grosshaendler senden`; terminal node for this branch.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).

---

#### 2.5 Supplier Response Extraction
- **Overview:** Triggers on incoming wholesaler email replies containing "Preisanfrage", parses email headers and body content, uses OpenAI to extract commercial pricing terms into structured JSON, and saves each offer in the `Haendlerangebote` sheet tab.
- **Nodes Involved:** `Händlerantworten empfangen`, `Händlerantwort aufbereiten`, `Händlerangebot auswerten`, `Händlerangebot in Haendlerangebote speichern`, `Händlerantwort in Tabelle speichern` (Note: linked sequentially in flow).

- **Node Details:**
  - `Händlerantworten empfangen`
    - *Type:* `n8n-nodes-base.gmailTrigger`
    - *Technical Role:* Polls Gmail every minute for incoming messages with subject matching "Preisanfrage".
    - *Configuration:* Query `subject:Preisanfrage`, poll interval 1 minute.
    - *Input/Output:* Triggers execution; outputs raw email message.
    - *Credentials:* `Gmail OAuth2 API` (`Skhf9xJctEiDhKzm`).
  - `Händlerantwort aufbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Recursively parses Gmail multipart payloads and base64 payloads, extracts sender details, response text, and uses regex matching to find `Preisanfrage_ID` or `Angebotsnummer`.
    - *Configuration:* Custom JavaScript (`runOnceForEachItem`).
    - *Input/Output:* Input from trigger; output routes to `Händlerangebot auswerten`.
  - `Händlerangebot auswerten`
    - *Type:* `@n8n/n8n-nodes-langchain.openAi`
    - *Technical Role:* Analyzes wholesaler response text using OpenAI with strict JSON output formatting to extract pricing, VAT, delivery costs, lead time, validity, and position summaries.
    - *Configuration:* Model `gpt-5-mini`, JSON object response format.
    - *Input/Output:* Input from `Händlerantwort aufbereiten`; output routes to `Händlerangebot in Haendlerangebote speichern`.
    - *Credentials:* `OpenAI account` (`NhJg3mNoMQ0Z7X8y`).
  - `Händlerangebot in Haendlerangebote speichern`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Appends or updates the extracted supplier offer record in the `Haendlerangebote` sheet tab using a composite unique ID (`HA-<ID>-<MessageID>`).
    - *Configuration:* Operation `appendOrUpdate`, Sheet Name `Haendlerangebote` (`gid=348809967`), matching column `Haendlerangebot_ID`.
    - *Input/Output:* Input from OpenAI evaluation; output branches to `Händlerantwort in Tabelle speichern` and `Händlerangebote zur Preisanfrage abrufen`.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `Händlerantwort in Tabelle speichern`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Updates the inquiry record status to `ANGEBOT_AUSGEWERTET` in the `Haendleranfragen` tab.
    - *Configuration:* Operation `update`, matching column `Preisanfrage_ID`, sheet `Haendleranfragen`.
    - *Input/Output:* Input from `Händlerangebot in Haendlerangebote speichern`.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).

---

#### 2.6 Multi-Offer Supplier Comparison
- **Overview:** Retrieves all recorded offers for a given price inquiry, normalizes and sorts pricing data, triggers OpenAI to compare offers and recommend the best choice once at least two offers exist, and updates both offer rows and the `Haendlervergleiche` summary sheet.
- **Nodes Involved:** `Händlerangebote zur Preisanfrage abrufen`, `Händlerangebote für Vergleich vorbereiten`, `Mindestens zwei Händlerangebote prüfen`, `Händlerangebote vergleichen`, `Vergleichsergebnis für Tabellen vorbereiten`, `Angebotsaktualisierungen aufteilen`, `Händlerangebote mit Vergleich aktualisieren`, `Vergleichsdaten für Speicherung vorbereiten`, `Vergleichsergebnis in Haendlervergleiche speichern`, `Update row in sheet`.

- **Node Details:**
  - `Händlerangebote zur Preisanfrage abrufen`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Queries all wholesaler offers associated with the specific `Preisanfrage_ID`.
    - *Configuration:* Operation `get`, filter lookup column `Preisanfrage_ID`, Sheet Name `Haendlerangebote`.
    - *Input/Output:* Input from previous save node; output routes to `Händlerangebote für Vergleich vorbereiten`.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `Händlerangebote für Vergleich vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Deduplicates offers, parses German/international currency and delivery time formats into normalized numeric properties, and validates that at least two offers exist for the same inquiry.
    - *Configuration:* Custom JavaScript.
    - *Input/Output:* Input from sheet lookup; output routes to `Mindestens zwei Händlerangebote prüfen`.
  - `Mindestens zwei Händlerangebote prüfen`
    - *Type:* `n8n-nodes-base.if`
    - *Technical Role:* Halts comparison logic if fewer than two wholesaler offers are available.
    - *Configuration:* Checks `{{ $json.Vergleich_bereit }} equals true`.
    - *Input/Output:* Input from preparation node; output routes to `Händlerangebote vergleichen`.
  - `Händlerangebote vergleichen`
    - *Type:* `@n8n/n8n-nodes-langchain.openAi`
    - *Technical Role:* Utilizes OpenAI to evaluate all offers based on gross price, delivery time, delivery costs, payment terms, and completeness, returning a structured JSON recommendation.
    - *Configuration:* Model `gpt-5-mini`, JSON object output format.
    - *Input/Output:* Input from IF node; output routes to `Vergleichsergebnis für Tabellen vorbereiten`.
    - *Credentials:* `OpenAI account` (`NhJg3mNoMQ0Z7X8y`).
  - `Vergleichsergebnis für Tabellen vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Parses the LLM JSON response, validates fields, and constructs bulk update objects for all participating offer rows and the summary comparison record.
    - *Configuration:* Custom JavaScript.
    - *Input/Output:* Input from OpenAI comparison; output fans out to `Angebotsaktualisierungen aufteilen`, `Vergleichsdaten für Speicherung vorbereiten`, and `Update row in sheet`.
  - `Angebotsaktualisierungen aufteilen`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Splits the array of offer updates into individual items so each row can be updated independently.
    - *Configuration:* Custom JavaScript.
    - *Input/Output:* Input from `Vergleichsergebnis für Tabellen vorbereiten`; output routes to `Händlerangebote mit Vergleich aktualisieren`.
  - `Händlerangebote mit Vergleich aktualisieren`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Updates each supplier offer row in `Haendlerangebote` with its comparison status, whether it is the best offer, and AI recommendation notes.
    - *Configuration:* Operation `update`, matching column `Haendlerangebot_ID`, Sheet Name `Haendlerangebote`.
    - *Input/Output:* Input from split node; terminal update node.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `Vergleichsdaten für Speicherung vorbereiten`
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Formats the overall comparison result payload for storage in the summary sheet.
    - *Configuration:* Custom JavaScript.
    - *Input/Output:* Input from `Vergleichsergebnis für Tabellen vorbereiten`; output routes to `Vergleichsergebnis in Haendlervergleiche speichern`.
  - `Vergleichsergebnis in Haendlervergleiche speichern`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Appends or updates the central comparison record in the `Haendlervergleiche` sheet tab.
    - *Configuration:* Operation `appendOrUpdate`, matching column `Preisanfrage_ID`, Sheet Name `Haendlervergleiche` (`gid=1558224467`).
    - *Input/Output:* Input from preparation node; terminal storage node.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).
  - `Update row in sheet`
    - *Type:* `n8n-nodes-base.googleSheets`
    - *Technical Role:* Updates the inquiry status in the `Haendleranfragen` sheet tab to `ANGEBOTE_VERGLICHEN`.
    - *Configuration:* Operation `update`, matching column `Preisanfrage_ID`, Sheet Name `Haendleranfragen`.
    - *Input/Output:* Input from `Vergleichsergebnis für Tabellen vorbereiten`; terminal update node.
    - *Credentials:* `Google Sheets OAuth2 API` (`fe6UFtQyV9zXBJqy`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Angebote überwachen` | `googleSheetsTrigger` | Polls Google Sheets for quote changes | None | `Freigabe prüfen` | 1. Generate and send approved customer quotes |
| `Freigabe prüfen` | `if` | Filters rows where Status equals FREIGEGEBEN | `Angebote überwachen` | `Angebotsdaten berechnen` | 1. Generate and send approved customer quotes |
| `Angebotsdaten berechnen` | `code` | Validates mandatory fields & computes totals | `Freigabe prüfen` | `Angebotstext erstellen` | 1. Generate and send approved customer quotes |
| `Angebotstext erstellen` | `agent` | Drafts professional German quote text via AI | `Angebotsdaten berechnen` | `Angebot zusammenfuehren` | 1. Generate and send approved customer quotes |
| `OpenAI Chat Model` | `lmChatOpenAi` | LLM model backend for quote text generation | None | `Angebotstext erstellen` | 1. Generate and send approved customer quotes |
| `Angebot zusammenfuehren` | `code` | Combines base data with generated text | `Angebotstext erstellen` | `Angebot in Tabelle aktualisieren` | 1. Generate and send approved customer quotes |
| `Angebot in Tabelle aktualisieren` | `googleSheets` | Updates sheet with quote text | `Angebot zusammenfuehren` | `Angebotsdokument erstellen` | 1. Generate and send approved customer quotes |
| `Angebotsdokument erstellen` | `googleDrive` | Creates Google Doc from quote text | `Angebot in Tabelle aktualisieren` | `Angebot als PDF exportieren` | 1. Generate and send approved customer quotes |
| `Angebot als PDF exportieren` | `googleDrive` | Exports Google Doc as PDF binary | `Angebotsdokument erstellen` | `Angebots-PDF speichern` | 1. Generate and send approved customer quotes |
| `Angebots-PDF speichern` | `googleDrive` | Uploads PDF to Google Drive | `Angebot als PDF exportieren` | `PDF-Link zusammenführen` | 1. Generate and send approved customer quotes |
| `PDF-Link zusammenführen` | `code` | Merges quote data with Drive file URL | `Angebots-PDF speichern` | `PDF-Link in Tabelle speichern` | 1. Generate and send approved customer quotes |
| `PDF-Link in Tabelle speichern` | `googleSheets` | Stores public PDF link in sheet | `PDF-Link zusammenführen` | `PDF für E-Mail laden` | 1. Generate and send approved customer quotes |
| `PDF für E-Mail laden` | `googleDrive` | Downloads PDF binary for email attachment | `PDF-Link in Tabelle speichern` | `Angebot per E-Mail senden` | 1. Generate and send approved customer quotes |
| `Angebot per E-Mail senden` | `gmail` | Emails quote PDF to customer | `PDF für E-Mail laden` | `Status auf VERSENDET setzen` | 1. Generate and send approved customer quotes |
| `Status auf VERSENDET setzen` | `googleSheets` | Sets quote status to VERSENDET | `Angebot per E-Mail senden` | None | 1. Generate and send approved customer quotes |
| `Angebotsanfragen empfangen` | `gmailTrigger` | Polls Gmail for incoming quote requests | None | `Anfrage vollständig laden` | 2. Capture and structure incoming quote requests |
| `Anfrage vollständig laden` | `gmail` | Loads full email content and attachments | `Angebotsanfragen empfangen` | `E-Mail-Daten vorbereiten`, `PDF-Anhang vorbereiten` | 2. Capture and structure incoming quote requests |
| `E-Mail-Daten vorbereiten` | `code` | Sanitizes email body and sender details | `Anfrage vollständig laden` | `E-Mail und PDF zusammenführen` | 2. Capture and structure incoming quote requests |
| `PDF-Anhang vorbereiten` | `code` | Filters binary attachments for PDFs | `Anfrage vollständig laden` | `PDF-Anhang prüfen` | 2. Capture and structure incoming quote requests |
| `PDF-Anhang prüfen` | `if` | Checks if PDF attachment exists | `PDF-Anhang vorbereiten` | `PDF-Text auslesen`, `PDF-Inhalt vorbereiten` | 2. Capture and structure incoming quote requests |
| `PDF-Text auslesen` | `extractFromFile` | Extracts text from PDF attachment | `PDF-Anhang prüfen` | `PDF-Inhalt vorbereiten` | 2. Capture and structure incoming quote requests |
| `PDF-Inhalt vorbereiten` | `code` | Formats extracted PDF text and metadata | `PDF-Text auslesen`, `PDF-Anhang prüfen` | `E-Mail und PDF zusammenführen` | 2. Capture and structure incoming quote requests |
| `E-Mail und PDF zusammenführen` | `merge` | Combines email and PDF text contents | `E-Mail-Daten vorbereiten`, `PDF-Inhalt vorbereiten` | `Angebotsdaten aus E-Mail extrahieren` | 2. Capture and structure incoming quote requests |
| `Angebotsdaten aus E-Mail extrahieren` | `informationExtractor` | Extracts project data using OpenAI | `E-Mail und PDF zusammenführen` | `Angebotsanfrage prüfen` | 2. Capture and structure incoming quote requests |
| `OpenAI-Chatmodell Angebotsdaten` | `lmChatOpenAi` | LLM model for information extraction | None | `Angebotsdaten aus E-Mail extrahieren` | 2. Capture and structure incoming quote requests |
| `Angebotsanfrage prüfen` | `if` | Validates if email is a quote request | `Angebotsdaten aus E-Mail extrahieren` | `Bestehende Angebote abrufen` | 2. Capture and structure incoming quote requests |
| `Bestehende Angebote abrufen` | `googleSheets` | Fetches existing quotes for numbering | `Angebotsanfrage prüfen` | `Angebotsentwurf vorbereiten` | 2. Capture and structure incoming quote requests |
| `Angebotsentwurf vorbereiten` | `code` | Calculates next quote number and payload | `Bestehende Angebote abrufen` | `Angebotsentwurf in Tabelle speichern` | 2. Capture and structure incoming quote requests |
| `Angebotsentwurf in Tabelle speichern` | `googleSheets` | Saves new quote draft in sheet | `Angebotsentwurf vorbereiten` | `Anfrage als gelesen markieren`, `Leistungsverzeichnis prüfen` | 2. Capture and structure incoming quote requests |
| `Anfrage als gelesen markieren` | `gmail` | Marks processed Gmail message as read | `Angebotsentwurf in Tabelle speichern` | None | 2. Capture and structure incoming quote requests |
| `Leistungsverzeichnis prüfen` | `if` | Checks if request has a bill of quantities | `Angebotsentwurf in Tabelle speichern` | `Preisanfrage vorbereiten` | 3. Create supplier price requests from specifications |
| `Preisanfrage vorbereiten` | `code` | Compiles supplier price inquiry text | `Leistungsverzeichnis prüfen` | `Preisanfrage in Tabelle speichern` | 3. Create supplier price requests from specifications |
| `Preisanfrage in Tabelle speichern` | `googleSheets` | Saves supplier price inquiry draft | `Preisanfrage vorbereiten` | None | 3. Create supplier price requests from specifications |
| `Preisanfragen überwachen` | `googleSheetsTrigger` | Polls supplier inquiry sheet changes | None | `Freigabe der Preisanfrage prüfen` | 4. Send approved supplier price requests |
| `Freigabe der Preisanfrage prüfen` | `if` | Checks if inquiry status is FREIGEGEBEN | `Preisanfragen überwachen` | `Preisanfrage an Grosshaendler senden` | 4. Send approved supplier price requests |
| `Preisanfrage an Grosshaendler senden` | `gmail` | Emails price inquiry to wholesaler | `Freigabe der Preisanfrage prüfen` | `Preisanfrage auf VERSENDET setzen` | 4. Send approved supplier price requests |
| `Preisanfrage auf VERSENDET setzen` | `googleSheets` | Updates inquiry status to VERSENDET | `Preisanfrage an Grosshaendler senden` | None | 4. Send approved supplier price requests |
| `Händlerantworten empfangen` | `gmailTrigger` | Polls Gmail for wholesaler email replies | None | `Händlerantwort aufbereiten` | 5. Extract and store supplier responses |
| `Händlerantwort aufbereiten` | `code` | Parses email headers, body, and IDs | `Händlerantworten empfangen` | `Händlerangebot auswerten` | 5. Extract and store supplier responses |
| `Händlerangebot auswerten` | `openAi` | Extracts commercial offer terms via AI | `Händlerantwort aufbereiten` | `Händlerangebot in Haendlerangebote speichern` | 5. Extract and store supplier responses |
| `Händlerangebot in Haendlerangebote speichern` | `googleSheets` | Stores supplier offer in sheet | `Händlerangebot auswerten` | `Händlerantwort in Tabelle speichern`, `Händlerangebote zur Preisanfrage abrufen` | 5. Extract and store supplier responses |
| `Händlerantwort in Tabelle speichern` | `googleSheets` | Updates inquiry status to ANGEBOT_AUSGEWERTET | `Händlerangebot in Haendlerangebote speichern` | None | 5. Extract and store supplier responses |
| `Händlerangebote zur Preisanfrage abrufen` | `googleSheets` | Fetches all offers for an inquiry ID | `Händlerangebot in Haendlerangebote speichern` | `Händlerangebote für Vergleich vorbereiten` | 6. Compare supplier offers and write back the result |
| `Händlerangebote für Vergleich vorbereiten` | `code` | Normalizes pricing and checks offer count | `Händlerangebote zur Preisanfrage abrufen` | `Mindestens zwei Händlerangebote prüfen` | 6. Compare supplier offers and write back the result |
| `Mindestens zwei Händlerangebote prüfen` | `if` | Checks if at least 2 offers exist | `Händlerangebote für Vergleich vorbereiten` | `Händlerangebote vergleichen` | 6. Compare supplier offers and write back the result |
| `Händlerangebote vergleichen` | `openAi` | Compares wholesaler offers using AI | `Mindestens zwei Händlerangebote prüfen` | `Vergleichsergebnis für Tabellen vorbereiten` | 6. Compare supplier offers and write back the result |
| `Vergleichsergebnis für Tabellen vorbereiten` | `code` | Prepares bulk update payloads for comparison | `Händlerangebote vergleichen` | `Angebotsaktualisierungen aufteilen`, `Vergleichsdaten für Speicherung vorbereiten`, `Update row in sheet` | 6. Compare supplier offers and write back the result |
| `Angebotsaktualisierungen aufteilen` | `code` | Splits offer updates into individual items | `Vergleichsergebnis für Tabellen vorbereiten` | `Händlerangebote mit Vergleich aktualisieren` | 6. Compare supplier offers and write back the result |
| `Händlerangebote mit Vergleich aktualisieren` | `googleSheets` | Updates offer rows with comparison status | `Angebotsaktualisierungen aufteilen` | None | 6. Compare supplier offers and write back the result |
| `Vergleichsdaten für Speicherung vorbereiten` | `code` | Formats summary comparison payload | `Vergleichsergebnis für Tabellen vorbereiten` | `Vergleichsergebnis in Haendlervergleiche speichern` | 6. Compare supplier offers and write back the result |
| `Vergleichsergebnis in Haendlervergleiche speichern` | `googleSheets` | Saves comparison summary in sheet | `Vergleichsdaten für Speicherung vorbereiten` | None | 6. Compare supplier offers and write back the result |
| `Update row in sheet` | `googleSheets` | Updates inquiry status to ANGEBOTE_VERGLICHEN | `Vergleichsergebnis für Tabellen vorbereiten` | None | 6. Compare supplier offers and write back the result |
| `README - Template setup` | `stickyNote` | Overview documentation and setup guide | None | None | Generate and send quote PDFs with OpenAI, Gmail and Google Sheets |
| `Section 1 - Approved offers` | `stickyNote` | Block 1 documentation note | None | None | 1. Generate and send approved customer quotes |
| `Section 2 - Incoming requests` | `stickyNote` | Block 2 documentation note | None | None | 2. Capture and structure incoming quote requests |
| `Section 3 - Supplier request creation` | `stickyNote` | Block 3 documentation note | None | None | 3. Create supplier price requests from specifications |
| `Section 4 - Send supplier requests` | `stickyNote` | Block 4 documentation note | None | None | 4. Send approved supplier price requests |
| `Section 5 - Supplier responses` | `stickyNote` | Block 5 documentation note | None | None | 5. Extract and store supplier responses |
| `Section 6 - Compare supplier offers` | `stickyNote` | Block 6 documentation note | None | None | 6. Compare supplier offers and write back the result |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Google Sheets & Drive Setup
1. Create a Google Sheets workbook named `FLOW AI Angebote` containing four tabs:
   - `Tabellenblatt1` (Main quotes sheet)
   - `Haendleranfragen` (Supplier inquiries sheet)
   - `Haendlerangebote` (Supplier offers sheet)
   - `Haendlervergleiche` (Supplier comparisons sheet)
2. Create a designated Google Drive destination folder if desired (default uses root folder).

#### Step 2: Build Block 1 (Approved Customer Quote Generation)
1. Add a **Google Sheets Trigger** node (`Angebote überwachen`) polling `Tabellenblatt1` every minute.
2. Add an **If** node (`Freigabe prüfen`) checking if `{{ $json.Status }}` equals `FREIGEGEBEN`. Connect trigger output.
3. Add a **Code** node (`Angebotsdaten berechnen`) to validate mandatory fields, parse financials, and format dates. Connect true output of the If node.
4. Add an **AI Agent** node (`Angebotstext erstellen`) configured with system prompts and mustache expressions. Connect input from the Code node.
5. Attach an **OpenAI Chat Model** node (`OpenAI Chat Model`) configured to model `gpt-5-mini` to the AI Agent.
6. Add a **Code** node (`Angebot zusammenfuehren`) to combine base data and generated text. Connect output from the AI Agent.
7. Add a **Google Sheets** node (`Angebot in Tabelle aktualisieren`) updating `Tabellenblatt1` matching on `Angebotsnummer`.
8. Add a **Google Drive** node (`Angebotsdokument erstellen`) with operation `createFromText`, converting text to a Google Doc.
9. Add a **Google Drive** node (`Angebot als PDF exportieren`) downloading the file as `application/pdf`.
10. Add a **Google Drive** node (`Angebots-PDF speichern`) saving the PDF binary to the Drive root.
11. Add a **Code** node (`PDF-Link zusammenführen`) attaching the `webViewLink`.
12. Add a **Google Sheets** node (`PDF-Link in Tabelle speichern`) updating the `PDF_Link` column in `Tabellenblatt1`.
13. Add a **Google Drive** node (`PDF für E-Mail laden`) downloading the PDF binary using the stored file ID.
14. Add a **Gmail** node (`Angebot per E-Mail senden`) sending the HTML email with the PDF attachment.
15. Add a **Google Sheets** node (`Status auf VERSENDET setzen`) updating the status column to `VERSENDET`.

#### Step 3: Build Block 2 (Inbound Request Processing)
1. Add a **Gmail Trigger** node (`Angebotsanfragen empfangen`) querying `has:attachment -subject:Preisanfrage`.
2. Add a **Gmail** node (`Anfrage vollständig laden`) retrieving full message content and attachments (`downloadAttachments: true`).
3. Add a **Code** node (`E-Mail-Daten vorbereiten`) to clean and decode email text.
4. Add a **Code** node (`PDF-Anhang vorbereiten`) to filter binary attachments for PDF files.
5. Add an **If** node (`PDF-Anhang prüfen`) checking `{{ $json.Hat_PDF_Anhang }}`.
6. Add an **Extract From File** node (`PDF-Text auslesen`) operating on PDF files.
7. Add a **Code** node (`PDF-Inhalt vorbereiten`) to sanitize extracted PDF text.
8. Add a **Merge** node (`E-Mail und PDF zusammenführen`) combining email and PDF contents using `combineByPosition`.
9. Add an **Information Extractor** node (`Angebotsdaten aus E-Mail extrahieren`) using OpenAI (`gpt-5-mini`) with required project schema attributes.
10. Add an **If** node (`Angebotsanfrage prüfen`) verifying `Ist_Angebotsanfrage` is true.
11. Add a **Google Sheets** node (`Bestehende Angebote abrufen`) reading `Tabellenblatt1`.
12. Add a **Code** node (`Angebotsentwurf vorbereiten`) calculating sequential quote numbering (`ANG-YYYY-XXXX`).
13. Add a **Google Sheets** node (`Angebotsentwurf in Tabelle speichern`) appending the draft row (`ENTWURF`).
14. Add a **Gmail** node (`Anfrage als gelesen markieren`) marking the message as read.

#### Step 4: Build Block 3 (Supplier Price Inquiry Creation)
1. Add an **If** node (`Leistungsverzeichnis prüfen`) verifying `PDF_Ist_Leistungsverzeichnis` is true. Connect input from the draft save node.
2. Add a **Code** node (`Preisanfrage vorbereiten`) generating inquiry ID (`PA-...`) and inquiry template text.
3. Add a **Google Sheets** node (`Preisanfrage in Tabelle speichern`) appending the record to the `Haendleranfragen` sheet tab.

#### Step 5: Build Block 4 (Outbound Supplier Inquiry Dispatch)
1. Add a **Google Sheets Trigger** node (`Preisanfragen überwachen`) polling the `Haendleranfragen` tab every minute.
2. Add an **If** node (`Freigabe der Preisanfrage prüfen`) checking if `Status` equals `FREIGEGEBEN`.
3. Add a **Gmail** node (`Preisanfrage an Grosshaendler senden`) emailing the inquiry text to the wholesaler.
4. Add a **Google Sheets** node (`Preisanfrage auf VERSENDET setzen`) updating the status to `VERSENDET`.

#### Step 6: Build Block 5 (Supplier Response Extraction)
1. Add a **Gmail Trigger** node (`Händlerantworten empfangen`) querying `subject:Preisanfrage`.
2. Add a **Code** node (`Händlerantwort aufbereiten`) parsing multipart MIME structures and regex matching IDs.
3. Add an **OpenAI** node (`Händlerangebot auswerten`) using `gpt-5-mini` with JSON output mode to extract commercial offer terms.
4. Add a **Google Sheets** node (`Händlerangebot in Haendlerangebote speichern`) using `appendOrUpdate` on the `Haendlerangebote` tab.
5. Add a **Google Sheets** node (`Händlerantwort in Tabelle speichern`) updating the inquiry status to `ANGEBOT_AUSGEWERTET`.

#### Step 7: Build Block 6 (Multi-Offer Supplier Comparison)
1. Add a **Google Sheets** node (`Händlerangebote zur Preisanfrage abrufen`) fetching all offers matching `Preisanfrage_ID` from `Haendlerangebote`.
2. Add a **Code** node (`Händlerangebote für Vergleich vorbereiten`) normalizing prices, parsing lead times, and validating offer count.
3. Add an **If** node (`Mindestens zwei Händlerangebote prüfen`) checking `Vergleich_bereit` is true.
4. Add an **OpenAI** node (`Händlerangebote vergleichen`) using `gpt-5-mini` to evaluate and recommend the best offer.
5. Add a **Code** node (`Vergleichsergebnis für Tabellen vorbereiten`) parsing LLM output and preparing bulk updates.
6. Add a **Code** node (`Angebotsaktualisierungen aufteilen`) splitting updates into individual items.
7. Add a **Google Sheets** node (`Händlerangebote mit Vergleich aktualisieren`) updating offer rows in `Haendlerangebote`.
8. Add a **Code** node (`Vergleichsdaten für Speicherung vorbereiten`) formatting summary comparison data.
9. Add a **Google Sheets** node (`Vergleichsergebnis in Haendlervergleiche speichern`) appending or updating the `Haendlervergleiche` tab.
10. Add a **Google Sheets** node (`Update row in sheet`) updating the inquiry status to `ANGEBOTE_VERGLICHEN` in `Haendleranfragen`.

#### Credentials Configuration Required:
- **Gmail OAuth2 API** (for email triggers, reading, sending, and marking read)
- **Google Sheets / Google Sheets Trigger OAuth2 API** (for sheet automation across all tabs)
- **Google Drive OAuth2 API** (for document creation, PDF export, storage, and retrieval)
- **OpenAI API** (for agent text generation, information extraction, offer evaluation, and comparison)

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Generate and send quote PDFs with OpenAI, Gmail and Google Sheets | Official workflow template overview and architecture |
| Required Sheet Tabs | `Tabellenblatt1`, `Haendleranfragen`, `Haendlerangebote`, `Haendlervergleiche` |
| Primary AI Model | `gpt-5-mini` (Required across all LangChain and OpenAI nodes) |