## How I Found and Fixed the Bug

Running `pytest -v`, the test
`test_api_request_refreshes_when_token_is_dict` failed because the
`Authorization` header was missing. In `http_client.py`, I found that
the refresh logic only checked whether the token was `None` or an
expired `OAuth2Token`. If the token was a dictionary, neither condition
applied, so no refresh occurred.

I fixed this by adding:

    not isinstance(self.oauth2_token, OAuth2Token)

to the refresh condition. This ensures any non-`OAuth2Token` triggers a
refresh. After the change, all tests passed. I also added
`test_api_request_refreshes_when_token_is_dict_even_with_future_expiry`
to confirm that dictionary tokens always trigger a refresh.

## What Was the Bug?

The issue occurred when `Client.oauth2_token` was stored as a raw
dictionary instead of an `OAuth2Token` instance. In that case, the
client neither refreshed the token nor attached the `Authorization`
header.

## Why Did It Happen?

The refresh logic only handled two cases: no token or an expired
`OAuth2Token`. When the token was a dictionary: - It was truthy, so
`not self.oauth2_token` evaluated to `False`. - It was not an
`OAuth2Token`, so the expiration check was skipped.

As a result, refresh and header attachment were bypassed.

## One Realistic Case My Tests Still Don’t Cover?

The tests do not cover a token expiring during a long-running request.
Since expiration is checked only once before sending the request, a
mid-request expiration could cause retries to use stale credentials
until the next refresh cycle.