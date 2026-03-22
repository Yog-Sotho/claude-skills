---
name: x-api
description: X/Twitter API integration patterns for posting tweets and threads, reading timelines, searching, media upload, pagination, and real-time streaming. Covers OAuth 2.0 and 1.0a authentication, rate limit handling, and content policy compliance. Always activate when the user wants to interact with X programmatically, says "post to X", "tweet this", "X API", "Twitter API", or is building an X bot, scheduler, analytics tool, or content automation workflow.
---

# X API

Programmatic interaction with X (Twitter) for posting, reading, searching, and monitoring.

## Workflow

When this skill activates:

1. **Identify the operation** — posting/writing, reading/searching, or real-time monitoring.
2. **Choose the right auth method** from the decision table below — this is the most common source of confusion.
3. **Choose the right client** — `tweepy` for most Python use cases; raw `requests` for advanced control; `twitter-api-v2` for TypeScript.
4. **Handle pagination** — any production search or timeline query will need `next_token` iteration.
5. **Read rate limit headers at runtime** — never hardcode rate limit assumptions; they change by tier and endpoint.
6. **Verify content policy compliance** before shipping automated posting — account suspensions are hard to reverse.

For TypeScript patterns and filtered stream (real-time), see `references/advanced.md`.

---

## Authentication

### Decision Table

| Operation | Auth Method |
|-----------|-------------|
| Read public data, search, timelines | OAuth 2.0 Bearer Token (app-only) |
| Post tweets, manage account, DMs | OAuth 2.0 PKCE *or* OAuth 1.0a (user context) |
| Server-to-server on behalf of a user | OAuth 1.0a (established apps) or OAuth 2.0 PKCE (new apps) |

**Why you can't post with the bearer token:** Bearer tokens authenticate your *app*, not a *user*. Write operations require user context — either OAuth 1.0a or OAuth 2.0 with user-delegated access (PKCE).

### OAuth 2.0 Bearer Token (App-Only — Read Operations)

```bash
export X_BEARER_TOKEN="your-bearer-token"
```

```python
import os, requests

HEADERS = {"Authorization": f"Bearer {os.environ['X_BEARER_TOKEN']}"}

def get(url: str, params: dict) -> dict:
    resp = requests.get(url, headers=HEADERS, params=params)
    resp.raise_for_status()   # raises HTTPError on 4xx/5xx
    return resp.json()
```

### OAuth 1.0a (User Context — Write Operations)

```bash
export X_API_KEY="your-api-key"
export X_API_SECRET="your-api-secret"
export X_ACCESS_TOKEN="your-access-token"
export X_ACCESS_SECRET="your-access-secret"
```

```python
from requests_oauthlib import OAuth1Session

def make_oauth() -> OAuth1Session:
    return OAuth1Session(
        os.environ["X_API_KEY"],
        client_secret=os.environ["X_API_SECRET"],
        resource_owner_key=os.environ["X_ACCESS_TOKEN"],
        resource_owner_secret=os.environ["X_ACCESS_SECRET"],
    )
```

### tweepy (Recommended Python Client)

`tweepy` wraps both auth flows, handles rate limit backoff automatically, and provides typed response objects:

```bash
pip install tweepy
```

```python
import tweepy

# App-only (read)
read_client = tweepy.Client(bearer_token=os.environ["X_BEARER_TOKEN"])

# User context (read + write)
rw_client = tweepy.Client(
    consumer_key=os.environ["X_API_KEY"],
    consumer_secret=os.environ["X_API_SECRET"],
    access_token=os.environ["X_ACCESS_TOKEN"],
    access_token_secret=os.environ["X_ACCESS_SECRET"],
)

# Post a tweet
response = rw_client.create_tweet(text="Hello from Claude")
tweet_id = response.data["id"]
```

Use raw `requests` when you need direct control over headers, custom retry logic, or endpoints not yet wrapped by tweepy.

---

## Core Operations

### Post a Tweet

```python
# tweepy
response = rw_client.create_tweet(text="Hello from Claude")
tweet_id = response.data["id"]

# raw requests
oauth = make_oauth()
resp = oauth.post("https://api.x.com/2/tweets", json={"text": "Hello from Claude"})
resp.raise_for_status()
tweet_id = resp.json()["data"]["id"]
```

### Post a Thread (with per-tweet error handling)

```python
from dataclasses import dataclass

@dataclass
class ThreadResult:
    succeeded: list[str]   # tweet IDs posted successfully
    failed_at: int | None  # index of first failure, None if complete

def post_thread(client: tweepy.Client, tweets: list[str]) -> ThreadResult:
    """Post a thread. Returns IDs of successfully posted tweets and the
    index of the first failure (if any), so partial threads can be handled."""
    ids: list[str] = []
    reply_to: str | None = None

    for i, text in enumerate(tweets):
        try:
            kwargs: dict = {"text": text}
            if reply_to:
                kwargs["in_reply_to_tweet_id"] = reply_to
            resp = client.create_tweet(**kwargs)
            tweet_id = resp.data["id"]
            ids.append(tweet_id)
            reply_to = tweet_id
        except tweepy.TweepyException as e:
            return ThreadResult(succeeded=ids, failed_at=i)

    return ThreadResult(succeeded=ids, failed_at=None)

# Usage
result = post_thread(rw_client, ["Tweet 1", "Tweet 2", "Tweet 3"])
if result.failed_at is not None:
    print(f"Thread partial — failed at tweet {result.failed_at}")
    print(f"Posted IDs: {result.succeeded}")
```

### Search Tweets

`search/recent` covers the last **7 days only**. Full-archive search (`search/all`) requires elevated API access.

```python
# tweepy
results = read_client.search_recent_tweets(
    query="claude code -is:retweet lang:en",
    max_results=10,
    tweet_fields=["created_at", "public_metrics", "author_id"],
)
for tweet in results.data or []:
    print(tweet.text, tweet.public_metrics)

# raw requests
resp = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    headers=HEADERS,
    params={
        "query": "from:example_user -is:retweet",
        "max_results": 10,
        "tweet.fields": "public_metrics,created_at",
    },
)
resp.raise_for_status()
data = resp.json()
```

**Query operators:** `from:user`, `-is:retweet`, `lang:en`, `has:images`, `has:links`, `-is:reply`

### Read User Timeline

```python
# Requires knowing the numeric user_id — resolve from username first
user = read_client.get_user(username="example_user")
user_id = user.data.id

results = read_client.get_users_tweets(
    id=user_id,
    max_results=10,
    tweet_fields=["created_at", "public_metrics"],
    exclude=["retweets", "replies"],
)
```

### Get User by Username

```python
user = read_client.get_user(
    username="example_user",
    user_fields=["public_metrics", "description", "created_at"],
)
print(user.data.public_metrics)   # followers_count, following_count, tweet_count
```

### Upload Media and Post

Media upload uses the v1.1 endpoint (v2 media upload is not yet GA). Requires OAuth 1.0a:

```python
import tweepy

# Media upload requires v1.1 API — use tweepy's v1 client alongside v2
auth = tweepy.OAuth1UserHandler(
    os.environ["X_API_KEY"],
    os.environ["X_API_SECRET"],
    os.environ["X_ACCESS_TOKEN"],
    os.environ["X_ACCESS_SECRET"],
)
api_v1 = tweepy.API(auth)

# Upload — always use a context manager to avoid file handle leaks
with open("image.png", "rb") as f:
    media = api_v1.media_upload(filename="image.png", file=f)

# Post with media via v2 client
rw_client.create_tweet(
    text="Check this out",
    media_ids=[media.media_id_string],
)
```

---

## Pagination

The v2 API uses cursor-based pagination via `next_token`. Always iterate to completion for production use:

```python
def search_all_pages(
    query: str,
    max_results_per_page: int = 100,
    page_limit: int = 10,
) -> list[dict]:
    """Fetch up to page_limit pages of search results."""
    all_tweets = []
    next_token = None

    for _ in range(page_limit):
        params = {
            "query": query,
            "max_results": max_results_per_page,
            "tweet.fields": "created_at,public_metrics",
        }
        if next_token:
            params["next_token"] = next_token

        resp = requests.get(
            "https://api.x.com/2/tweets/search/recent",
            headers=HEADERS,
            params=params,
        )
        resp.raise_for_status()
        data = resp.json()

        all_tweets.extend(data.get("data", []))

        next_token = data.get("meta", {}).get("next_token")
        if not next_token:
            break   # no more pages

    return all_tweets
```

tweepy handles this automatically via `Paginator`:

```python
for tweet in tweepy.Paginator(
    read_client.search_recent_tweets,
    query="claude code -is:retweet",
    tweet_fields=["created_at", "public_metrics"],
    max_results=100,
).flatten(limit=500):    # flatten across pages, cap at 500 total
    print(tweet.text)
```

---

## Rate Limits

Rate limits vary by endpoint, auth method, and API tier (Free, Basic, Pro, Enterprise). **Do not hardcode them** — they change. Read headers at runtime:

```python
import time

def check_rate_limit(resp: requests.Response, warn_threshold: int = 5) -> None:
    remaining = int(resp.headers.get("x-rate-limit-remaining", 999))
    if remaining < warn_threshold:
        reset = int(resp.headers.get("x-rate-limit-reset", 0))
        wait = max(0, reset - int(time.time()))
        print(f"Rate limit low ({remaining} remaining). Resets in {wait}s")

def wait_if_rate_limited(resp: requests.Response) -> None:
    if resp.status_code == 429:
        reset = int(resp.headers.get("x-rate-limit-reset", int(time.time()) + 60))
        wait = max(1, reset - int(time.time()))
        print(f"Rate limited. Waiting {wait}s...")
        time.sleep(wait)
```

tweepy's `wait_on_rate_limit=True` handles this automatically:

```python
client = tweepy.Client(..., wait_on_rate_limit=True)
```

---

## Error Handling

```python
import tweepy

def post_tweet_safe(client: tweepy.Client, text: str) -> str | None:
    """Post a tweet. Returns tweet ID on success, None on known failure."""
    try:
        resp = client.create_tweet(text=text)
        return resp.data["id"]
    except tweepy.errors.Forbidden as e:
        # 403 — permission issue, duplicate tweet, or content policy
        print(f"Forbidden: {e}")
        return None
    except tweepy.errors.TooManyRequests as e:
        # 429 — rate limit (shouldn't happen if wait_on_rate_limit=True)
        print(f"Rate limit hit: {e}")
        raise
    except tweepy.errors.TwitterServerError as e:
        # 5xx — X infrastructure issue, retry is appropriate
        print(f"X server error: {e}")
        raise
    except tweepy.errors.TweepyException as e:
        # Other API error
        print(f"API error: {e}")
        raise

# Raw requests error handling
def handle_response(resp: requests.Response) -> dict:
    if resp.status_code == 201:
        return resp.json()
    if resp.status_code == 429:
        reset = resp.headers.get("x-rate-limit-reset", "unknown")
        raise Exception(f"Rate limited. Resets at epoch {reset}")
    if resp.status_code == 403:
        detail = resp.json().get("detail", "check permissions and token scopes")
        raise PermissionError(f"Forbidden: {detail}")
    resp.raise_for_status()   # raises HTTPError for all other 4xx/5xx
    return resp.json()
```

**Common error codes:**
- `403` — wrong token type (e.g., bearer token used for write), duplicate tweet, content policy
- `429` — rate limit exceeded
- `401` — invalid or expired credentials
- `400` — malformed request body

---

## Content Policy Compliance

Account suspensions from automation policy violations are difficult to reverse. Before shipping any automated posting:

- **No duplicate content** — posting the same or near-identical text repeatedly triggers spam detection
- **No unsolicited @mentions at scale** — mass-mentioning users is classified as mention spam
- **Respect rate limits** — aggressive posting that hits limits repeatedly signals bot behavior
- **No coordinated inauthentic behavior** — multiple accounts posting the same content simultaneously
- **Label automated accounts** — bots must be clearly identified per X's developer policy
- **Read the Automation Rules** — `developer.x.com/en/developer-terms/policy` before deploying

---

## Security

- Never hardcode tokens — use environment variables or a secrets manager
- Never commit `.env` files — add to `.gitignore` before first commit
- Rotate tokens immediately if exposed — regenerate at `developer.x.com`
- Use read-only tokens (bearer token) when write access is not needed
- Never log raw tokens, OAuth headers, or API secrets
- Scope app permissions to the minimum required — read-only if posting isn't needed

---

## Related Skills

- `content-engine` — Generate platform-native content for X (character limits, thread structure, hooks)
- `crosspost` — Distribute content across X, LinkedIn, and other platforms
- `article-writing` — Long-form content to thread breakdowns
