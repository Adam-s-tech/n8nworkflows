Send new Spitogatos.gr listings with agent details to Google Sheets and Telegram via Apify

https://n8nworkflows.xyz/workflows/send-new-spitogatos-gr-listings-with-agent-details-to-google-sheets-and-telegram-via-apify-20185


# Send new Spitogatos.gr listings with agent details to Google Sheets and Telegram via Apify

### 1. Workflow Overview

This workflow automates the daily discovery, logging, and notification of real estate listings from Spitogatos.gr (Greece's largest property portal). Running every morning, it scrapes targeted locations based on user-defined parameters, filters out previously processed listings to prevent duplicates, logs every new property to a Google Sheet, and evaluates criteria (such as price-per-m² thresholds or price cuts) to broadcast a consolidated summary message to a Telegram chat.

The logic is divided into the following functional blocks:

- **1.1 Schedule and Parameter Initialization:** Triggers the workflow daily at 08:00 and defines global search criteria, targets, and integration endpoints.
- **1.2 Data Acquisition and Deduplication:** Interfaces with the Apify actor to scrape fresh Spitogatos.gr listings and filters out items processed in prior executions.
- **1.3 Data Transformation and Logging:** Normalizes property attributes and appends every new listing into a Google Sheets document.
- **1.4 Filtering, Aggregation, and Notification:** Applies deal criteria (max €/m², price reductions), aggregates the items, and dispatches a formatted HTML message to Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Parameter Initialization
This block initiates the execution pipeline on a timed schedule and sets configuration variables that govern search behavior and destination endpoints.

- **Nodes Involved:** `Every morning at 8`, `Set your search`

##### Node Details:
- **`Every morning at 8`**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node). Initiates the workflow execution sequence.
  - **Configuration:** Configured to trigger daily at hour 08:00.
  - **Expressions:** None.
  - **Input/Output Connections:** Output connects to `Set your search`.
  - **Edge Cases / Failures:** Missed executions if the n8n instance is offline at the scheduled time.

- **`Set your search`**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation/Variable Initialization). Establishes parameters for the scraper, filtering rules, and destination links.
  - **Configuration:** Assigns values for `locations`, `listingType`, `propertyType`, `priceMax`, `areaMin`, `roomsMin`, `postedWithin`, `maxPages`, `maxPricePerM2`, `alertOnPriceCuts`, `googleSheetUrl`, and `telegramChatId`.
  - **Expressions:** Static string, number, and boolean assignments.
  - **Input/Output Connections:** Input from `Every morning at 8`; output to `Find new listings on Spitogatos.gr`.
  - **Edge Cases / Failures:** Invalid placeholders (e.g., `PASTE_YOUR_GOOGLE_SHEET_URL_HERE`) will cause downstream integration failures if not updated before activation.

---

#### 2.2 Data Acquisition and Deduplication
This block executes the web scraper via Apify using the defined search criteria and removes duplicate listings based on unique listing identifiers.

- **Nodes Involved:** `Find new listings on Spitogatos.gr`, `Skip listings already sent`

##### Node Details:
- **`Find new listings on Spitogatos.gr`**
  - **Type & Technical Role:** `@apify/n8n-nodes-apify.apify` (External API Integration). Executes the remote Spitogatos.gr Scraper actor.
  - **Configuration:** Uses Actor ID `aAohRNwnunh3U1L6d` in "Run actor and get dataset" mode. 
  - **Expressions:** Dynamically constructs input parameters by reading from the upstream `Set your search` node (splitting comma-separated locations, mapping filters, setting bounds).
  - **Input/Output Connections:** Input from `Set your search`; output to `Skip listings already sent`.
  - **Edge Cases / Failures:** Apify authentication failure, insufficient account credits, actor timeouts, or payload structure changes on the target website.

- **`Skip listings already sent`**
  - **Type & Technical Role:** `n8n-nodes-base.removeDuplicates` (Stateful Filtering Node). Filters out items that have already been processed in previous workflow executions.
  - **Configuration:** Operates on the `removeItemsSeenInPreviousExecutions` mode, keyed against the listing ID.
  - **Expressions:** Deduplication value configured as `={{ $json.id }}`.
  - **Input/Output Connections:** Input from Apify actor; output to `Format property row`.
  - **Edge Cases / Failures:** Memory or persistence cache clearing in n8n can reset the seen-items registry, causing previously sent listings to be re-processed.

---

#### 2.3 Data Transformation and Logging
This block standardizes the raw property data fields into structured columns and records every new listing into a Google Sheet.

- **Nodes Involved:** `Format property row`, `Save every listing to Google Sheets`

##### Node Details:
- **`Format property row`**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation Node). Maps and normalizes raw scraper output fields into clean schema columns.
  - **Configuration:** `ignoreConversionErrors` enabled to gracefully handle missing numeric fields.
  - **Expressions:** Uses JavaScript expressions to parse prices (`Number($json.price)`), surface areas, calculate rounded price-per-m² (`Math.round(...)`), format floor levels, extract image links, determine price cuts, and capture current timestamps (`$now.toISODate()`).
  - **Input/Output Connections:** Input from `Skip listings already sent`; dual outputs to `Save every listing to Google Sheets` and `Keep matching deals`.
  - **Edge Cases / Failures:** Expression errors if unexpected data types arrive (e.g., non-numeric strings where numbers are expected).

- **`Save every listing to Google Sheets`**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (External API Integration). Appends structured property rows into a Google Sheet.
  - **Configuration:** Operation set to `append`. Document ID dynamically referenced from the setup node.
  - **Expressions:** Document ID evaluated via `={{ $('Set your search').first().json.googleSheetUrl }}`.
  - **Input/Output Connections:** Input from `Format property row`; terminal node for this branch.
  - **Edge Cases / Failures:** OAuth credential expiration, incorrect sheet URL/ID, or permission errors on the target Google Sheet.

---

#### 2.4 Filtering, Aggregation, and Notification
This block evaluates specific user-defined investment thresholds (such as maximum price per square meter or price drops), aggregates matching records, and broadcasts a summary to Telegram.

- **Nodes Involved:** `Keep matching deals`, `Combine into one message`, `Send listings to Telegram`

##### Node Details:
- **`Keep matching deals`**
  - **Type & Technical Role:** `n8n-nodes-base.filter` (Logical Control Node). Filters items based on pricing and price-cut criteria.
  - **Configuration:** Condition set to loose type validation with a custom JavaScript evaluator block.
  - **Expressions:** Evaluates whether `maxPricePerM2` is bypassed (0), whether the property's EUR/m² is under the cap, or whether `alertOnPriceCuts` is active and a price drop is flagged.
  - **Input/Output Connections:** Input from `Format property row`; output to `Combine into one message`.
  - **Edge Cases / Failures:** Empty incoming datasets or incorrect type comparisons if fields are undefined.

- **`Combine into one message`**
  - **Type & Technical Role:** `n8n-nodes-base.aggregate` (Data Aggregation Node). Consolidates all filtered items into a single array payload.
  - **Configuration:** Aggregates all item data into a destination field named `items`.
  - **Expressions:** None (handled natively by the aggregation operation).
  - **Input/Output Connections:** Input from `Keep matching deals`; output to `Send listings to Telegram`.
  - **Edge Cases / Failures:** Memory limits if an extremely large number of properties match the filter in a single execution.

- **`Send listings to Telegram`**
  - **Type & Technical Role:** `n8n-nodes-base.telegram` (External API Integration). Sends an HTML-formatted message summarizing matching listings to a specified Telegram chat.
  - **Configuration:** HTML parse mode enabled, web page previews disabled, attribution disabled.
  - **Expressions:** Uses a IIFE script to build an HTML-escaped string listing up to 20 properties with clickable links, pricing, specs, and agency details, appending a counter note if more than 20 properties match. Chat ID retrieved dynamically from `$('Set your search').first().json.telegramChatId`.
  - **Input/Output Connections:** Input from `Combine into one message`; terminal workflow node.
  - **Edge Cases / Failures:** Telegram Bot API restrictions, invalid chat IDs, message length limitations (exceeding Telegram's 4096 character limit if too many items bypass slicing limits), or unescaped HTML characters causing parsing errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - About` | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration overview. | None | None | ## Send new Spitogatos.gr property listings with agent phones to Google Sheets and Telegram<br><br>Every morning this workflow checks Spitogatos.gr, Greece's largest property portal, for listings published in the last 24 hours in the areas you pick. Each new listing is saved to Google Sheets with its price per m², floor, year built and the agent's name and phone number, and a summary is sent to Telegram.<br><br>Νέες αγγελίες ακινήτων από το Spitogatos.gr με τηλέφωνο μεσίτη, κάθε πρωί στο Telegram.<br><br>### Who it's for<br>Property investors, buyers relocating to Greece, and agents who watch a neighbourhood.<br><br>### How it works<br>1. **Schedule** runs the workflow every morning at 8:00.<br>2. **Set your search** holds your areas, budget, size and deal settings.<br>3. **Apify** runs the [Spitogatos.gr Scraper](https://apify.com/fayoussef/spitogatos-scraper?fpr=youssef) for listings first posted in the last 24 hours.<br>4. **Remove Duplicates** drops listings you already received.<br>5. Every new listing is saved to your Google Sheet.<br>6. Matching listings go to Telegram in one message.<br><br>### How to set up<br>1. Create a free [Apify account](https://apify.com/?fpr=youssef) and connect it in the **Apify** node (OAuth or API token).<br>2. Create an empty Google Sheet and paste its URL in **Set your search**.<br>3. Create a Telegram bot with @BotFather, connect it, and paste your chat id in **Set your search**.<br>4. Edit the areas and filters, run once, then activate.<br><br>### Requirements<br>- Apify account (the free plan works)<br>- Google Sheets and a Telegram bot<br><br>### How to customize<br>- Set `maxPricePerM2` (for example 2500) to alert only on listings under that price per m².<br>- Use `rent` in `listingType` to track rentals.<br>- Add more areas, in English or Greek, separated by commas. |
| `Sticky Note - Step 1` | `n8n-nodes-base.stickyNote` | Parameter configuration guide. | None | None | ### 1. Set your search<br>`locations`: area names in English or Greek, separated by commas (for example `Kolonaki, Pangrati`).<br>`listingType`: `sale` or `rent`.<br>`propertyType`: `apartment`, `studio`, `maisonette`, `detached`, `villa` or empty for all homes.<br>`maxPricePerM2`: 0 sends every new listing. Set a number to get only listings under it, plus price cuts when `alertOnPriceCuts` is true. |
| `Sticky Note - Step 2` | `n8n-nodes-base.stickyNote` | Scraping and deduplication notes. | None | None | ### 2. Find new listings<br>Runs the [Spitogatos.gr Scraper](https://apify.com/fayoussef/spitogatos-scraper?fpr=youssef) on your Apify account for listings first posted within `postedWithin`. `maxPages` keeps each run small.<br><br>**Skip listings already sent** remembers every listing id across runs, so you never see the same home twice.<br><br>Usage is billed per listing on your Apify account. |
| `Sticky Note - Step 3` | `n8n-nodes-base.stickyNote` | Logging and notification overview. | None | None | ### 3. Save and alert<br>Every new listing becomes one row in your Google Sheet with the agent's phone number (headers are created on the first run).<br><br>Matching listings reach Telegram in one message with up to 20 homes. The rest are in the sheet. |
| `Every morning at 8` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow daily at 08:00. | None | `Set your search` | |
| `Set your search` | `n8n-nodes-base.set` | Initializes search filters, limits, and integration endpoints. | `Every morning at 8` | `Find new listings on Spitogatos.gr` | |
| `Find new listings on Spitogatos.gr` | `@apify/n8n-nodes-apify.apify` | Executes the Spitogatos.gr Scraper actor on Apify. | `Set your search` | `Skip listings already sent` | |
| `Skip listings already sent` | `n8n-nodes-base.removeDuplicates` | Filters out previously processed listings using execution history. | `Find new listings on Spitogatos.gr` | `Format property row` | |
| `Format property row` | `n8n-nodes-base.set` | Normalizes and maps raw scraper properties into structured columns. | `Skip listings already sent` | `Save every listing to Google Sheets`, `Keep matching deals` | |
| `Save every listing to Google Sheets` | `n8n-nodes-base.googleSheets` | Appends all new normalized property records to a Google Sheet. | `Format property row` | None | |
| `Keep matching deals` | `n8n-nodes-base.filter` | Filters properties meeting user-defined pricing thresholds or price cuts. | `Format property row` | `Combine into one message` | |
| `Combine into one message` | `n8n-nodes-base.aggregate` | Consolidates filtered listings into a single array payload. | `Keep matching deals` | `Send listings to Telegram` | |
| `Send listings to Telegram` | `n8n-nodes-base.telegram` | Dispatches an HTML-formatted digest of deals to a Telegram chat. | `Combine into one message` | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Name it `Every morning at 8`.
   - Configure the interval rule to trigger daily at `8` hours.

2. **Add Search Parameter Definition Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Name it `Set your search`.
   - Add the following string, number, and boolean assignments:
     - `locations` (String): `Glyfada, Voula` (or desired locations separated by commas).
     - `listingType` (String): `sale`
     - `propertyType` (String): `apartment`
     - `priceMax` (Number): `350000`
     - `areaMin` (Number): `50`
     - `roomsMin` (Number): `2`
     - `postedWithin` (String): `24hours`
     - `maxPages` (Number): `3`
     - `maxPricePerM2` (Number): `0`
     - `alertOnPriceCuts` (Boolean): `true`
     - `googleSheetUrl` (String): `PASTE_YOUR_GOOGLE_SHEET_URL_HERE`
     - `telegramChatId` (String): `PASTE_YOUR_TELEGRAM_CHAT_ID_HERE`
   - Connect `Every morning at 8` to `Set your search`.

3. **Add the Apify Scraper Node:**
   - Add an **Apify** node (`@apify/n8n-nodes-apify.apify`).
   - Name it `Find new listings on Spitogatos.gr`.
   - Configure credentials for Apify (OAuth or API token).
   - Set Operation to `Run actor and get dataset` and Actor ID to `aAohRNwnunh3U1L6d`.
   - Set Custom Body using the expression:
     ```javascript
     ={{ JSON.stringify(Object.fromEntries(Object.entries({
       start_urls: [],
       locations: $json.locations.split(',').map(s => s.trim()).filter(Boolean),
       listing_type: $json.listingType || 'sale',
       category: 'residential',
       property_types: $json.propertyType.trim() || null,
       price_max: $json.priceMax || null,
       area_min: $json.areaMin || null,
       rooms_min: $json.roomsMin || null,
       posted_within: $json.postedWithin || '24hours',
       max_depth: $json.maxPages || 3
     }).filter(([k, v]) => v !== null))) }}
     ```
   - Connect `Set your search` output to this node.

4. **Add Deduplication Node:**
   - Add a **Remove Duplicates** node (`n8n-nodes-base.removeDuplicates`).
   - Name it `Skip listings already sent`.
   - Set Operation to `Remove items seen in previous executions`.
   - Set Dedupe Value expression to `={{ $json.id }}`.
   - Connect `Find new listings on Spitogatos.gr` output to this node.

5. **Add Property Formatting Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Name it `Format property row`.
   - Enable `Ignore Conversion Errors` in options.
   - Assign the following values:
     - `Price (EUR)`: `={{ Number($json.price) }}`
     - `Area (m2)`: `={{ Number($json.sq_meters) }}`
     - `EUR per m2`: `={{ Math.round(Number($json.pricePerSqMeters)) }}`
     - `Rooms`: `={{ $json.rooms ?? '' }}`
     - `Floor`: `={{ (f => f === '' ? '' : (isNaN(f) ? f : 'Floor ' + f))(String($json['Floor Level'] ?? $json.floorNumber ?? '')) }}`
     - `Built`: `={{ $json.year_of_construction || '' }}`
     - `Neighbourhood`: `={{ $json.geographiesByLevel_4_fullName || $json.geographiesByLevel_2_fullName || '' }}`
     - `Region`: `={{ $json.geographiesByLevel_2_fullName || '' }}`
     - `Price cut (%)`: `={{ String($json.priceReduced) === 'True' || $json.priceReduced === true ? Number($json.priceChangePercentage) || '' : '' }}`
     - `Agency`: `={{ $json.agency_name || '' }}`
     - `Contact`: `={{ $json.contact_name || '' }}`
     - `Phone`: `={{ $json.contact_telephone || $json.agency_telephone || '' }}`
     - `First listed`: `={{ $json.firstPublishDate || '' }}`
     - `Photo`: `={{ Array.isArray($json.images) ? ($json.images[0] || '') : ((String($json.images || '').match(/https?:[^'\",\\]\\s]+/) || [''])[0]) }}`
     - `Found on`: `={{ $now.toISODate() }}`
     - `Link`: `={{ $json.url }}`
   - Connect `Skip listings already sent` output to this node.

6. **Add Google Sheets Logging Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Name it `Save every listing to Google Sheets`.
   - Configure Google Sheets OAuth2 credentials.
   - Set Operation to `Append`.
   - Bind Document ID using expression: `={{ $('Set your search').first().json.googleSheetUrl }}`.
   - Set Sheet Name to use ID `gid=0` (or target worksheet name).
   - Connect the first main output of `Format property row` to this node.

7. **Add Deal Filtering Node:**
   - Add a **Filter** node (`n8n-nodes-base.filter`).
   - Name it `Keep matching deals`.
   - Configure condition with loose type validation using the custom expression:
     ```javascript
     ={{ (() => { const s = $('Set your search').first().json; const cap = Number(s.maxPricePerM2) || 0; if (!cap) return true; return $json['EUR per m2'] <= cap || (s.alertOnPriceCuts && $json['Price cut (%)'] !== ''); })() }}
     ```
   - Connect the second main output of `Format property row` to this node.

8. **Add Aggregation Node:**
   - Add an **Aggregate** node (`n8n-nodes-base.aggregate`).
   - Name it `Combine into one message`.
   - Set aggregation type to `Aggregate all item data` and destination field name to `items`.
   - Connect `Keep matching deals` output to this node.

9. **Add Telegram Notification Node:**
   - Add a **Telegram** node (`n8n-nodes-base.telegram`).
   - Name it `Send listings to Telegram`.
   - Configure Telegram Bot credentials.
   - Set Chat ID expression to `={{ $('Set your search').first().json.telegramChatId }}`.
   - Set Text expression:
     ```javascript
     ={{ (() => { const esc = s => String(s ?? '').replaceAll('&', '&amp;').replaceAll('<', '&lt;').replaceAll('>', '&gt;');
     const homes = $json.items;
     const lines = homes.slice(0, 20).map(h => {
       const cut = h['Price cut (%)'] !== '' ? ' · ↓' + h['Price cut (%)'] + '%' : '';
       return '🏠 <b><a href=\"' + h.Link + '\">€' + Number(h['Price (EUR)']).toLocaleString('en-US') + ' · ' + h['Area (m2)'] + ' m²</a></b> (€'
         + Number(h['EUR per m2']).toLocaleString('en-US') + '/m²)' + cut + '\\n'
         + esc([h.Rooms ? h.Rooms + ' rooms' : '', h.Floor, h.Built ? 'built ' + h.Built : ''].filter(Boolean).join(' · ')) + '\\n'
         + esc(h.Neighbourhood) + '\\n'
         + esc(h.Agency || h.Contact) + (h.Phone ? ' · ☎ ' + esc(h.Phone) : '');
     });
     return '<b>' + homes.length + ' new Spitogatos listing' + (homes.length === 1 ? '' : 's') + '</b>\\n\\n' + lines.join('\\n\\n')
       + (homes.length > 20 ? '\\n\\n+ ' + (homes.length - 20) + ' more in your Google Sheet' : '');
     })() }}
     ```
   - In Additional Fields, set `Parse Mode` to `HTML` and disable web page previews.
   - Connect `Combine into one message` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Actor Platform & Documentation | [Apify Spitogatos Scraper](https://apify.com/fayoussef/spitogatos-scraper?fpr=youssef) |
| Apify Platform Registration | [Apify Signup](https://apify.com/?fpr=youssef) |