# X API — Advanced Patterns

TypeScript integration, filtered stream (real-time monitoring), OAuth 2.0 PKCE flow, and webhook setup.

---

## TypeScript / JavaScript

### Installation

```bash
npm install twitter-api-v2
```

### Client Setup

```typescript
import { TwitterApi } from "twitter-api-v2";

// App-only client (read operations)
const readClient = new TwitterApi(process.env.X_BEARER_TOKEN!);
const roClient = readClient.readOnly;

// User context client (read + write)
const rwClient = new TwitterApi({
  appKey: process.env.X_API_KEY!,
  appSecret: process.env.X_API_SECRET!,
  accessToken: process.env.X_ACCESS_TOKEN!,
  accessSecret: process.env.X_ACCESS_SECRET!,
});
```

### Post a Tweet

```typescript
const tweet = await rwClient.v2.tweet("Hello from Claude");
console.log(tweet.data.id);
```

### Post a Thread

```typescript
async function postThread(client: TwitterApi, texts: string[]): Promise<string[]> {
  const ids: string[] = [];
  let replyToId: string | undefined;

  for (const text of texts) {
    const payload: Parameters<typeof client.v2.tweet>[0] = { text };
    if (replyToId) {
      payload.reply = { in_reply_to_tweet_id: replyToId };
    }

    const response = await client.v2.tweet(payload);
    const id = response.data.id;
    ids.push(id);
    replyToId = id;
  }

  return ids;
}
```

### Search with Pagination

```typescript
// Paginate automatically — collect up to 500 tweets
const paginator = await roClient.v2.search("claude code -is:retweet", {
  max_results: 100,
  "tweet.fields": ["created_at", "public_metrics"],
});

const tweets = await paginator.fetchLast(500);
for (const tweet of tweets) {
  console.log(tweet.text, tweet.public_metrics);
}
```

### Read User Timeline

```typescript
const user = await roClient.v2.userByUsername("example_user");
const userId = user.data.id;

const timeline = await roClient.v2.userTimeline(userId, {
  max_results: 10,
  "tweet.fields": ["created_at", "public_metrics"],
  exclude: ["retweets", "replies"],
});

for await (const tweet of timeline) {
  console.log(tweet.text);
}
```

### Error Handling (TypeScript)

```typescript
import { ApiResponseError, ApiRequestError } from "twitter-api-v2";

async function safeTweet(client: TwitterApi, text: string): Promise<string | null> {
  try {
    const response = await client.v2.tweet(text);
    return response.data.id;
  } catch (error) {
    if (error instanceof ApiResponseError) {
      if (error.rateLimitError && error.rateLimit) {
        const resetMs = error.rateLimit.reset * 1000 - Date.now();
        console.error(`Rate limited. Retry after ${Math.ceil(resetMs / 1000)}s`);
      } else {
        console.error(`API error ${error.code}: ${error.message}`);
      }
    } else if (error instanceof ApiRequestError) {
      console.error(`Network error: ${error.requestError.message}`);
    }
    return null;
  }
}
```

---

## OAuth 2.0 PKCE Flow (New Apps)

OAuth 2.0 with PKCE is the recommended approach for new applications requiring user-delegated write access. It uses short-lived access tokens with refresh tokens, which is more secure than OAuth 1.0a's long-lived access tokens.

### Python (tweepy)

```python
import tweepy

# Step 1: Create the OAuth2 handler
oauth2_handler = tweepy.OAuth2UserHandler(
    client_id=os.environ["X_OAUTH2_CLIENT_ID"],
    redirect_uri="https://your-app.com/callback",
    scope=["tweet.read", "tweet.write", "users.read", "offline.access"],
    client_secret=os.environ["X_OAUTH2_CLIENT_SECRET"],  # for confidential clients
)

# Step 2: Generate authorization URL — redirect user here
auth_url = oauth2_handler.get_authorization_url()
print(f"Authorize at: {auth_url}")

# Step 3: After user authorizes, exchange the callback URL for tokens
# (callback_url is the full URL the user was redirected to after authorization)
access_token = oauth2_handler.fetch_token(callback_url)

# Step 4: Create an authenticated client
client = tweepy.Client(access_token=access_token["access_token"])
response = client.create_tweet(text="Authorized via OAuth 2.0 PKCE")
```

### TypeScript (twitter-api-v2)

```typescript
import { TwitterApi } from "twitter-api-v2";

const client = new TwitterApi({
  clientId: process.env.X_OAUTH2_CLIENT_ID!,
  clientSecret: process.env.X_OAUTH2_CLIENT_SECRET,
});

// Step 1: Generate auth link
const { url, codeVerifier, state } = client.generateOAuth2AuthLink(
  "https://your-app.com/callback",
  { scope: ["tweet.read", "tweet.write", "users.read", "offline.access"] },
);
// Store codeVerifier and state in session — needed for step 2

// Step 2: After callback — exchange code for tokens
const { client: loggedInClient, accessToken, refreshToken } =
  await client.loginWithOAuth2({
    code: callbackParams.code,
    codeVerifier,           // from session
    redirectUri: "https://your-app.com/callback",
  });

// Step 3: Use the authenticated client
await loggedInClient.v2.tweet("Authorized via OAuth 2.0 PKCE");

// Step 4: Refresh when access token expires (if offline.access scope included)
const { client: refreshedClient, accessToken: newToken } =
  await client.refreshOAuth2Token(refreshToken);
```

---

## Filtered Stream (Real-Time Monitoring)

Use filtered stream instead of polling for real-time mention and keyword monitoring. Significantly more efficient.

### Python (tweepy)

```python
class MentionListener(tweepy.StreamingClient):
    def on_tweet(self, tweet: tweepy.Tweet) -> None:
        print(f"New tweet: {tweet.text}")
        # Add your processing logic here

    def on_errors(self, errors: list) -> None:
        print(f"Stream errors: {errors}")

    def on_connection_error(self) -> bool:
        print("Connection error — reconnecting...")
        return True   # return True to reconnect automatically

# Set up the stream
stream = MentionListener(os.environ["X_BEARER_TOKEN"])

# Add filter rules (persistent — survives stream restarts)
# First, clear any existing rules
existing = stream.get_rules()
if existing.data:
    stream.delete_rules([rule.id for rule in existing.data])

# Add new rules
stream.add_rules([
    tweepy.StreamRule("@example_user -is:retweet", tag="mentions"),
    tweepy.StreamRule("from:competitor_brand", tag="competitor"),
])

# Start streaming — runs until manually stopped or error
stream.filter(
    tweet_fields=["created_at", "author_id", "public_metrics"],
    expansions=["author_id"],
)
```

### Raw requests (manual stream)

```python
import json

def stream_filtered(rules: list[str]) -> None:
    with requests.get(
        "https://api.x.com/2/tweets/search/stream",
        headers=HEADERS,
        stream=True,
        timeout=90,
    ) as resp:
        resp.raise_for_status()
        for line in resp.iter_lines():
            if line:
                data = json.loads(line)
                print(data["data"]["text"])
```

---

## Engagement Analytics

Track performance of posted content:

```python
def get_tweet_metrics(tweet_id: str) -> dict:
    resp = requests.get(
        f"https://api.x.com/2/tweets/{tweet_id}",
        headers=HEADERS,
        params={"tweet.fields": "public_metrics,non_public_metrics,organic_metrics"},
    )
    resp.raise_for_status()
    return resp.json()["data"]["public_metrics"]

# public_metrics (available without elevated access):
# retweet_count, reply_count, like_count, quote_count, impression_count

# non_public_metrics and organic_metrics require OAuth user context
# and are only available for tweets posted by the authenticated account
```

### Bulk Analytics for a Thread

```python
def thread_analytics(tweet_ids: list[str]) -> list[dict]:
    """Fetch metrics for up to 100 tweets in one request."""
    resp = requests.get(
        "https://api.x.com/2/tweets",
        headers=HEADERS,
        params={
            "ids": ",".join(tweet_ids),
            "tweet.fields": "public_metrics",
        },
    )
    resp.raise_for_status()
    return resp.json().get("data", [])
```

---

## Environment Setup Summary

```bash
# Required for all operations
export X_BEARER_TOKEN="..."          # app-only read access

# Required for write operations (OAuth 1.0a)
export X_API_KEY="..."
export X_API_SECRET="..."
export X_ACCESS_TOKEN="..."
export X_ACCESS_SECRET="..."

# Required for OAuth 2.0 PKCE flow
export X_OAUTH2_CLIENT_ID="..."
export X_OAUTH2_CLIENT_SECRET="..."  # only for confidential clients
```

Never commit these to source control. Add `.env` to `.gitignore` before the first commit.
