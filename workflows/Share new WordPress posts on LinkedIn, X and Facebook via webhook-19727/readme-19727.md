Share new WordPress posts on LinkedIn, X and Facebook via webhook

https://n8nworkflows.xyz/workflows/share-new-wordpress-posts-on-linkedin--x-and-facebook-via-webhook-19727


# Share new WordPress posts on LinkedIn, X and Facebook via webhook

### 1. Workflow Overview

This workflow automates the distribution of newly published WordPress content to three major social media networks—LinkedIn, X (formerly Twitter), and Facebook. Triggered directly by a WordPress event, it eliminates the need for polling intervals, persistent state tables, or specialized multi-network WordPress plugins. 

The logic is segmented into the following functional blocks:

- **1.1 Input Reception & Configuration:** Receives the incoming webhook payload from WordPress containing the newly published post ID and merges it with foundational global configuration settings (such as the target site URL).
- **1.2 Post Fetching & Parsing:** Queries the public WordPress REST API to retrieve the complete post resource, then cleans and extracts core metadata (title, summary, permalink, and featured image URL).
- **1.3 Asset Retrieval & Branching Logic:** Downloads the featured image binary asset (if present) and evaluates its existence to determine whether LinkedIn should receive a native image upload or a standard article link card.
- **1.4 Social Media Publishing:** Dispatches formatted posts concurrently to LinkedIn, X, and a Facebook Page using isolated error-handling boundaries so that failures in one platform do not disrupt transmissions to the others.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Captures incoming HTTP POST requests from WordPress secured via header authentication, appending the base site URL configuration required for downstream API calls.
- **Nodes Involved:** `New Post Webhook`, `Settings`.
- **Node Details:**
  - **New Post Webhook**
    - Type: `n8n-nodes-base.webhook` (v2.1)
    - Technical Role: Acts as the primary webhook entry point listening for POST requests on path `wordpress-post-published`.
    - Configuration: Authenticated using Header Authentication (`X-Webhook-Token`).
    - Input/Output: Receives incoming HTTP requests; outputs JSON payload containing `body.post_id`.
    - Edge Cases: Unauthorized requests failing header validation will be rejected at the boundary; missing payloads or malformed JSON will halt execution.
  - **Settings**
    - Type: `n8n-nodes-base.set` (v3.4)
    - Technical Role: Injects global configuration variables into the execution context.
    - Configuration: Assigns string value `https://example.com` to variable `site_url` while preserving incoming fields.
    - Input/Output: Input connected from `New Post Webhook`; output connected to `Get Post`.

#### 2.2 Post Fetching & Parsing
- **Overview:** Queries the WordPress REST API using the dynamic post ID and sanitizes the returned HTML content into plain text fields suitable for social distribution.
- **Nodes Involved:** `Get Post`, `Post Fields`.
- **Node Details:**
  - **Get Post**
    - Type: `n8n-nodes-base.httpRequest` (v4.2)
    - Technical Role: Fetches full post details and embedded media from the WordPress REST API endpoint.
    - Configuration: GET method targeting `={{ $json.site_url.replace(/\/+$/, '') }}/wp-json/wp/v2/posts/{{ $json.body.post_id }}` with query parameter `_embed=wp:featuredmedia`.
    - Input/Output: Input from `Settings`; output to `Post Fields`.
    - Edge Cases: Network timeouts, invalid WordPress URLs, or non-existent post IDs will cause the HTTP request to fail.
  - **Post Fields**
    - Type: `n8n-nodes-base.code` (v2.0)
    - Technical Role: Normalizes and sanitizes WordPress REST API JSON payloads.
    - Configuration: JavaScript block that decodes HTML entities (`&#8217;`, `&amp;`, etc.), strips HTML tags from excerpts, standardizes ellipses, extracts the featured image URL, and generates a truncated character-safe title (`short_title`) capped at 240 characters for X.
    - Input/Output: Input from `Get Post`; outputs `title`, `summary`, `url`, `image_url`, and `short_title`.
    - Edge Cases: Unconventional REST schema structures or unexpected null fields in embedded media require optional chaining to avoid runtime exceptions.

#### 2.3 Asset Retrieval & Branching Logic
- **Overview:** Downloads the post’s featured image binary file and verifies its retrieval success to branch between image-based and link-card formats for LinkedIn.
- **Nodes Involved:** `Get Featured Image`, `Has Featured Image`.
- **Node Details:**
  - **Get Featured Image**
    - Type: `n8n-nodes-base.httpRequest` (v4.2)
    - Technical Role: Downloads the binary image file from the extracted `image_url`.
    - Configuration: GET method using `={{ $json.image_url }}`; response format set to file outputting to binary property `data`. Error handling is set to `continueRegularOutput`.
    - Input/Output: Input from `Post Fields`; output to `Has Featured Image`.
    - Edge Cases: Broken image links or slow remote servers can cause timeout or HTTP errors; handled gracefully via `continueRegularOutput`.
  - **Has Featured Image**
    - Type: `n8n-nodes-base.if` (v2.2)
    - Technical Role: Evaluates whether a valid binary image file was successfully downloaded.
    - Configuration: Evaluates condition `={{ !!$binary.data }}` (Boolean true).
    - Input/Output: Input from `Get Featured Image`; outputs true branch to `Publish on LinkedIn` and false branch to `Publish Link on LinkedIn`.

#### 2.4 Social Media Publishing
- **Overview:** Distributes the processed post payload to LinkedIn, X, and Facebook in parallel, isolating each platform with independent error-handling boundaries.
- **Nodes Involved:** `Publish on LinkedIn`, `Publish Link on LinkedIn`, `Publish on X`, `Publish on Facebook`.
- **Node Details:**
  - **Publish on LinkedIn**
    - Type: `n8n-nodes-base.linkedIn` (v1)
    - Technical Role: Publishes an image post containing text, summary, permalink, and the attached media binary.
    - Configuration: `postAs` set to `person`; `shareMediaCategory` set to `IMAGE`; binary property mapped to `data`. Error handling set to `continueRegularOutput`.
    - Credentials: LinkedIn OAuth2 API.
    - Input/Output: Input from `Has Featured Image` (True); no direct downstream outputs.
  - **Publish Link on LinkedIn**
    - Type: `n8n-nodes-base.linkedIn` (v1)
    - Technical Role: Publishes an article link card when no featured image is present.
    - Configuration: `postAs` set to `person`; `shareMediaCategory` set to `ARTICLE`; additional fields mapped for title, description, and `originalUrl`. Error handling set to `continueRegularOutput`.
    - Credentials: LinkedIn OAuth2 API.
    - Input/Output: Input from `Has Featured Image` (False); no direct downstream outputs.
  - **Publish on X**
    - Type: `n8n-nodes-base.twitter` (v2)
    - Technical Role: Posts status updates to X.
    - Configuration: Text parameter set to `={{ $json.short_title }}\n\n{{ $json.url }}`. Error handling set to `continueRegularOutput`.
    - Credentials: X OAuth2 API.
    - Input/Output: Input from `Post Fields`; no direct downstream outputs.
    - Edge Cases: Returns HTTP 402 if API developer account credits are exhausted.
  - **Publish on Facebook**
    - Type: `n8n-nodes-base.facebookGraphApi` (v1)
    - Technical Role: Publishes stories/feed posts to a Facebook Page via Graph API.
    - Configuration: Graph API version `v23.0`; HTTP POST to node `YOUR_PAGE_ID`, edge `feed`, passing query parameters `message` and `link`. Error handling set to `continueRegularOutput`.
    - Credentials: Facebook Graph API (requires Page access token).
    - Input/Output: Input from `Post Fields`; no direct downstream outputs.
    - Edge Cases: Fails with Forbidden error if a user token is supplied instead of a Page access token.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration guide | None | None | ## Share new WordPress posts to LinkedIn, X and Facebook<br><br>### How it works<br>WordPress sends the post ID here when a post is published. The workflow reads the post from your site's REST API and publishes it to LinkedIn, X and your Facebook Page. Nothing polls, so there is no schedule to tune and no list of already-posted IDs to keep.<br><br>### Setup steps<br>1. **Settings**: enter your site URL.<br>2. **New Post Webhook**: add a Header Auth credential and copy the production URL.<br>3. Get API access: [X developer console](https://console.x.com) (pay-per-use since February 2026: load credits under Billing first, a post with a link costs $0.20), [LinkedIn Posts API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api), [Meta access tokens](https://developers.facebook.com/docs/facebook-login/guides/access-tokens#portabletokens). Connect the credentials on the three publish nodes.<br>4. **Publish on Facebook**: replace YOUR_PAGE_ID with your Page ID.<br>5. In WordPress, POST `{"post_id": 123}` to the webhook URL on `publish_post`. The template description shows two ways: a webhook plugin, or a short PHP snippet for `functions.php`. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Input reception and data normalization overview | None | None | ## Receive and read<br>WordPress only sends the post ID. **Get Post** reads the title, permalink, excerpt and featured image from the site's public REST API, and **Post Fields** cleans them up. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Publishing architecture and error-resilience notes | None | None | ## Publish<br>Each node continues if another one fails, so a LinkedIn outage does not stop the X and Facebook posts. LinkedIn gets the featured image as an image post, or a link card when the post has none. Edit the wording per network here. |
| `New Post Webhook` | `n8n-nodes-base.webhook` | Webhook entry point for WordPress post events | None | `Settings` | ## Share new WordPress posts to LinkedIn, X and Facebook<br><br>### How it works<br>WordPress sends the post ID here when a post is published. The workflow reads the post from your site's REST API and publishes it to LinkedIn, X and your Facebook Page. Nothing polls, so there is no schedule to tune and no list of already-posted IDs to keep.<br><br>### Setup steps<br>1. **Settings**: enter your site URL.<br>2. **New Post Webhook**: add a Header Auth credential and copy the production URL.<br>3. Get API access: [X developer console](https://console.x.com) (pay-per-use since February 2026: load credits under Billing first, a post with a link costs $0.20), [LinkedIn Posts API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api), [Meta access tokens](https://developers.facebook.com/docs/facebook-login/guides/access-tokens#portabletokens). Connect the credentials on the three publish nodes.<br>4. **Publish on Facebook**: replace YOUR_PAGE_ID with your Page ID.<br>5. In WordPress, POST `{"post_id": 123}` to the webhook URL on `publish_post`. The template description shows two ways: a webhook plugin, or a short PHP snippet for `functions.php`. |
| `Settings` | `n8n-nodes-base.set` | Injects global site URL configuration | `New Post Webhook` | `Get Post` | ## Share new WordPress posts to LinkedIn, X and Facebook<br><br>### How it works<br>WordPress sends the post ID here when a post is published. The workflow reads the post from your site's REST API and publishes it to LinkedIn, X and your Facebook Page. Nothing polls, so there is no schedule to tune and no list of already-posted IDs to keep.<br><br>### Setup steps<br>1. **Settings**: enter your site URL.<br>2. **New Post Webhook**: add a Header Auth credential and copy the production URL.<br>3. Get API access: [X developer console](https://console.x.com) (pay-per-use since February 2026: load credits under Billing first, a post with a link costs $0.20), [LinkedIn Posts API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api), [Meta access tokens](https://developers.facebook.com/docs/facebook-login/guides/access-tokens#portabletokens). Connect the credentials on the three publish nodes.<br>4. **Publish on Facebook**: replace YOUR_PAGE_ID with your Page ID.<br>5. In WordPress, POST `{"post_id": 123}` to the webhook URL on `publish_post`. The template description shows two ways: a webhook plugin, or a short PHP snippet for `functions.php`. |
| `Get Post` | `n8n-nodes-base.httpRequest` | Fetches post data from WordPress REST API | `Settings` | `Post Fields` | ## Receive and read<br>WordPress only sends the post ID. **Get Post** reads the title, permalink, excerpt and featured image from the site's public REST API, and **Post Fields** cleans them up. |
| `Post Fields` | `n8n-nodes-base.code` | Parses, decodes, and sanitizes API response fields | `Get Post` | `Get Featured Image`, `Publish on X`, `Publish on Facebook` | ## Receive and read<br>WordPress only sends the post ID. **Get Post** reads the title, permalink, excerpt and featured image from the site's public REST API, and **Post Fields** cleans them up. |
| `Get Featured Image` | `n8n-nodes-base.httpRequest` | Downloads binary featured image asset | `Post Fields` | `Has Featured Image` | |
| `Has Featured Image` | `n8n-nodes-base.if` | Evaluates presence of downloaded image binary | `Get Featured Image` | `Publish on LinkedIn`, `Publish Link on LinkedIn` | |
| `Publish on LinkedIn` | `n8n-nodes-base.linkedIn` | Publishes post with media to LinkedIn | `Has Featured Image` (True) | None | ## Publish<br>Each node continues if another one fails, so a LinkedIn outage does not stop the X and Facebook posts. LinkedIn gets the featured image as an image post, or a link card when the post has none. Edit the wording per network here. |
| `Publish Link on LinkedIn` | `n8n-nodes-base.linkedIn` | Publishes article link card to LinkedIn | `Has Featured Image` (False) | None | ## Publish<br>Each node continues if another one fails, so a LinkedIn outage does not stop the X and Facebook posts. LinkedIn gets the featured image as an image post, or a link card when the post has none. Edit the wording per network here. |
| `Publish on X` | `n8n-nodes-base.twitter` | Publishes status update and link to X | `Post Fields` | None | ## Publish<br>Each node continues if another one fails, so a LinkedIn outage does not stop the X and Facebook posts. LinkedIn gets the featured image as an image post, or a link card when the post has none. Edit the wording per network here. |
| `Publish on Facebook` | `n8n-nodes-base.facebookGraphApi` | Publishes post to Facebook Page feed | `Post Fields` | None | ## Publish<br>Each node continues if another one fails, so a LinkedIn outage does not stop the X and Facebook posts. LinkedIn gets the featured image as an image post, or a link card when the post has none. Edit the wording per network here. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Entry Point:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`, v2.1).
   - Set Path to `wordpress-post-published`, HTTP Method to `POST`, and Authentication to `Header Auth`.
   - Configure a Header Auth credential (e.g., token name `X-Webhook-Token`).
2. **Add the Configuration Node:**
   - Add a **Set** node (`n8n-nodes-base.set`, v3.4).
   - Connect the Webhook output to this node.
   - Add an assignment: Name `site_url`, Type `string`, Value `https://example.com` (replace with your actual domain). Ensure "Include Other Fields" is enabled.
3. **Fetch Post Data from WordPress:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`, v4.2).
   - Connect the Settings node output to this node.
   - Set Method to `GET`, URL to `={{ $json.site_url.replace(/\/+$/, '') }}/wp-json/wp/v2/posts/{{ $json.body.post_id }}`.
   - Add a query parameter: Name `_embed`, Value `wp:featuredmedia`.
4. **Sanitize Post Fields (Code Node):**
   - Add a **Code** node (`n8n-nodes-base.code`, v2.0).
   - Connect the Get Post node output.
   - Insert JavaScript to decode HTML entities, strip HTML tags from the excerpt, extract permalinks, check for featured media sources, and create a truncated `short_title` (max 240 characters). Return an object containing `title`, `summary`, `url`, `image_url`, and `short_title`.
5. **Download Featured Image:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`, v4.2) named `Get Featured Image`.
   - Connect the Post Fields node output.
   - Set Method to `GET`, URL to `={{ $json.image_url }}`, and configure Response Format to `File` with output property name `data`.
   - Set Error Handling (`On Error`) to `Continue Regular Output`.
6. **Evaluate Image Presence:**
   - Add an **If** node (`n8n-nodes-base.if`, v2.2) named `Has Featured Image`.
   - Connect the Get Featured Image node output.
   - Set condition to evaluate whether `={{ !!$binary.data }}` is true.
7. **Publish to LinkedIn (Image Branch):**
   - Add a **LinkedIn** node (`n8n-nodes-base.linkedIn`, v1) named `Publish on LinkedIn`.
   - Connect the True output of `Has Featured Image`.
   - Configure credentials (`linkedInOAuth2Api`), set Post As to `Person`, Share Media Category to `IMAGE`, map binary property to `data`, and set additional field title to `={{ $('Post Fields').item.json.title }}`. Set text to include title, summary, and URL.
   - Set Error Handling to `Continue Regular Output`.
8. **Publish to LinkedIn (Link Branch):**
   - Add a **LinkedIn** node (`n8n-nodes-base.linkedIn`, v1) named `Publish Link on LinkedIn`.
   - Connect the False output of `Has Featured Image`.
   - Configure credentials (`linkedInOAuth2Api`), set Post As to `Person`, Share Media Category to `ARTICLE`, and map additional fields (`title`, `description`, `originalUrl`). Set text to include title and summary.
   - Set Error Handling to `Continue Regular Output`.
9. **Publish to X:**
   - Add a **Twitter / X** node (`n8n-nodes-base.twitter`, v2) named `Publish on X`.
   - Connect the Post Fields node output.
   - Configure credentials (`twitterOAuth2Api`), setting text to `={{ $json.short_title }}\n\n{{ $json.url }}`.
   - Set Error Handling to `Continue Regular Output`.
10. **Publish to Facebook:**
    - Add a **Facebook Graph API** node (`n8n-nodes-base.facebookGraphApi`, v1) named `Publish on Facebook`.
    - Connect the Post Fields node output.
    - Configure credentials (`facebookGraphApi` using a Page access token). Set Graph API Version to `v23.0`, HTTP Method to `POST`, Node to your Facebook Page ID (e.g., numeric ID replacing `YOUR_PAGE_ID`), Edge to `feed`, and query parameters for `message` and `link`.
    - Set Error Handling to `Continue Regular Output`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| X Developer Console | [X Developer Console](https://console.x.com) |
| LinkedIn Posts API Documentation | [LinkedIn Posts API Guide](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api) |
| Meta Access Token Documentation | [Facebook Access Tokens Guide](https://developers.facebook.com/docs/facebook-login/guides/access-tokens#portabletokens) |