Book phone appointments with ElevenLabs, Exotel, and Cal.com

https://n8nworkflows.xyz/workflows/book-phone-appointments-with-elevenlabs--exotel--and-cal-com-19813


# Book phone appointments with ElevenLabs, Exotel, and Cal.com

### 1. Workflow Overview

This workflow exposes two primary webhook endpoints designed to integrate conversational voice agents (specifically ElevenLabs and Exotel) with Cal.com for end-to-end appointment scheduling. It handles slot availability queries, processes booking requests with attendee details, and manages auxiliary communication streaming URLs.

The workflow logic is categorized into the following functional blocks:

- **1.1 Input Reception & Routing:** Captures incoming HTTP POST requests from the voice assistant, evaluating payload metadata to determine the required action (`check_available_slot`, `book_appointment`, or an unknown operation).
- **1.2 Availability Check & Response:** Queries the Cal.com Slots API for available times within a specified date range, processes the JSON response, converts times to the `Asia/Kolkata` timezone, and returns up to four available slots.
- **1.3 Booking Validation & Execution:** Verifies booking requests, submits appointment creation payloads to the Cal.com Bookings API with attendee information, and routes the outcome to success or failure webhook responses.
- **1.4 Secondary Stream Webhook:** Operates an isolated webhook endpoint to return a direct WebSocket URL (`wss://`) routing Exotel audio streams to an ElevenLabs Conversational AI agent.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Routing
- **Overview:** Receives incoming appointment tool requests via webhook and routes them based on the requested operation type. It also includes an independent auxiliary endpoint for telephony stream configuration.
- **Nodes Involved:** 
  - `Appointment Booking Webhook`
  - `Check Available Slot Request`
  - `Check Appointment Booking Request`
  - `Handle Unknown Tool Response`
  - `Exotel Stream Webhook`
  - `Send Exotel Response`

- **Node Details:**
  - **Appointment Booking Webhook**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (v2) — Entry point for appointment assistant tool calls.
    - *Configuration:* Listens on path `appointment-webhook` for HTTP `POST` requests, holding responses for the response node.
    - *Expressions:* None (triggers on payload).
    - *Connections:* Input: None; Output: `Check Available Slot Request`.
    - *Edge Cases:* Missing or malformed JSON payloads will cause downstream conditional evaluations to evaluate as empty strings.

  - **Check Available Slot Request**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Evaluates whether the requested tool action is an availability check.
    - *Configuration:* Strict type validation, case-sensitive string matching.
    - *Key Expressions:* `={{ $json.body?.tool ? $json.body.tool.trim() : '' }}` equals `check_available_slot`.
    - *Connections:* Input: `Appointment Booking Webhook`; Output (True): `Fetch Available Slots`; Output (False): `Check Appointment Booking Request`.
    - *Edge Cases:* Untrimmed strings or unexpected casing are handled by the `.trim()` function, though entirely missing properties fall back safely to empty strings.

  - **Check Appointment Booking Request**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Secondary conditional gate verifying booking intent.
    - *Configuration:* Strict type validation, case-sensitive string matching.
    - *Key Expressions:* `={{ $json.body?.tool ? $json.body.tool.trim() : '' }}` equals `book_appointment`.
    - *Connections:* Input: `Check Available Slot Request` (False branch); Output (True): `Send Booking Request`; Output (False): `Handle Unknown Tool Response`.

  - **Handle Unknown Tool Response**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.2) — Fallback responder for unrecognized tool names.
    - *Configuration:* Responds with JSON payload.
    - *Key Expressions:* Returns `success: false` alongside the unrecognized tool name in the message body.
    - *Connections:* Input: `Check Appointment Booking Request` (False branch); Output: None (Terminal).

  - **Exotel Stream Webhook**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (v2.1) — Secondary entry point for Exotel streaming callbacks.
    - *Configuration:* Listens on path `exotel-stream` with response mode set to use a response node.
    - *Connections:* Input: None; Output: `Send Exotel Response`.

  - **Send Exotel Response**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.5) — Returns the WebSocket endpoint configuration for ElevenLabs-Exotel bridging.
    - *Configuration:* Responds with plain text (`Content-Type: text/plain`).
    - *Key Expressions:* Returns `wss://api.elevenlabs.io/v1/convai/conversation/exotel?agent_id=agent_3701kzdx6st5eszs2jzha6cq33f9`.
    - *Connections:* Input: `Exotel Stream Webhook`; Output: None (Terminal).

---

#### Block 1.2: Availability Check & Response
- **Overview:** Queries the Cal.com API for available time slots based on the requested date and constructs a formatted JSON response listing up to four available slots in local time.
- **Nodes Involved:**
  - `Fetch Available Slots`
  - `Send Slots Response`

- **Node Details:**
  - **Fetch Available Slots**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — External API call to Cal.com v2 slots endpoint.
    - *Configuration:* GET request to `https://api.cal.com/v2/slots`, authenticated via Header Auth and Cal API credentials. Sends query parameters and required API version headers.
    - *Key Expressions:* 
      - `startTime`: Evaluates incoming ISO start time, defaults to current date/time in `Asia/Kolkata`.
      - `endTime`: Sets search boundary to +1 day from start time.
      - Fixed query parameters: `eventTypeId: 6586355`, `timeZone: Asia/Kolkata`.
      - Header: `cal-api-version: 2024-08-13`.
    - *Connections:* Input: `Check Available Slot Request` (True branch); Output: `Send Slots Response`.
    - *Edge Cases:* API rate limits, invalid API keys, or incorrect event type IDs will return HTTP error responses.

  - **Send Slots Response**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.2) — Transforms raw slot objects into a chatbot-friendly JSON response.
    - *Configuration:* Responds with JSON.
    - *Key Expressions:* JavaScript IIFE parsing slot objects, extracting up to 4 indices, mapping ISO strings to formatted local time strings (`h:mm a`), and returning a availability boolean alongside a descriptive message.
    - *Connections:* Input: `Fetch Available Slots`; Output: None (Terminal).

---

#### Block 1.3: Booking Validation & Execution
- **Overview:** Submits appointment booking payloads containing attendee details to Cal.com and returns a confirmation or graceful error message depending on API success.
- **Nodes Involved:**
  - `Send Booking Request`
  - `Send Booking Success Response`
  - `Send Booking Error Response`

- **Node Details:**
  - **Send Booking Request**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — External API call to create bookings in Cal.com.
    - *Configuration:* POST request to `https://api.cal.com/v2/bookings`, authenticated via Header Auth. Configured with `onError: continueErrorOutput`.
    - *Key Expressions:* Dynamically builds JSON body converting input dates to UTC ISO format, passing attendee name, email, phone number (with automatic extraction and fallback parsing), and fixed `eventTypeId: 6586355`. Header: `cal-api-version: 2024-08-13`.
    - *Connections:* Input: `Check Appointment Booking Request` (True branch); Output 0 (Success): `Send Booking Success Response`; Output 1 (Error/Continue): `Send Booking Error Response`.
    - *Edge Cases:* Slot contention or past timestamps trigger the error output route instead of crashing execution.

  - **Send Booking Success Response**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.2) — Returns confirmation JSON containing the booking UID and start timestamp.
    - *Configuration:* Responds with JSON.
    - *Key Expressions:* Extracts booking UID and start properties from HTTP response data.
    - *Connections:* Input: `Send Booking Request` (Success branch); Output: None (Terminal).

  - **Send Booking Error Response**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.2) — Returns a user-friendly failure notice when slot booking fails.
    - *Configuration:* Responds with JSON.
    - *Key Expressions:* Static fallback JSON message indicating potential slot unavailability.
    - *Connections:* Input: `Send Booking Request` (Error branch); Output: None (Terminal).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation container for overall workflow description and setup instructions. | None | None | ## Appointment Booking<br><br>### How it works<br><br>This workflow exposes webhook endpoints... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation for the initial request receipt and routing block. | None | None | ## Receive and route request<br><br>The main appointment webhook receives incoming requests... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation covering the slot fetching branch. | None | None | ## Fetch available slots<br><br>This upper branch calls the Cal.com slots endpoint... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation for the booking validation routing cluster. | None | None | ## Validate booking request<br><br>This middle-lower routing cluster checks whether... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation for the booking creation HTTP request and response path. | None | None | ## Create booking response<br><br>This right-side booking cluster posts the appointment request... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation for the secondary Exotel webhook endpoint. | None | None | ## Secondary webhook reply<br><br>This separate lower-left cluster contains an independent webhook... |
| Appointment Booking Webhook | n8n-nodes-base.webhook | Entry point for incoming appointment tool requests. | None | Check Available Slot Request | ## Appointment Booking... |
| Check Available Slot Request | n8n-nodes-base.if | Evaluates if the tool request is for checking slot availability. | Appointment Booking Webhook | Fetch Available Slots, Check Appointment Booking Request | ## Receive and route request... |
| Exotel Stream Webhook | n8n-nodes-base.webhook | Entry point for Exotel audio streaming integration. | None | Send Exotel Response | ## Secondary webhook reply... |
| Send Exotel Response | n8n-nodes-base.respondToWebhook | Returns ElevenLabs Conversational AI WebSocket URL. | Exotel Stream Webhook | None | ## Secondary webhook reply... |
| Fetch Available Slots | n8n-nodes-base.httpRequest | Queries Cal.com API for available time slots. | Check Available Slot Request | Send Slots Response | ## Fetch available slots... |
| Send Slots Response | n8n-nodes-base.respondToWebhook | Formats available slots into a JSON response. | Fetch Available Slots | None | ## Fetch available slots... |
| Check Appointment Booking Request | n8n-nodes-base.if | Evaluates if the tool request is for booking an appointment. | Check Available Slot Request | Send Booking Request, Handle Unknown Tool Response | ## Validate booking request... |
| Handle Unknown Tool Response | n8n-nodes-base.respondToWebhook | Returns a fallback response for unrecognized tool actions. | Check Appointment Booking Request | None | ## Validate booking request... |
| Send Booking Request | n8n-nodes-base.httpRequest | Submits appointment creation payload to Cal.com API. | Check Appointment Booking Request | Send Booking Success Response, Send Booking Error Response | ## Create booking response... |
| Send Booking Success Response | n8n-nodes-base.respondToWebhook | Returns booking confirmation details upon success. | Send Booking Request | None | ## Create booking response... |
| Send Booking Error Response | n8n-nodes-base.respondToWebhook | Returns a failure message when booking creation fails. | Send Booking Request | Send Booking Success Response | ## Create booking response... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Nodes:**
   - Add a **Webhook** node named `Appointment Booking Webhook` (`path`: `appointment-webhook`, HTTP Method: `POST`, Response Mode: `Response Node`).
   - Add a second **Webhook** node named `Exotel Stream Webhook` (`path`: `exotel-stream`, Response Mode: `Response Node`).

2. **Set Up Availability Routing:**
   - Add an **If** node named `Check Available Slot Request`.
   - Set condition: Left Value `={{ $json.body?.tool ? $json.body.tool.trim() : '' }}`, Operator `Equals`, Right Value `check_available_slot`.
   - Connect `Appointment Booking Webhook` output to `Check Available Slot Request`.

3. **Configure Slot Retrieval & Response:**
   - Add an **HTTP Request** node named `Fetch Available Slots` (`Method`: `GET`, `URL`: `https://api.cal.com/v2/slots`).
   - Configure authentication using generic header auth (`cal-api-version`: `2024-08-13`).
   - Add Query Parameters:
     - `startTime`: `={{ $json.body?.startTime ? DateTime.fromISO($json.body.startTime).setZone('Asia/Kolkata').startOf('day').toISODate() : DateTime.now().setZone('Asia/Kolkata').toISODate() }}`
     - `endTime`: `={{ $json.body?.startTime ? DateTime.fromISO($json.body.startTime).setZone('Asia/Kolkata').plus({ days: 1 }).startOf('day').toISODate() : DateTime.now().setZone('Asia/Kolkata').plus({ days: 1 }).toISODate() }}`
     - `eventTypeId`: `6586355`
     - `timeZone`: `Asia/Kolkata`
   - Connect the True output of `Check Available Slot Request` to `Fetch Available Slots`.
   - Add a **Respond to Webhook** node named `Send Slots Response` (`Respond With`: `JSON`), using the slot mapping IIFE JavaScript expression provided in the reference configuration. Connect `Fetch Available Slots` to this node.

4. **Configure Booking Routing & Fallbacks:**
   - Add an **If** node named `Check Appointment Booking Request`.
   - Set condition: Left Value `={{ $json.body?.tool ? $json.body.tool.trim() : '' }}`, Operator `Equals`, Right Value `book_appointment`.
   - Connect the False output of `Check Available Slot Request` to `Check Appointment Booking Request`.
   - Add a **Respond to Webhook** node named `Handle Unknown Tool Response` (`Respond With`: `JSON`) returning the unrecognized tool payload. Connect the False output of `Check Appointment Booking Request` to this node.

5. **Configure Booking Submission & Responses:**
   - Add an **HTTP Request** node named `Send Booking Request` (`Method`: `POST`, `URL`: `https://api.cal.com/v2/bookings`).
   - Configure authentication with header auth (`cal-api-version`: `2024-08-13`), specifying a JSON body with dynamic start time, event type ID, and attendee contact properties. Set error handling to continue on error output.
   - Connect the True output of `Check Appointment Booking Request` to `Send Booking Request`.
   - Add a **Respond to Webhook** node named `Send Booking Success Response` (`Respond With`: `JSON`), connected to the success output of `Send Booking Request`.
   - Add a **Respond to Webhook** node named `Send Booking Error Response` (`Respond With`: `JSON`), connected to the error output of `Send Booking Request`.

6. **Configure Exotel Stream Handling:**
   - Add a **Respond to Webhook** node named `Send Exotel Response` (`Respond With`: `Text`, Response Header `Content-Type`: `text/plain`), supplying the ElevenLabs conversational WebSocket URL as the response body.
   - Connect `Exotel Stream Webhook` output to `Send Exotel Response`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Cal.com API v2 Integration Standards | Requires `cal-api-version` header set to `2024-08-13` for all requests. |
| Timezone Standardization | Operates primarily within the `Asia/Kolkata` timezone for availability slots and conversion rules. |
| ElevenLabs & Exotel Streaming Bridge | Exposes dedicated webhook callback endpoints for conversational voice agent integrations. |