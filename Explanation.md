## How I found and fixed the bug

When I ran `pytest -v`, I noticed that `test_api_request_refreshes_when_token_is_dict` was failing because the `Authorization` header was missing. I traced the issue to `http_client.py` and discovered that the refresh condition only checked whether the token was `None` or an expired `OAuth2Token`.

If the token was a dictionary, neither condition was triggered, so the code simply fell through without refreshing the token or setting the header.

To fix this, I updated the refresh condition by adding:

    not isinstance(self.oauth2_token, OAuth2Token)

This ensures that any token that is not an `OAuth2Token` instance triggers a refresh. After applying the fix, I re-ran the test suite and confirmed that all tests passed.

I also added a new test, `test_api_request_refreshes_when_token_is_dict_even_with_future_expiry`, to ensure that dictionary-based tokens with a future expiry still trigger a refresh. The client should only operate with `OAuth2Token` instances, never raw dictionaries.

## What was the bug?

The bug occurred when `Client.oauth2_token` was stored as a raw dictionary (for example, after being deserialized from JSON) instead of as an `OAuth2Token` instance.

In this case, the client neither refreshed the token nor added the `Authorization` header to API requests. As a result, requests made with `api=True` were sent without authentication when the token was stored as a dictionary.

## Why did it happen?

The refresh condition only handled two cases:

- `not self.oauth2_token` (token is `None` or missing)
- `isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired`

When the token was a dictionary:

- It was truthy, so `not self.oauth2_token` evaluated to `False`
- It was not an instance of `OAuth2Token`, so the expired check was skipped

Because neither condition applied, the refresh logic was bypassed. Additionally, since the header was only set when the token was an `OAuth2Token`, the request was sent without authentication. The dictionary case effectively fell through both branches.

## Why does my fix solve it?

My fix adds:

    not isinstance(self.oauth2_token, OAuth2Token)

to the refresh condition.

Now, any value that is not an `OAuth2Token` (including dictionaries, strings, or other invalid types) triggers a refresh. After the refresh completes, `self.oauth2_token` becomes a proper `OAuth2Token` instance, ensuring that the `Authorization` header is correctly set for API requests.

## One realistic case my tests still don't cover

My tests do not cover the scenario where `oauth2_token` is a valid `OAuth2Token` at the time of the check but expires during a long-running request or retry loop.

Currently, the expiration check happens only once before the request is made. If the token expires after that check but before the response is received, a subsequent retry could use a stale token until the next refresh cycle.

A more robust implementation would re-check token expiration before each retry or implement a more proactive token lifetime management strategy.
