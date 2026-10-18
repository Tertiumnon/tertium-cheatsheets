# JWT

## Token body

_Base64Url_ encoded strings with "." delimiter.

{header}.{payload}.{signature}

### Header

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload

```json
{
  "iss": "itCanBeUserId",
  "iat": "2024-01-04T09:29:25.593Z"
}
```

### Signature

Example:

```js
HMACSHA256(base64UrlEncode(header) + '.' + base64UrlEncode(payload), secret);
```

## Usage

### Token Types

| Question             | Access token                      | Refresh token                                      |
| -------------------- | --------------------------------- | -------------------------------------------------- |
| What to store        | iss ("Issuer"), iat ("Issued At") | iss ("Issuer"), iat ("Issued At")                  |
| Where to store       | Application memory                | Application secured cookies                        |
| Recommended Lifetime | 5 min - 1 day                     | 60 min - 30 days OR after Access Token refreshing  |
| When to send         | Every request                     | If HTTP status in response is 401 ("Unauthorized") |
| How to send          | HTTP headers OR cookies           | As HTTP parameter                                  |

### How to invalidate token

- Check "Issued At"
- If "Issued At" has more lifetime period than expected - return "Unauthorized" (401) HTTP error.
