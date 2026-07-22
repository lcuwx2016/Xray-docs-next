# XHTTP: Beyond REALITY

XHTTP is a native HTTP transport with three modes (`packet-up`, `stream-up`, and `stream-one`). It can pass through most HTTP-capable middleboxes, including CDNs and reverse proxies, and natively supports QUIC H3 through a CDN.

It provides true upload/download separation, header padding, XMUX multiplexing, and broad deployment compatibility.

::: tip
XHTTP multiplexes by default. It has lower latency than Vision, but multi-threaded speed tests may perform worse unless you set `"maxConcurrency": 1` before testing.
:::

::: danger
Do not enable mux.cool with XHTTP. Recent Xray servers enforce this and accept only plain XUDP.
:::

## Quick start

For either TLS or REALITY, you normally only need to set `path`:

1. **Set only `path`; leave the other options unset.**
2. If the server supports QUIC H3, set the client `alpn` to `"h3"`.
3. When using an optimized CDN IP, put the IP in client `address` and the domain in `serverName` (SNI).
4. If Cloudflare cannot connect, enable gRPC support in the Cloudflare dashboard.
5. If Nginx cannot proxy the connection, replace `proxy_pass` with `grpc_pass`.
6. If another CDN or reverse proxy is incompatible, select `"packet-up"`, the most compatible mode.

::: warning
Some CDNs terminate HTTP connections that have no actual downstream data for a long time. With Cloudflare, long-lived sessions such as SSH should also use application-level keepalive: set `ClientAliveInterval` in the SSH server's `sshd_config`, or set `ServerAliveInterval` on the SSH client. Do not rely solely on XHTTP transport keepalive.
:::

## XHTTPObject

`XHTTPObject` is used by `xhttpSettings` in [`StreamSettingsObject`](../transport.md#streamsettingsobject).

```json
{
  // outbound example; the same settings can be used for an inbound
  "outbounds": [
    {
      // ...
      "streamSettings": {
        "method": "xhttp",
        // [!code focus:36]
        "xhttpSettings": {
          "host": "example.com",
          "path": "/yourpath",
          "mode": "auto",
          "extra": {
            "headers": {
              "Key": "Value"
            },
            "xPaddingBytes": "100-1000",
            "noGRPCHeader": false,
            "noSSEHeader": false,
            "scMaxEachPostBytes": 1000000,
            "scMinPostsIntervalMs": 30,
            "scMaxBufferedPosts": 30,
            "scStreamUpServerSecs": "20-80",
            "xmux": {
              "maxConcurrency": "16-32",
              "maxConnections": 0,
              "cMaxReuseTimes": 0,
              "hMaxRequestTimes": "600-900",
              "hMaxReusableSecs": "1800-3000",
              "hKeepAlivePeriod": 0
            },
            "downloadSettings": {
              "address": "",
              "port": 443,
              "network": "xhttp",
              "security": "tls",
              "tlsSettings": {},
              "xhttpSettings": {
                "path": "/yourpath"
              },
              "sockopt": {}
            }
          }
        }
      }
    }
  ]
}
```

> `host`: string

The HTTP Host header sent by the client. Client priority is `host` > `serverName` > `address`.

When configured on the server, the server checks that the client value matches. Otherwise it performs no Host check. It is generally best left unset, and it must not be placed in `headers`. The default is empty.

> `path`: string

The HTTP request path used by XHTTP. The default is `"/"`. The server associates upload and download streams using a random UUID under this path.

> `mode`: "auto" | "packet-up" | "stream-up" | "stream-one"

Transport mode. The default is `"auto"`.

Client `"auto"` behavior:

- **TLS H2** → `stream-up`
- **REALITY** → `stream-one` (`stream-up` when `downloadSettings` is present)
- **Otherwise** → `packet-up`

Server `"auto"` accepts all three modes. A specific mode normally accepts only that mode, except `"stream-up"`, which also accepts `stream-one`.

| Mode         | Upload                | Download                       | Compatibility | Typical use                                                |
| ------------ | --------------------- | ------------------------------ | ------------- | ---------------------------------------------------------- |
| `packet-up`  | Chunked POST requests | Streaming GET                  | Highest       | CDNs and diverse middleboxes                               |
| `stream-up`  | Streaming POST        | Streaming GET                  | High          | TLS H2 through a CDN, REALITY                              |
| `stream-one` | Streaming POST body   | Response body of the same POST | Medium        | REALITY and middleboxes supporting bidirectional streaming |

> `extra`: object

Raw JSON container for every option other than `host`, `path`, and `mode`. When `extra` is present, only these four fields (`host`, `path`, `mode`, and `extra`) take effect.

Share links and GUIs generally expose only these four fields. The service publisher should prepare and distribute `extra`; clients should not modify it arbitrarily.

### Common extra options

> `headers`: map {string: string}

Client only. Custom HTTP header key-value pairs. Empty by default.

> `xPaddingBytes`: string | number

Header-padding size used to avoid fixed-length request and response headers. Client requests use `Referer: ...?x_padding=XXX...`; server responses use `X-Padding: XXX...`. Using `Referer` avoids unnecessary CORS preflight requests with Browser Dialer. The server validates that client padding is within its allowed range.

Ranges such as `"100-1000"` are randomized for every request or response. The default is `"100-1000"`.

### HTTP appearance and obfuscation options

These options change how padding, session metadata, and upload data appear in HTTP requests. They are intended mainly for compatibility with CDNs or WAFs that filter fixed XHTTP signatures. Client and server settings must be mutually compatible. They reduce static signatures but do not make traffic inherently undetectable.

```json
{
  "xPaddingObfsMode": true,
  "xPaddingKey": "_dc",
  "xPaddingHeader": "Referer",
  "xPaddingPlacement": "queryInHeader",
  "xPaddingMethod": "tokenish",
  "uplinkHTTPMethod": "POST",
  "sessionPlacement": "path",
  "sessionKey": "",
  "seqPlacement": "path",
  "seqKey": "",
  "uplinkDataPlacement": "auto",
  "uplinkDataKey": "X-Data",
  "uplinkChunkSize": "3000-4000",
  "serverMaxHeaderBytes": 0
}
```

> `xPaddingObfsMode`: true | false

Enables the configurable padding appearance. The default is `false`, preserving the legacy `Referer: ...?x_padding=XXX...` behavior for compatibility with old clients. When enabled, the following padding key, header, placement, and method settings control the new appearance.

> `xPaddingKey`: string

Name of the query parameter or cookie carrying padding. Default: `"x_padding"`.

> `xPaddingHeader`: string

When placement is `"queryInHeader"`, this HTTP header carries the URL containing the padding query. With `"header"`, it carries the padding directly. Default: `"X-Padding"`; legacy client requests still use `Referer`.

> `xPaddingPlacement`: "queryInHeader" | "query" | "header" | "cookie"

Padding location. The default `"queryInHeader"` puts padding in a URL query and that URL in the selected HTTP header. The other values put it directly in the request query, an HTTP header, or a cookie.

> `xPaddingMethod`: "repeat-x" | "tokenish"

Padding generation method. `"repeat-x"` repeats `X` and is the compatible default. `"tokenish"` generates random token-like characters and adjusts their size for HTTP header compression.

> `uplinkHTTPMethod`: string

HTTP method for upload requests. It is case-insensitive and normalized to uppercase. Default: `"POST"`. A middlebox may require `PUT`, `PATCH`, `GET`, or another method; `GET` is available only in `packet-up`. Xray does not probe or rotate methods automatically, so the selected method must pass every CDN and reverse proxy in the path.

> `sessionPlacement`: "path" | "query" | "header" | "cookie"

Session ID location. Default: `"path"`. For other placements, `sessionKey` customizes the name. Its defaults are `x_session` for query/cookie and `X-Session` for a header.

> `seqPlacement`: "path" | "query" | "header" | "cookie"

Location of the `packet-up` sequence number. Default: `"path"`. For other placements, `seqKey` customizes the name. Its defaults are `x_seq` for query/cookie and `X-Seq` for a header.

> `uplinkDataPlacement`: "auto" | "body" | "header" | "cookie"

Upload-data location. Default: `"auto"`; normal uploads use the body, while the server can automatically accept supported placements. `"header"` and `"cookie"` are available only in `packet-up`; data is Base64-encoded and split into chunks. These modes are intended for exceptional paths that only permit GET or reject request bodies, not for normal deployments.

> `uplinkDataKey`: string

Base name used for upload chunks in headers or cookies. Defaults to `x_data` for cookies and `X-Data` for headers/auto.

> `uplinkChunkSize`: number | string

Size of each Base64 chunk when upload data is carried in headers or cookies. Ranges are supported for randomization. This limits one chunk only; `scMaxEachPostBytes` still controls the total upload data in one request.

> `serverMaxHeaderBytes`: number

Server only. Changes the total request-header limit accepted by XHTTP's HTTP server. `0` uses the implementation default; negative values are invalid. Increase it only when you control the complete path, because a CDN or reverse proxy may still enforce a lower limit.

::: warning
Header/cookie upload adds Base64 overhead and is constrained by individual-field, total-header, reverse-proxy, and CDN limits. For HTTP `431`, reduce both `uplinkChunkSize` and `scMaxEachPostBytes`. More requests may also require a larger server-side `scMaxBufferedPosts`. Increasing `serverMaxHeaderBytes` alone cannot change upstream middlebox limits.
:::

> `noGRPCHeader`: true | false

Client only (`stream-up` / `stream-one`). Disables the upload-side `Content-Type: application/grpc` disguise. Default: `false`.

> `noSSEHeader`: true | false

Server only. Disables the download-side `Content-Type: text/event-stream` disguise. Default: `false`. If the combination of a gRPC request header and an SSE response header prevents `stream-one` from passing through a middlebox, try `true`.

### packet-up extra options

Here, sc means sub-connection. Limits are counted independently for each proxied connection, even when several of them share the same underlying H2/H3 connection.

> `scMaxEachPostBytes`: number | string

Maximum client data per chunked upload request, in bytes. The field retains `Post` in its name for compatibility and also applies to PUT, PATCH, and GET. It must remain below the limit imposed by the CDN or other middlebox; the server rejects oversized requests. Supports ranges such as `"500000-1000000"`. Default: `1000000` (1 MB).

> `scMinPostsIntervalMs`: number | string

Client only. Minimum interval in milliseconds between POSTs for one proxied connection. Supports ranges such as `"10-50"`. Default: `30`.

> `scMaxBufferedPosts`: number

Server only. Maximum number of POSTs buffered for one proxied connection before it is terminated. Default: `30`.

### stream-up extra options

> `scStreamUpServerSecs`: number | string

Server only. The server periodically sends `xPaddingBytes` bytes of padding to keep the connection alive and prevent a CDN from terminating an idle download together with the stream-up upload. Supports ranges such as `"20-80"`; this is also the default. Set `-1` to disable it, in which case the server may not send response headers immediately, matching older behavior.

### XMUX extra options

Client only. Controls H2/H3 multiplexing.

::: warning
After any XMUX field is explicitly set, the remaining fields no longer receive automatic defaults and must also be set explicitly.
:::

> `maxConcurrency`: number | string

Maximum simultaneous proxied connections per TCP/QUIC connection. Xray opens another connection when this limit is reached. Conflicts with `maxConnections`. When all XMUX fields are zero, the default is a randomized `"16-32"`.

> `maxConnections`: number | string

Maximum number of simultaneous underlying connections. New proxy requests open new connections until this limit is reached, after which existing connections are reused. Conflicts with `maxConcurrency`. Default: `0` (unlimited). Ranges are supported.

> `cMaxReuseTimes`: number | string

Maximum number of times an underlying connection may be reused for new proxy requests. Default: `0` (unlimited). Ranges are supported.

> `hMaxRequestTimes`: number | string

Maximum cumulative HTTP requests per connection. Nginx defaults to 1000. When all XMUX fields are zero, the default is a randomized `"600-900"`; otherwise it is `0` (unlimited). Typically `stream-one` uses one HTTP request, `stream-up` uses two, and `packet-up` uses many. Counting is not exact because requests such as GET may be retried, so do not set it right at a middlebox limit.

> `hMaxReusableSecs`: number | string

Maximum time in seconds an underlying connection remains reusable. Nginx defaults to one hour. When all XMUX fields are zero, the default is a randomized `"1800-3000"`; otherwise it is `0` (unlimited).

> `hKeepAlivePeriod`: number

Client keepalive interval for idle H2/H3 connections. `0` means Chrome H2's 45 seconds or quic-go H3's 10 seconds. A negative value such as `-1` disables it. Keeping `0` is recommended. This is the only XMUX field that does not accept a range.

`maxConnections: 1` favors reuse of one underlying connection. `maxConcurrency: 1` creates more underlying connections under concurrent load and is useful for multi-threaded speed tests. The defaults intentionally rotate H2/H3 connections over time.

### Upload/download separation

> `downloadSettings`: object

Client only. A complete set of downstream `streamSettings`, plus `address` and `port` for selecting another entry point.

```json
{
  "downloadSettings": {
    "address": "",
    "port": 443,
    "network": "xhttp",
    "security": "tls",
    "tlsSettings": {},
    "xhttpSettings": {
      "path": "/yourpath"
    },
    "sockopt": {}
  }
}
```

`network` must be `"xhttp"` and cannot be omitted. `security` may be `"tls"` or `"reality"`. The downstream `xhttpSettings.path` must match the upload path.

`sockopt` can be shared. Setting `penetrate` to `true` in the upload-side `sockopt` overrides the downstream socket options.

::: warning
Apart from the `sockopt.penetrate` exception above, `downloadSettings` inherits nothing from the upload side. Address, port, TLS/REALITY, XHTTP, and XMUX settings must be declared independently. Even default XMUX ranges are randomized independently in each direction.
:::

Upload and download may use different addresses, SNI values, IPv4/IPv6, H2/H3, CDNs, or REALITY entry points. They must ultimately reach the same XHTTP inbound on the same server with the same `path`.

## How it works

### packet-up

1. By default, the client uploads using `POST /yourpath/sameUUID/seq`.
   - A random UUID associates upload and download.
   - The session is terminated if the server cannot associate both directions within 30 seconds.
   - `seq` starts at 0. A POST body must finish before the next begins, but its response need not be awaited.
   - The server reorders POSTs that arrive out of order.
   - UUID and `seq` are in the path by default. Appearance options can instead put them in a query, header, or cookie.
2. The client starts downloading with `GET /yourpath/sameUUID`.
   - Response headers include `X-Accel-Buffering: no`, `Cache-Control: no-store`, and `Content-Type: text/event-stream`.
   - HTTP/1.1 also uses `Transfer-Encoding: chunked`; H2/H3 do not need it.
3. Responses include `Access-Control-Allow-Origin: *` and `Access-Control-Allow-Methods: GET, POST`.

packet-up creates many POSTs, and `Referer` padding can make reverse-proxy access logs large. Adjust logging for these requests when audit logs are unnecessary.

### stream-up

Replaces packet-up's segmented upload with a streaming `POST /yourpath/sameUUID`, without sacrificing upload efficiency. The upload uses `Content-Type: application/grpc` by default.

### stream-one

Uses `POST /yourpath/`; its request body is the upload and its response body is the download. A trailing `/` is added automatically.

## H1 / H2 / H3

Many CDNs and reverse proxies convert client H3 into H1 or H2 toward the origin, so a client using H3 does not require the XHTTP origin to listen for H3 directly.

By default, the server listens on TCP for H1 and H2. It uses quic-go on UDP for H3 only when TLS is enabled and `alpn` contains only `"h3"`. It is generally preferable to place XHTTP behind a real Nginx or Caddy instance and let that proxy handle public H3, reducing implementation fingerprints exposed by the origin.

Client behavior:

1. TLS/REALITY defaults to H2; otherwise it uses HTTP/1.1.
2. An `alpn` containing only `"http/1.1"` selects H1.
3. An `alpn` containing only `"h3"` selects quic-go H3.
4. With Browser Dialer, the browser chooses the HTTP version, and `tlsSettings` does not control that connection's ALPN or TLS fingerprint.

::: tip
Set the log level to `"info"` to confirm the actual HTTP version, Host, XHTTP mode, and upload/download separation settings.
:::

## stream-up compared with gRPC

| Item             | stream-up                                                           | gRPC transport            |
| ---------------- | ------------------------------------------------------------------- | ------------------------- |
| Implementation   | No gRPC library; better performance                                 | Depends on a gRPC library |
| Download traffic | Independent GET, unaffected by CDN gRPC traffic limits              | Subject to CDN limits     |
| Enhancements     | Header padding, XMUX, upload/download separation, shareable `extra` | None                      |

## Troubleshooting

| Problem                                             | Check                                                                                            |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Cannot connect through Cloudflare                   | Enable gRPC support in the Cloudflare dashboard                                                  |
| Nginx cannot forward streaming uploads              | Use `grpc_pass`, not ordinary `proxy_pass`                                                       |
| Another CDN or reverse proxy is incompatible        | Change `mode` to `"packet-up"`                                                                   |
| `stream-one` is throttled or disconnected           | Try `"stream-up"`, or set server `"noSSEHeader": true`                                           |
| A long-lived connection disconnects after some time | Check `scStreamUpServerSecs` and configure application-level keepalive for protocols such as SSH |
| H3 was configured but is not used                   | Ensure client `alpn` contains only `"h3"` and check for Browser Dialer                           |
| Multi-threaded speed tests underperform             | Temporarily set `"maxConcurrency": 1`                                                            |
| Reverse-proxy logs are excessive                    | Adjust logging for packet-up POSTs and `Referer` padding                                         |
