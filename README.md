# Envoy JWT Claim to Header Extraction

This repo aims to provide a reproduction of an issue faced while extracting JWT claims to a request header, especifically  when that claim is a "url-formatted" claim, e.g. `https://example.org/some_claim`.

## What can you find here?

### The "stack"

This is a docker compose "stack" starting the following services:

- Envoy v1.29.2 (of course)

- [local-jwks-server](https://github.com/murar8/local-jwks-server)

From the original repo:
> This project provides a local server that can be used to serve a JSON Web Key Set endpoint for testing purposes.
> It can be used to test applications that rely on a JWKS endpoint for authentication, for example for mocking the
> auth0 signature verification.

- [echo-server](https://github.com/mendhak/docker-http-https-echo)

From the original repo:
> `mendhak/http-https-echo` is a Docker image that can echo various HTTP request properties back to client in the
> response, as well as in the Docker container logs. It comes with various options that can manipulate the response
> output, see the table of contents for a full list.

### Repository Structure

- `envoy/`: Contains two Envoy configs, `envoy.yaml` as a plain HTTP server, and `envoy-ssl.yaml` if you wish to use SSL. Check the section on [SSL/TLS](#optional-using-ssltls) for more information
- `keys/`: If you wish to use your own PEM-formatted private-key for the local jwks server
- `ssl/`: Contains the PEM-formatted SSL/TLS certificates. Place the private key in `ssl/private` and don't forget to apply the correct permissions if needed.
- `docker-compose.yaml`: Contains the definitions for the issue repro stack
- `rest-client.http`: If using a IDE or another HTTP/REST client capable of understanding these files

## [OPTIONAL] Using SSL/TLS

While SSL/TLS is disabled by default, to ease the issue reproduction, you can use SSL/TLS if you so wish.
You will need to provide a certificate(s) and the respective key(s). 
You may also use [MKCert](https://github.com/FiloSottile/mkcert) if you wish, caveats apply

1. Make the necessary changes to the `docker-compose.yaml` file by uncommenting the SSL/TLS sections
2. Paste your certificate(s) on `ssl/cert.crt` and your key on `ssl/private/cert.key`
(or replace these files and ensure the correct file permissions)

## [OPTIONAL] Docker Static IPs & DNS resolution

The `docker-compose.yaml` file contains some commented `network` configurations that uses manually-assigned IPs and `extra_hosts` to add a `example.org` entry pointing to the Docker host.
If you do use them, don't forget to also add the entry/entries to your `/etc/hosts` file.

---

## The issue

While envoy is capable of extracting jwt claims to http headers, on some instances, this extraction might fail,
 for instance, when that claim contains special characters, like those found on urls.

To demonstrate this issue, this repo contains a envoy configuration that is setup to extract different claims into
 different headers.
The requests are then forwarded to the `echo-server`, which as the name implies, will echo back the request, formatted as JSON.

Given the following claim

```json
{
  "iss": "http://example.org/",
  "sub": "johndoe@example.org",
  "iat": 1712240289,
  "exp": 1743776289,
  "aud": "http://example.org",
  "flavour": "chocolate",
  "parent_token": "abc",
  "some_url_value": "http://example.org/about",
  "http://example.org/parent_token": "xyz"
}
```
and the following `claim_to_headers` block

```yaml
claim_to_headers:
- header_name: cookie
  claim_name: flavour
- header_name: x-subject
  claim_name: sub
- header_name: x-simple-claim
  claim_name: parent_token
- header_name: x-url-value-claim
  claim_name: some_url_value
- header_name: x-url-key-claim
  claim_name: http://example.org/parent_token
- header_name: x-quoted-claim
  claim_name: 'http://example.org/parent_token'
- header_name: x-regex-1-claim
  claim_name: http:\/\/example.org\/parent_token
- header_name: x-regex-2-claim
  claim_name: http:\\/\\/example\\.org\\/parent_token
```
it is expected to obtain the following headers and respective values:

| Claim                                | Header            | Expected Value             | Present?           |
| :----------------------------------- | :---------------- | :------------------------- | :----------------- |
| `flavour`                            | cookie            | `chocolate`                | :white_check_mark: |
| `sub`                                | x-subject         | `johndoe@example.org`      | :white_check_mark: |
| `parent_token`                       | x-simple-claim    | `abc`                      | :white_check_mark: |
| `some_url_value`                     | x-url-value-claim | `http://example.org/about` | :white_check_mark: |
| `http://example.org/parent_token`    | x-url-key-claim   | `xyz`                      | :boom:             |
| `http://example.org/parent_token`    | x-quoted-claim    | `xyz`                      | :boom:             |
| `http://example.org/parent_token`    | x-regex-1-claim   | `xyz`                      | :boom:             |
| `http://example.org/parent_token`    | x-regex-2-claim   | `xyz`                      | :boom:             |

However, on some of these instances, the request/extraction will **NOT** contain some of the expected headers, as listed on the `Present?` column of the previous table.

### LUA-based Claim to Header extraction

Additionally, this reproduction also contains a [Lua script](./envoy/config.yaml#L145) that also performs the token extraction, and in this situation, there is no issue with the characters on the token claim key nor value, and we can see the expected output on the `/echo` response

| Claim                                | Header                 | Expected Value             | Present?           |
| :----------------------------------- | :----------------      | :------------------------- | :----------------- |
| `flavour`                            | Lua-Flavour-claim      | `chocolate`                | :white_check_mark: |
| `sub`                                | Lua-User-claim         | `johndoe@example.org`      | :white_check_mark: |
| `parent_token`                       | Lua-Parent-Token-claim | `abc`                      | :white_check_mark: |
| `some_url_value`                     | Lua-Url-Value-claim    | `http://example.org/about` | :white_check_mark: |
| `http://example.org/parent_token`    | Lua-Url-Key-claim      | `xyz`                      | :white_check_mark: |

### How to reproduce

1. Start up the docker-compose stack

```bash
docker compose up
```

2. Validate everything is up and in a working state by ensuring you get responses to the following requests

```bash
# Validate the JWKS server is working by ensuring you get a jwks back:
curl --request GET \
  --url http://localhost/.well-known/jwks.json \
  --header 'accept: application/json' \
  --header 'content-type: application/json'

# Validate the echo server is working by getting a JSON echo of this request:
curl --request GET \
  --url http://localhost/echo \
  --header 'accept: application/json' \
  --header 'content-type: application/json'
```

3. If everything is in working order, then go ahead and sign a JWT (by providing it as input):

```bash
curl --request POST \
  --url http://localhost/jwt/sign \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{"iss": "http://example.org/","sub": "johndoe@example.org","iat": 1712240289,"exp": 1743776289,"aud": "http://example.org","flavour": "chocolate","parent_token": "abc","some_url_value": "http://example.org/about","http://example.org/parent_token": "xyz"}'

# If you have jq and want to create an ENV variable with the output use this command:
export AUTH_TOKEN=$(curl --request POST \
  --url http://localhost/jwt/sign \
  --header 'accept: application/json' \
  --header 'content-type: application/json' \
  --data '{"iss": "http://example.org/","sub": "johndoe@example.org","iat": 1712240289,"exp": 1743776289,"aud": "http://example.org","flavour": "chocolate","parent_token": "abc","some_url_value": "http://example.org/about","http://example.org/parent_token": "xyz"}')
```

4. With the obtained token, issue another call to the echo server and provide the token in the `Authorization` header:

```bash
curl --request POST \
  --url http://localhost/echo \
  --header 'accept: application/json' \
  --header 'authorization: Bearer <YOUR_TOKEN>' \
  --header 'content-type: application/json' \
  --header 'cookie: vanilla' \
  --cookie 'vanilla' \
  --data '{"ping": "pong"}'

# If you have created an ENV variable with the token, use the following command:
curl --request POST \
  --url http://localhost/echo \
  --header 'accept: application/json' \
  --header 'authorization: Bearer '${AUTH_TOKEN} \
  --header 'content-type: application/json' \
  --header 'cookie: vanilla' \
  --cookie 'vanilla' \
  --data '{"ping": "pong"}'
```

5. You'll now verify that we are now missing some of the expected headers from the claim-to-header extraction

## Open Questions

1. Is this not expected functionality?
2. If it IS expected functionality, where is it breaking? It's not clear from the `jwt` or `golang` logs nor others tested.
  2.1. Which log(s) should expose this error?
3. If it needs to be treated as a regex, what is the correct way to escape the string?
  3.1. Go format?
  3.2. Javascript format?
  3.3. Other?

