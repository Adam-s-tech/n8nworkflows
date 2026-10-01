Sync Calendly bookings with Google Calendar, Airtable CRM, and Slack

https://n8nworkflows.xyz/workflows/sync-calendly-bookings-with-google-calendar--airtable-crm--and-slack-19719


# Sync Calendly bookings with Google Calendar, Airtable CRM, and Slack

### 1. Workflow Overview

This workflow automates the processing of new Calendly bookings by synchronizing schedule data with Google Calendar, updating an Airtable CRM, and broadcasting a formatted notification to a Slack channel. It eliminates manual data entry for sales and operations teams when a meeting is booked.

The execution logic is divided into four main functional blocks:
- **1.1 Input Reception & Normalization:** Listens for Calendly booking webhooks, pauses briefly to allow external synchronization, and structures raw payload data into clean operational variables.
- **1.2 Calendar Synchronization:** Searches Google Calendar for an event matching the booking timeframe, filters by attendee or Calendly UUID, and updates the event with attendees.
- **1.3 CRM Processing:** Searches Airtable for an existing contact by email, creates or updates the contact record, packages the meeting payload, checks for duplicate meeting logs using the Calendly event ID, and logs new meetings.
- **1.4 Team Notification:** Formats the compiled meeting details, client answers, and platform links into an alert and posts it to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Captures incoming webhooks from Calendly, implements a short delay to accommodate calendar sync timing, and maps nested payloads into standard variables.
- **Nodes Involved:** `New Calendly Booking`, `Wait for Calendar Sync`, `Extract Booking Data`.
- **Node Details:**
  - **New Calendly Booking**
    - *Type & Role:* `n8n-nodes-base.calendlyTrigger` (Webhook trigger). Listens for the `invitee.created` event via OAuth2.
    - *Configuration:* Authentication set to OAuth2 (`Calendly account`).
    - *Key Expressions:* None (Webhook input).
    - *Connections:* Input: None (Trigger); Output: `Wait for Calendar Sync`.
    - *Edge Cases:* Webhook delivery failure if Calendly API credentials expire or network interruptions occur.
  - **Wait for Calendar Sync**
    - *Type & Role:* `n8n-nodes-base.wait` (Flow control). Delays execution to ensure the corresponding calendar event is generated.
    - *Configuration:* Amount: `1`, Unit: `minutes`.
    - *Key Expressions:* None.
    - *Connections:* Input: `New Calendly Booking`; Output: `Extract Booking Data`.
    - *Edge Cases:* None.
  - **Extract Booking Data**
    - *Type & Role:* `n8n-nodes-base.set` (Data transformation). Extracts and sanitizes properties from the raw webhook payload.
    - *Configuration:* Mapping mode defines custom string and array fields.
    - *Key Expressions:* 
      - `calendly_event_uuid`: `={{ $json.payload.scheduled_event.uri.split('/').pop() }}`
      - `invitee_email`: `={{ ($json.payload.email || '').trim().toLowerCase() }}`
      - `questions_and_answers`: `={{ $json.payload.questions_and_answers || [] }}`
    - *Connections:* Input: `Wait for Calendar Sync`; Outputs: `Find Matching Calendar Event`, `Find CRM Contact`.
    - *Edge Cases:* Missing payload nodes resulting in empty strings or undefined array outputs.

---

#### 2.2 Calendar Synchronization
- **Overview:** Locates the corresponding Google Calendar event based on timestamps, filters matches via invitee email or Calendly event UUID, isolates the first result, and adds event attendees.
- **Nodes Involved:** `Find Matching Calendar Event`, `Match Calendar Event`, `Keep First Calendar Match`, `Extract Google Event ID`, `Add Calendar Attendees`.
- **Node Details:**
  - **Find Matching Calendar Event**
    - *Type & Role:* `n8n-nodes-base.googleCalendar` (API integration). Retrieves all calendar events within a specified window around the booking time.
    - *Configuration:* Operation `getAll`, returns all records.
    - *Key Expressions:* 
      - `timeMin`: `={{ DateTime.fromISO($('Extract Booking Data').first().json.start_time).minus({ minutes: 30 }).toISO() }}`
      - `timeMax`: `={{ DateTime.fromISO($('Extract Booking Data').first().json.end_time).plus({ minutes: 30 }).toISO() }}`
    - *Connections:* Input: `Extract Booking Data`; Output: `Match Calendar Event`.
    - *Edge Cases:* Incorrect calendar ID mapping or timezone evaluation errors.
  - **Match Calendar Event**
    - *Type & Role:* `n8n-nodes-base.filter` (Flow control). Filters retrieved events to find the exact match using email or UUID.
    - *Configuration:* Combinator `or`, checks attendee list and description fields.
    - *Key Expressions:* 
      - Left Value: `={{ ($json.attendees || []).map(a => String(a.email || '').toLowerCase()).join(',') }}`
      - Right Value: `={{ $('Extract Booking Data').first().json.invitee_email }}`
    - *Connections:* Input: `Find Matching Calendar Event`; Output: `Keep First Calendar Match`.
    - *Edge Cases:* Empty attendee arrays.
  - **Keep First Calendar Match**
    - *Type & Role:* `n8n-nodes-base.limit` (Flow control). Restricts output flow to the first matching record.
    - *Configuration:* Default limit parameters.
    - *Connections:* Input: `Match Calendar Event`; Output: `Extract Google Event ID`.
  - **Extract Google Event ID**
    - *Type & Role:* `n8n-nodes-base.set` (Data transformation). Extracts the internal Google Event ID and existing attendees.
    - *Key Expressions:* 
      - `google_event_id`: `={{ $json.id }}`
      - `existing_attendees`: `={{ ($json.attendees || []).map(a => a.email).filter(Boolean) }}`
    - *Connections:* Input: `Keep First Calendar Match`; Output: `Add Calendar Attendees`.
  - **Add Calendar Attendees**
    - *Type & Role:* `n8n-nodes-base.googleCalendar` (API integration). Updates the Google Calendar event.
    - *Configuration:* Operation `update`.
    - *Key Expressions:* `eventId`: `={{ $('Extract Google Event ID').first().json.google_event_id }}`
    - *Connections:* Input: `Extract Google Event ID`; Output: None (Terminal node for this branch).
    - *Edge Cases:* API rate limits or permission issues on the target calendar.

---

#### 2.3 CRM Processing
- **Overview:** Manages Airtable contacts and meetings, preventing duplicate database records and maintaining up-to-date tracking details.
- **Nodes Involved:** `Find CRM Contact`, `Contact Found?`, `Update Existing CRM Contact`, `Create CRM Contact`, `Prepare Meeting Data`, `Check for Duplicate Meeting`, `Meeting Already Logged?`, `Skip Duplicate Meeting`, `Create CRM Meeting`.
- **Node Details:**
  - **Find CRM Contact**
    - *Type & Role:* `n8n-nodes-base.airtable` (Database search). Searches contacts table by email address.
    - *Configuration:* Operation `search`, `alwaysOutputData: true`.
    - *Key Expressions:* `filterByFormula`: `={{ $json.invitee_email ? `LOWER({Email}) = LOWER('${$json.invitee_email.replaceAll(\"'\", \"\\\\'\")}')` : 'FALSE()' }}`
    - *Connections:* Input: `Extract Booking Data`; Output: `Contact Found?`.
  - **Contact Found?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow control). Branches based on whether an existing contact was retrieved.
    - *Key Expressions:* `={{ $json.id }}` (not empty).
    - *Connections:* Input: `Find CRM Contact`; Outputs: `Update Existing CRM Contact` (True), `Create CRM Contact` (False).
  - **Update Existing CRM Contact**
    - *Type & Role:* `n8n-nodes-base.airtable` (Database update). Updates contact properties with recent meeting metadata.
    - *Configuration:* Operation `update`, matching columns `id`.
    - *Key Expressions:* Maps Name, Email, Source, Last Meeting At, and Last Meeting Type.
    - *Connections:* Input: `Contact Found?` (True); Output: `Prepare Meeting Data`.
  - **Create CRM Contact**
    - *Type & Role:* `n8n-nodes-base.airtable` (Database creation). Creates a new contact entry if none exists.
    - *Configuration:* Operation `create`.
    - *Key Expressions:* Maps Name, Email, Source, Last Meeting At, and Last Meeting Type.
    - *Connections:* Input: `Contact Found?` (False); Output: `Prepare Meeting Data`.
  - **Prepare Meeting Data**
    - *Type & Role:* `n8n-nodes-base.set` (Data transformation). Consolidates contact ID and booking parameters for meeting logging.
    - *Connections:* Input: `Update Existing CRM Contact` / `Create CRM Contact`; Output: `Check for Duplicate Meeting`.
  - **Check for Duplicate Meeting**
    - *Type & Role:* `n8n-nodes-base.airtable` (Database search). Searches meetings table for existing Calendly Event IDs.
    - *Configuration:* Operation `search`, limit `1`.
    - *Key Expressions:* `filterByFormula`: `={{ `{Calendly Event ID} = '${$('Prepare Meeting Data').first().json.calendly_event_uuid.replaceAll(\"'\", \"\\\\'\")}'` }}`
    - *Connections:* Input: `Prepare Meeting Data`; Output: `Meeting Already Logged?`.
  - **Meeting Already Logged?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow control). Checks if a meeting record was found.
    - *Key Expressions:* `={{ $json.id }}` (not empty).
    - *Connections:* Input: `Check for Duplicate Meeting`; Outputs: `Skip Duplicate Meeting` (True), `Create CRM Meeting` (False).
  - **Skip Duplicate Meeting**
    - *Type & Role:* `n8n-nodes-base.noOp` (Flow control). No-operation placeholder for existing logs.
    - *Connections:* Input: `Meeting Already Logged?` (True); Output: None.
  - **Create CRM Meeting**
    - *Type & Role:* `n8n-nodes-base.airtable` (Database creation). Writes a new meeting record to Airtable.
    - *Configuration:* Operation `create`, typecast enabled.
    - *Key Expressions:* Maps Status, Start/End Time, Timezone, Event Type, Meeting URL, Invitee details, and Q&A strings.
    - *Connections:* Input: `Meeting Already Logged?` (False); Output: `Notify Team in Slack`.

---

#### 2.4 Team Notification
- **Overview:** Formats meeting variables, client questions, and integration links into an alert payload and sends it to Slack.
- **Nodes Involved:** `Notify Team in Slack`.
- **Node Details:**
  - **Notify Team In Slack**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging integration). Posts messages to a specified channel.
    - *Configuration:* Resource message, operation `post`, target selected channel ID.
    - *Key Expressions:* Constructs rich text markup using template expressions accessing `Prepare Meeting Data` properties, timestamps, and Q&A objects.
    - *Connections:* Input: `Create CRM Meeting`; Output: None.
    - *Edge Cases:* Invalid Slack token scopes or missing channel IDs preventing message delivery.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Workflow Overview | n8n-nodes-base.stickyNote | Documentation note | None | None | # Sync Calendly bookings to Airtable CRM and Slack... |
| Calendly Section | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Calendly booking... |
| Calendar Section | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Calendar sync... |
| CRM and Slack Section | n8n-nodes-base.stickyNote | Documentation note | None | None | ## CRM + Slack... |
| New Calendly Booking | n8n-nodes-base.calendlyTrigger | Trigger new booking event | None | Wait for Calendar Sync | ## Calendly booking... |
| Wait for Calendar Sync | n8n-nodes-base.wait | Delay execution | New Calendly Booking | Extract Booking Data | ## Calendly booking... |
| Extract Booking Data | n8n-nodes-base.set | Normalize booking variables | Wait for Calendar Sync | Find Matching Calendar Event, Find CRM Contact | ## Calendly booking... |
| Find Matching Calendar Event | n8n-nodes-base.googleCalendar | Query events in timeframe | Extract Booking Data | Match Calendar Event | ## Calendar sync... |
| Match Calendar Event | n8n-nodes-base.filter | Filter events by email/UUID | Find Matching Calendar Event | Keep First Calendar Match | ## Calendar sync... |
| Keep First Calendar Match | n8n-nodes-base.limit | Limit records to first match | Match Calendar Event | Extract Google Event ID | ## Calendar sync... |
| Extract Google Event ID | n8n-nodes-base.set | Extract event ID and attendees | Keep First Calendar Match | Add Calendar Attendees | ## Calendar sync... |
| Add Calendar Attendees | n8n-nodes-base.googleCalendar | Update event attendees | Extract Google Event ID | None | ## Calendar sync... |
| Find CRM Contact | n8n-nodes-base.airtable | Search contact by email | Extract Booking Data | Contact Found? | ## CRM + Slack... |
| Contact Found? | n8n-nodes-base.if | Check if contact exists | Find CRM Contact | Update Existing CRM Contact, Create CRM Contact | ## CRM + Slack... |
| Update Existing CRM Contact | n8n-nodes-base.airtable | Update existing CRM contact | Contact Found? | Prepare Meeting Data | ## CRM + Slack... |
| Create CRM Contact | n8n-nodes-base.airtable | Create new CRM contact | Contact Found? | Prepare Meeting Data | ## CRM + Slack... |
| Prepare Meeting Data | n8n-nodes-base.set | Bundle meeting properties | Update Existing CRM Contact, Create CRM Contact | Check for Duplicate Meeting | ## CRM + Slack... |
| Check for Duplicate Meeting | n8n-nodes-base.airtable | Query existing meeting logs | Prepare Meeting Data | Meeting Already Logged? | ## CRM + Slack... |
| Meeting Already Logged? | n8n-nodes-base.if | Prevent duplicate entries | Check for Duplicate Meeting | Skip Duplicate Meeting, Create CRM Meeting | ## CRM + Slack... |
| Skip Duplicate Meeting | n8n-nodes-base.noOp | Skip duplicate processing | Meeting Already Logged? | None | ## CRM + Slack... |
| Create CRM Meeting | n8n-nodes-base.airtable | Log meeting to Airtable | Meeting Already Logged? | Notify Team in Slack | ## CRM + Slack... |
| Notify Team in Slack | n8n-nodes-base.slack | Post notification message | Create CRM Meeting | None | ## CRM + Slack... |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to rebuild the workflow manually:

1. **Create the Trigger Node:** Add a `Calendly Trigger` node named `New Calendly Booking`. Set authentication to OAuth2 using your Calendly account credential, and listen for the `invitee.created` event.
2. **Add a Wait Node:** Create a `Wait` node named `Wait for Calendar Sync`. Set the amount to `1` and unit to `minutes`. Connect `New Calendly Booking` to this node.
3. **Extract Booking Data:** Add a `Set` node named `Extract Booking Data`. Configure string and array assignments matching the expressions for `calendly_event_uuid`, `invitee_name`, `invitee_email`, `event_name`, `start_time`, `end_time`, `invitee_timezone`, `join_url`, `booked_at`, and `questions_and_answers`. Connect `Wait for Calendar Sync` here.
4. **Setup Calendar Search:** Add a `Google Calendar` node named `Find Matching Calendar Event` (Operation: `getAll`). Use Luxon expressions (`DateTime.fromISO...`) to set `timeMin` and `timeMax` relative to the extracted start and end times. Connect `Extract Booking Data` to this node.
5. **Filter and Isolate Calendar Events:** 
   - Add a `Filter` node named `Match Calendar Event` with an OR combinator to evaluate invitee email and event UUID against attendees and descriptions. Connect `Find Matching Calendar Event` here.
   - Add a `Limit` node named `Keep First Calendar Match`. Connect `Match Calendar Event` here.
   - Add a `Set` node named `Extract Google Event ID` to output `google_event_id` and `existing_attendees`. Connect `Keep First Calendar Match` here.
   - Add a second `Google Calendar` node named `Add Calendar Attendees` (Operation: `update`). Set the Event ID expression to reference `Extract Google Event ID`. Connect `Extract Google Event ID` here.
6. **Setup CRM Search:** Add an `Airtable` node named `Find CRM Contact` (Operation: `search`). Enable `alwaysOutputData` and configure the formula filter to query contacts by email case-insensitively. Connect `Extract Booking Data` to this node.
7. **Branch Contact Creation/Update:**
   - Add an `If` node named `Contact Found?` evaluating whether `$json.id` is present. Connect `Find CRM Contact` here.
   - Add an `Airtable` node named `Update Existing CRM Contact` (Operation: `update`). Map fields (`Name`, `Email`, `Source`, `Last Meeting At`, `Last Meeting Type`) and set matching columns to `id`. Connect to the True branch of `Contact Found?`.
   - Add an `Airtable` node named `Create CRM Contact` (Operation: `create`). Map the same fields without matching columns. Connect to the False branch of `Contact Found?`.
8. **Prepare Meeting Payload:** Add a `Set` node named `Prepare Meeting Data`. Map all required meeting parameters (`contact_record_id`, attendee details, timestamps, URLs, Q&A array). Connect both `Update Existing CRM Contact` and `Create CRM Contact` outputs to this node.
9. **Check for Duplicate Meetings:**
   - Add an `Airtable` node named `Check for Duplicate Meeting` (Operation: `search`, limit `1`). Configure formula filtering using the `Calendly Event ID`. Connect `Prepare Meeting Data` here.
   - Add an `If` node named `Meeting Already Logged?` checking if `$json.id` exists. Connect `Check for Duplicate Meeting` here.
   - Add a `NoOp` node named `Skip Duplicate Meeting`. Connect to the True branch of `Meeting Already Logged?`.
10. **Log Meeting and Notify:**
    - Add an `Airtable` node named `Create CRM Meeting` (Operation: `create`, typecast enabled). Map fields for meeting status, start/end times, timezone, URL, event type, invitee info, and parsed Q&A text. Connect to the False branch of `Meeting Already Logged?`.
    - Add a `Slack` node named `Notify Team in Slack` (Operation: `post`). Configure the message template to use expressions pulling from `Prepare Meeting Data` and format Q&A items using `.map()`. Connect `Create CRM Meeting` here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Calendly Webhook Integration Guide | Ensure external webhooks are active or use native n8n polling/triggers via OAuth2. |
| Airtable Schema Requirements | Ensure Airtable base tables (`Contacts` and `Meetings`) contain the exact column names referenced in the node mappings. |