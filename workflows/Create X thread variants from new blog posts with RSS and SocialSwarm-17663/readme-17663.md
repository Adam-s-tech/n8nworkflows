Create X thread variants from new blog posts with RSS and SocialSwarm

https://n8nworkflows.xyz/workflows/create-x-thread-variants-from-new-blog-posts-with-rss-and-socialswarm-17663


# Create X thread variants from new blog posts with RSS and SocialSwarm

### 1. Workflow Overview

This workflow automates the content repurposing pipeline by monitoring an RSS/Atom blog feed for new publications and transforming each article into multiple X/Twitter thread variants using SocialSwarm. The logical architecture consists of a single processing stream divided into three functional blocks:

- **1.1 Input Reception:** Polls a specified RSS/Atom feed to capture new blog posts and trigger the execution pipeline.
- **1.2 AI Processing:** Sends the extracted post URL to the SocialSwarm API to generate multiple structured X/Twitter thread variants based on distinct hooks (such as curiosity, challenge, or story).
- **1.3 Data Formatting:** Flattens the array of generated thread variants into individual items, outputting each thread separately for downstream routing, review, or scheduling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** This block monitors an RSS feed at scheduled intervals and initiates the workflow execution whenever a new item is detected.
- **Nodes Involved:** `On new blog post`
- **Node Details:**
  - **Node Name:** On new blog post
  - **Type & Technical Role:** `n8n-nodes-base.rssFeedReadTrigger` — Acts as the workflow event trigger by polling an RSS feed URL.
  - **Configuration Choices:** Configured to monitor the default n8n blog RSS feed URL (`https://blog.n8n.io/rss/`).
  - **Key Expressions or Variables:** None (uses native trigger output payload).
  - **Input and Output Connections:** 
    - Input: None (Trigger node).
    - Output: Connects to `Generate thread variants`.
  - **Version-specific Requirements:** Version 1.
  - **Edge Cases & Potential Failures:** Feed unavailability, invalid XML formatting in the feed, or rate-limiting by the hosting server. A manual execution processes all recent items unless restricted by a downstream Limit node.

#### 2.2 AI Processing
- **Overview:** This block takes the link of the newly published blog post and calls the SocialSwarm API to analyze the content and generate structured, multi-tweet thread variants.
- **Nodes Involved:** `Generate thread variants`
- **Node Details:**
  - **Node Name:** Generate thread variants
  - **Type & Technical Role:** `n8n-nodes-socialswarm.socialSwarm` — Communicates with the SocialSwarm service to generate social media threads.
  - **Configuration Choices:** 
    - Operation: `generate`
    - Source Type: `url`
    - Variant Count: `3` (can be increased to 5 for additional angles).
  - **Key expressions or variables:** `={{ $json.link }}` extracts the article URL from the preceding RSS trigger node.
  - **Input and Output Connections:**
    - Input: Receives main execution data from `On new blog post`.
    - Output: Connects to `One item per thread`.
  - **Version-specific Requirements:** Version 1. Requires a valid SocialSwarm API credential configured in n8n.
  - **Edge Cases & Potential Failures:** API authentication errors, invalid API keys, timeout errors during content scraping, or incorrect URL formats.

#### 2.3 Data Formatting
- **Overview:** This block takes the aggregated response array from SocialSwarm and splits it so that each thread variant is handled as an independent item.
- **Nodes Involved:** `One item per thread`
- **Node Details:**
  - **Node Name:** One item per thread
  - **Type & Technical Role:** `n8n-nodes-base.splitOut` — Splits items containing an array into individual items per array element.
  - **Configuration Choices:** Field to split out is set to `variants`.
  - **Key expressions or variables:** Operates on the incoming JSON property `variants`.
  - **Input and Output Connections:**
    - Input: Receives main execution data from `Generate thread variants`.
    - Output: Terminal output of the workflow (ready for downstream routing nodes like Slack, Google Sheets, or Buffer).
  - **Version-specific Requirements:** Version 1.
  - **Edge Cases & Potential Failures:** Fails if the expected `variants` array is missing or malformed due to an unexpected API response.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On new blog post | `n8n-nodes-base.rssFeedReadTrigger` | Polls RSS/Atom feed for new entries | None | Generate thread variants | Turn every new blog post into X/Twitter thread variants<br><br>Watches an RSS feed and turns each new post into ready-to-post X/Twitter threads — three complete variants, each built around a different hook (curiosity, challenge, story), so you can pick the angle that fits.<br><br>### How it works<br><br>1. **On new blog post** polls any RSS/Atom feed and fires once per new item.<br>2. **Generate thread variants** sends the post URL to SocialSwarm, which fetches the article and writes three full threads from it. Every tweet arrives with its character count already calculated, so nothing needs re-checking before it goes out.<br>3. **One item per thread** splits the result so each thread becomes its own item — ready to loop over, filter, or send onward.<br><br>### Setup<br><br>1. Create a SocialSwarm API key on the [developer portal](https://social-swarm-main-aa77a19.zuplo.site). There is a free plan and it does not ask for a card.<br>2. On **Generate thread variants**, add a **SocialSwarm API** credential and paste the key.<br>3. Replace the feed URL on **On new blog post** with your own.<br>4. Execute once to check the output, then activate.<br><br>### Customization tips<br><br>- Switch **Number of Variants** to 5 for more angles per post.<br>- Set **Source Type** to `Text` to repurpose newsletters or release notes instead of a feed.<br>- Add your own node after **One item per thread** to send drafts to Slack, Google Sheets, Buffer, or X.<br>- A manual run processes every recent feed item — add a **Limit** node after the trigger to handle only the newest. |
| Generate thread variants | `n8n-nodes-socialswarm.socialSwarm` | Generates X thread variants using SocialSwarm API | On new blog post | One item per thread | Turn every new blog post into X/Twitter thread variants<br><br>Watches an RSS feed and turns each new post into ready-to-post X/Twitter threads — three complete variants, each built around a different hook (curiosity, challenge, story), so you can pick the angle that fits.<br><br>### How it works<br><br>1. **On new blog post** polls any RSS/Atom feed and fires once per new item.<br>2. **Generate thread variants** sends the post URL to SocialSwarm, which fetches the article and writes three full threads from it. Every tweet arrives with its character count already calculated, so nothing needs re-checking before it goes out.<br>3. **One item per thread** splits the result so each thread becomes its own item — ready to loop over, filter, or send onward.<br><br>### Setup<br><br>1. Create a SocialSwarm API key on the [developer portal](https://social-swarm-main-aa77a19.zuplo.site). There is a free plan and it does not ask for a card.<br>2. On **Generate thread variants**, add a **SocialSwarm API** credential and paste the key.<br>3. Replace the feed URL on **On new blog post** with your own.<br>4. Execute once to check the output, then activate.<br><br>### Customization tips<br><br>- Switch **Number of Variants** to 5 for more angles per post.<br>- Set **Source Type** to `Text` to repurpose newsletters or release notes instead of a feed.<br>- Add your own node after **One item per thread** to send drafts to Slack, Google Sheets, Buffer, or X.<br>- A manual run processes every recent feed item — add a **Limit** node after the trigger to handle only the newest.<br><br>RSS post → thread variants<br><br>Each new feed item goes to SocialSwarm as a URL, then splits into one item per thread. Add your scheduler or inbox after the last node. |
| One item per thread | `n8n-nodes-base.splitOut` | Splits variant arrays into individual items | Generate thread variants | None | Turn every new blog post into X/Twitter thread variants<br><br>Watches an RSS feed and turns each new post into ready-to-post X/Twitter threads — three complete variants, each built around a different hook (curiosity, challenge, story), so you can pick the angle that fits.<br><br>### How it works<br><br>1. **On new blog post** polls any RSS/Atom feed and fires once per new item.<br>2. **Generate thread variants** sends the post URL to SocialSwarm, which fetches the article and writes three full threads from it. Every tweet arrives with its character count already calculated, so nothing needs re-checking before it goes out.<br>3. **One item per thread** splits the result so each thread becomes its own item — ready to loop over, filter, or send onward.<br><br>### Setup<br><br>1. Create a SocialSwarm API key on the [developer portal](https://social-swarm-main-aa77a19.zuplo.site). There is a free plan and it does not ask for a card.<br>2. On **Generate thread variants**, add a **SocialSwarm API** credential and paste the key.<br>3. Replace the feed URL on **On new blog post** with your own.<br>4. Execute once to check the output, then activate.<br><br>### Customization tips<br><br>- Switch **Number of Variants** to 5 for more angles per post.<br>- Set **Source Type** to `Text` to repurpose newsletters or release notes instead of a feed.<br>- Add your own node after **One item per thread** to send drafts to Slack, Google Sheets, Buffer, or X.<br>- A manual run processes every recent feed item — add a **Limit** node after the trigger to handle only the newest.<br><br>RSS post → thread variants<br><br>Each new feed item goes to SocialSwarm as a URL, then splits into one item per thread. Add your scheduler or inbox after the last node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:** Add an **RSS Feed Read Trigger** node (`n8n-nodes-base.rssFeedReadTrigger`) and rename it to `On new blog post`. Set the **Feed URL** parameter to `https://blog.n8n.io/rss/` (or your target blog feed).
2. **Create the AI Processing Node:** Add a **SocialSwarm** node (`n8n-nodes-socialswarm.socialSwarm`) and rename it to `Generate thread variants`. 
   - Set **Operation** to `generate`.
   - Set **Source Type** to `url`.
   - Set **Source URL** expression to `={{ $json.link }}`.
   - Set **Variant Count** to `3`.
   - Configure a **SocialSwarm API** credential using an API key generated from the SocialSwarm developer portal.
3. **Create the Data Formatting Node:** Add a **Split Out** node (`n8n-nodes-base.splitOut`) and rename it to `One item per thread`. Set **Field to Split Out** to `variants`.
4. **Establish Connections:**
   - Connect the output of `On new blog post` (Main) to the input of `Generate thread variants` (Main).
   - Connect the output of `Generate thread variants` (Main) to the input of `One item per thread` (Main).
5. **Finalize and Test:** Execute the workflow manually to verify data output shape (`variants[]` containing `hook`, `tweetCount`, and `tweets[]`), then activate the trigger.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SocialSwarm Developer Portal | [https://social-swarm-main-aa77a19.zuplo.site](https://social-swarm-main-aa77a19.zuplo.site) |