Run an English website chatbot with OpenAI GPT-5 and Qdrant

https://n8nworkflows.xyz/workflows/run-an-english-website-chatbot-with-openai-gpt-5-and-qdrant-20208


# Run an English website chatbot with OpenAI GPT-5 and Qdrant

### 1. Workflow Overview

This workflow implements a secure, intent-driven English website chatbot powered by OpenAI and Qdrant. It receives incoming chat messages via a webhook, classifies their intent (rejection, smalltalk, or knowledge search), processes them accordingly, and returns a structured JSON response to the caller.

The system logic is organized into four main functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming HTTP POST requests, validates/authenticates them, and normalizes the payload into a standard chat format.
- **1.2 Intent Classification & Routing:** Analyzes the normalized message using OpenAI to determine the intent and routes the execution flow via a Switch node.
- **1.3 Handling Rejections & Smalltalk:** Manages out-of-scope or unsafe queries with a static safety message, and generates conversational replies for smalltalk queries using OpenAI.
- **1.4 Knowledge Retrieval & Answer Generation:** Queries a Qdrant vector database using OpenAI embeddings (optionally reranked via Cohere), compiles contextual snippets, and passes them to an OpenAI-powered agent constrained by a strict JSON schema to formulate structured search results.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Receives raw HTTP requests from external chat clients or websites, secures them via header authentication, and standardizes the payload structure for downstream processing.
- **Nodes Involved:** 
  - `When Chat Webhook Received`
  - `Normalize Webhook Payload`
- **Node Details:**
  - **When Chat Webhook Received**
    - *Type and technical role:* `n8n-nodes-base.webhook` (Webhook Trigger). Listens for incoming HTTP POST requests.
    - *Configuration:* Configured with webhook ID `e5edb88d-488e-4159-8315-d06676e667bf` and header authentication enabled.
    - *Input/Output:* No incoming connections; outputs to `Normalize Webhook Payload`.
    - *Edge cases:* Unauthorized requests (missing or invalid headers) will fail authentication; missing JSON body fields can cause downstream mapping issues if not handled.
  - **Normalize Webhook Payload**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Extracts and standardizes the message text and session identifier from the raw HTTP body.
    - *Configuration:* Executes custom JavaScript to parse input fields into a unified schema (`chatMessage`, `sessionId`).
    - *Input/Output:* Input from `When Chat Webhook Received`; output to `OpenAI Intent and Language Classifier`.
    - *Edge cases:* Malformed incoming JSON will throw JavaScript runtime errors.

#### 2.2 Intent Classification & Routing
- **Overview:** Evaluates the user's message using an LLM to categorize the intent and directs the workflow along the appropriate execution branch.
- **Nodes Involved:**
  - `OpenAI Intent and Language Classifier`
  - `Route by Intent and Language`
- **Node Details:**
  - **OpenAI Intent and Language Classifier**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.openAi` (Advanced AI Node). Communicates with OpenAI to classify the intent (e.g., smalltalk, search, reject).
    - *Configuration:* Uses OpenAI model settings optimized for fast classification and prompt adherence.
    - *Input/Output:* Input from `Normalize Webhook Payload`; output to `Route by Intent and Language`.
    - *Edge cases:* API rate limits, upstream timeout errors, or unexpected classification labels outside the expected enum.
  - **Route by Intent and Language**
    - *Type and technical role:* `n8n-nodes-base.switch` (Switch Node). Evaluates classification output and splits execution into distinct paths.
    - *Configuration:* Multi-output routing based on intent values (Path 1: Reject, Path 2: Smalltalk, Path 3: Search).
    - *Input/Output:* Input from `OpenAI Intent and Language Classifier`; outputs to `Format Safety Message`, `OpenAI Smalltalk Reply Generator`, and `Search Qdrant Knowledge Base`.
    - *Edge cases:* Unmatched intent strings fall through unless a default route is configured.

#### 2.3 Handling Rejections & Smalltalk
- **Overview:** Manages non-knowledge queries by returning a predefined safety message for rejected requests or generating conversational text responses for smalltalk.
- **Nodes Involved:**
  - `Format Safety Message`
  - `OpenAI Smalltalk Reply Generator`
  - `Format Smalltalk Output`
- **Node Details:**
  - **Format Safety Message**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Generates a standardized JSON payload containing the safety/rejection message.
    - *Configuration:* JavaScript execution returning a fixed structure.
    - *Input/Output:* Input from `Route by Intent and Language`; output to `Return Chatbot Webhook Response`.
  - **OpenAI Smalltalk Reply Generator**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.openAi` (Advanced AI Node). Generates a short, polite English conversational reply.
    - *Configuration:* Prompted for concise social interaction.
    - *Input/Output:* Input from `Route by Intent and Language`; output to `Format Smalltalk Output`.
  - **Format Smalltalk Output**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Structures the generated smalltalk response into the required JSON format.
    - *Configuration:* JavaScript execution wrapping the LLM text.
    - *Input/Output:* Input from `OpenAI Smalltalk Reply Generator`; output to `Return Chatbot Webhook Response`.

#### 2.4 Knowledge Retrieval & Answer Generation
- **Overview:** Searches the vector database for relevant website content, compiles context, and uses an AI agent with a strict output schema to generate structured answers with sources.
- **Nodes Involved:**
  - `Search Qdrant Knowledge Base`
  - `Generate Query Embeddings with OpenAI`
  - `Cohere Rerank by Relevance`
  - `Build Answer Context`
  - `Answer Generator Agent`
  - `OpenAI GPT 5`
  - `Define Answer JSON Schema`
  - `Parse Structured Answer`
  - `Clean Final Output`
  - `Return Chatbot Webhook Response`
- **Node Details:**
  - **Generate Query Embeddings with OpenAI**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.embeddingsOpenAi` (Embeddings Node). Converts user queries into vector embeddings.
    - *Configuration:* Linked to Qdrant search.
    - *Input/Output:* Connects as an embedding provider to `Search Qdrant Knowledge Base`.
  - **Cohere Rerank by Relevance**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.rerankerCohere` (Reranker Node). Refines search result ordering by semantic relevance.
    - *Configuration:* Linked to Qdrant search.
    - *Input/Output:* Connects as a reranker to `Search Qdrant Knowledge Base`.
  - **Search Qdrant Knowledge Base**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.vectorStoreQdrant` (Vector Store Node). Queries the Qdrant collection for matching documents.
    - *Configuration:* Configured with collection name (e.g., `website_knowledge`).
    - *Input/Output:* Input from `Route by Intent and Language`, embeddings, and reranker; output to `Build Answer Context`.
  - **Build Answer Context**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Aggregates retrieved documents into a concise context string containing titles, URLs, and text snippets.
    - *Configuration:* JavaScript array reduction and string interpolation.
    - *Input/Output:* Input from `Search Qdrant Knowledge Base`; output to `Answer Generator Agent`.
  - **OpenAI GPT 5**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Node). Provides the LLM backend for the answer generation agent.
    - *Input/Output:* Connects as a language model to `Answer Generator Agent`.
  - **Define Answer JSON Schema**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Output Parser Node). Enforces a strict JSON schema on the agent's output.
    - *Input/Output:* Connects as an output parser to `Answer Generator Agent`.
  - **Answer Generator Agent**
    - *Type and technical role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent Node). Synthesizes an answer based on context and enforces structured JSON formatting.
    - *Input/Output:* Inputs from `Build Answer Context`, `OpenAI GPT 5`, and `Define Answer JSON Schema`; output to `Parse Structured Answer`.
  - **Parse Structured Answer**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Parses the structured string output from the agent into a JavaScript object.
    - *Input/Output:* Input from `Answer Generator Agent`; output to `Clean Final Output`.
  - **Clean Final Output**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Final sanitization and formatting of the JSON response payload.
    - *Input/Output:* Input from `Parse Structured Answer`; output to `Return Chatbot Webhook Response`.
  - **Return Chatbot Webhook Response**
    - *Type and technical role:* `n8n-nodes-base.respondToWebhook` (Webhook Response Node). Sends the final HTTP response back to the webhook caller.
    - *Configuration:* Responds with JSON data from upstream nodes (`Clean Final Output`, `Format Smalltalk Output`, or `Format Safety Message`).
    - *Input/Output:* Inputs from `Clean Final Output`, `Format Smalltalk Output`, and `Format Safety Message`; no outgoing connections.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Chat Webhook Received | `n8n-nodes-base.webhook` | Trigger HTTP POST requests | None | Normalize Webhook Payload | [English-only template verified by Hermes](https://example.com) |
| Normalize Webhook Payload | `n8n-nodes-base.code` | Standardize incoming JSON | When Chat Webhook Received | OpenAI Intent and Language Classifier | [English-only template verified by Hermes](https://example.com) |
| OpenAI Intent and Language Classifier | `@n8n/n8n-nodes-langchain.openAi` | Classify user message intent | Normalize Webhook Payload | Route by Intent and Language | |
| Route by Intent and Language | `n8n-nodes-base.switch` | Route execution by intent | OpenAI Intent and Language Classifier | Format Safety Message, OpenAI Smalltalk Reply Generator, Search Qdrant Knowledge Base | |
| Format Safety Message | `n8n-nodes-base.code` | Generate rejection payload | Route by Intent and Language | Return Chatbot Webhook Response | |
| OpenAI Smalltalk Reply Generator | `@n8n/n8n-nodes-langchain.openAi` | Generate casual conversation | Route by Intent and Language | Format Smalltalk Output | |
| Format Smalltalk Output | `n8n-nodes-base.code` | Format smalltalk JSON | OpenAI Smalltalk Reply Generator | Return Chatbot Webhook Response | |
| Generate Query Embeddings with OpenAI | `@n8n/n8n-nodes-langchain.embeddingsOpenAi` | Create query vector embeddings | None | Search Qdrant Knowledge Base | |
| Cohere Rerank by Relevance | `@n8n/n8n-nodes-langchain.rerankerCohere` | Rerank search results | None | Search Qdrant Knowledge Base | |
| Search Qdrant Knowledge Base | `@n8n/n8n-nodes-langchain.vectorStoreQdrant` | Query vector store | Route by Intent and Language, Generate Query Embeddings with OpenAI, Cohere Rerank by Relevance | Build Answer Context | |
| Build Answer Context | `n8n-nodes-base.code` | Compile snippets into context | Search Qdrant Knowledge Base | Answer Generator Agent | |
| OpenAI GPT 5 | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM backend for agent | None | Answer Generator Agent | |
| Define Answer JSON Schema | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce JSON output structure | None | Answer Generator Agent | |
| Answer Generator Agent | `@n8n/n8n-nodes-langchain.agent` | Generate structured answer | Build Answer Context, OpenAI GPT 5, Define Answer JSON Schema | Parse Structured Answer | |
| Parse Structured Answer | `n8n-nodes-base.code` | Parse agent output string | Answer Generator Agent | Clean Final Output | |
| Clean Final Output | `n8n-nodes-base.code` | Finalize response payload | Parse Structured Answer | Return Chatbot Webhook Response | |
| Return Chatbot Webhook Response | `n8n-nodes-base.respondToWebhook` | Send HTTP response | Format Safety Message, Format Smalltalk Output, Clean Final Output | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Webhook** node named `When Chat Webhook Received`.
   - Set HTTP Method to `POST`, path to your preferred webhook endpoint, and enable Header Authentication.

2. **Add Payload Normalization:**
   - Add a **Code** node named `Normalize Webhook Payload`.
   - Connect `When Chat Webhook Received` output to this node.
   - Configure JavaScript to extract `chatMessage` and `sessionId` from `$json.body`.

3. **Configure Intent Classification:**
   - Add an **OpenAI** node named `OpenAI Intent and Language Classifier`.
   - Configure credentials and prompt it to categorize the message into `reject`, `smalltalk`, or `search`.
   - Connect `Normalize Webhook Payload` to this node.

4. **Set Up Intent Routing:**
   - Add a **Switch** node named `Route by Intent and Language`.
   - Connect the classifier output to this node.
   - Define three output rules corresponding to `reject`, `smalltalk`, and `search`.

5. **Build the Rejection Branch:**
   - Add a **Code** node named `Format Safety Message`.
   - Connect Switch output index 0 (reject) to this node.
   - Return a static JSON response indicating the message is out of scope.

6. **Build the Smalltalk Branch:**
   - Add an **OpenAI** node named `OpenAI Smalltalk Reply Generator` and connect Switch output index 1 (smalltalk) to it.
   - Add a **Code** node named `Format Smalltalk Output` to wrap the generated text into a JSON structure, and connect the generator to it.

7. **Build the Knowledge Retrieval Branch:**
   - Add **Embeddings OpenAI** (`Generate Query Embeddings with OpenAI`) and **Reranker Cohere** (`Cohere Rerank by Relevance`) nodes, configuring their respective credentials.
   - Add a **Vector Store Qdrant** node named `Search Qdrant Knowledge Base`. Connect the embeddings and reranker nodes to their respective sub-node inputs, and connect Switch output index 2 (search) to the main input. Configure the Qdrant credential and collection name (`website_knowledge`).
   - Add a **Code** node named `Build Answer Context`. Connect Qdrant search output to this node to format retrieved documents into a context string.

8. **Build the AI Answer Generation Sub-Graph:**
   - Add an **OpenAI Chat Model** node (`OpenAI GPT 5`) and configure model parameters.
   - Add a **Structured Output Parser** node (`Define Answer JSON Schema`) specifying up to five sources and an optional follow-up question.
   - Add an **AI Agent** node named `Answer Generator Agent`. Connect `Build Answer Context` to its main input, and connect `OpenAI GPT 5` and `Define Answer JSON Schema` to their respective AI sub-node inputs.
   - Add a **Code** node named `Parse Structured Answer` and connect the agent output to it.
   - Add a **Code** node named `Clean Final Output` to sanitize the parsed object.

9. **Finalize Webhook Response:**
   - Add a **Respond to Webhook** node named `Return Chatbot Webhook Response`.
   - Connect the outputs from `Format Safety Message`, `Format Smalltalk Output`, and `Clean Final Output` into this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| English-only template verified by Hermes | [English-only template verified by Hermes](https://example.com) |