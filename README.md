# Mashape Analytics Agent Spec

Agents are libraries that can act as a middleweare / helper utility in reading request & response objects at the start of the request before processing, and at the end of the response before sending, in order to construct [ALF objects](https://github.com/Mashape/api-log-format) which can be used with the [Mashape Analytics](https://www.apianalytics.com/) service.

## Format

The Agent will use [API Log Format](https://github.com/Mashape/api-log-format) *(ALF)* to create log entries.

## Lifecycle

```
   ┌───────────────────────────────┐
   │ [client] new request          │──┐
   └───────────────────────────────┘  │
┌─────────────────────────────────────┘
│  ┌───────────────────────────────┐
└─▶│ [agent] process request       │──┐
   └───────────────────────────────┘  │
┌─────────────────────────────────────┘
│  ┌───────────────────────────────┐
└─▶│ [application] process request │──┐
   └───────────────────────────────┘  │
┌─────────────────────────────────────┘
│  ┌───────────────────────────────┐
└─▶│ [application] create response │──┐
   └───────────────────────────────┘  │
┌─────────────────────────────────────┘
│  ┌───────────────────────────────┐       ┌───────────────────────────┐
└─▶│ [agent] process response      │──┬───▶│ [agent] queue log entries │──┐
   └───────────────────────────────┘  │    └───────────────────────────┘  │
┌─────────────────────────────────────┘ ┌─────────────────────────────────┘
│  ┌───────────────────────────────┐    │  ┌───────────────────────────┐
└─▶│ [application] send response   │    └─▶│ [agent] send log entries  │
   └───────────────────────────────┘       └───────────────────────────┘
```

## Configurations

- Environment variables **must** be prefixed with: `MASHAPE_ANALYTICS_AGENT`
- Libraries can **optionally** allow to override the environment variables with local variables at initiation time

| Name              | Description                                                                                             | Required | Default |
| ----------------- | ------------------------------------------------------------------------------------------------------- | -------- | ------- |
| `SERVICE_TOKEN`   | [Mashape Analytics](https://www.apianalytics.com/) Service Token                                        | **yes**  | `-`     |
| `ENVIRONMENT`     | [Mashape Analytics](https://www.apianalytics.com/) Environment Name                                     | *no*     | `-`     |
| `MAX_BODY_SIZE`   | limit captured *request & response* body sizes *(in bytes)*                                             | *no*     | `0`     |
| `QUEUE_ALF`       | num of [ALF](https://github.com/Mashape/api-log-format) objects to queue before sending                 | *no*     | `10`    |
| `QUEUE_ENTRIES`   | num of HAR Entries per [ALF](https://github.com/Mashape/api-log-format) object to queue before sending  | *no*     | `100`   |
| `TIMEOUT`         | timeout *in seconds* before the Agent flushes the queues                                                | *no*     | `1`     |
| `TIMEOUT`         | timeout *in seconds* before the Agent flushes the queues                                                | *no*     | `1`     |
| `SSL`             | use SSL to encrypt connection to Socket Server                                                          | *no*     | `true`  |


### Shared Configuration

These advanced configuration options are shared with other Mashape Analytics Systems, and are used for on-premise installations of Mashape Analytics.

- unlike standard configurations, these environment variables **must** be prefixed with: `MASHAPE_ANALYTICS_`

| Name                 | Description                                                | Required | Default                        |
| -------------------- | ---------------------------------------------------------- | -------- | ------------------------------ |
| `SOCKET_SERVER_ADDR` | hostname or IP address of Mashape Analytics Socket Server  | **yes**  | `socket.analytics.mashape.com` |
| `SOCKET_SERVER_PORT` | Socket Server communication port                           | **yes**  | `443`                          |

## Capturing Data

### clientIPAddress

- The Agent should not Trust the client IP address provided from the application framework
- The Agent should parse all appropriate header schemas in order to find the client IP address
- Depending on the implementation language, some helper libraries or packages might exist to facilitate this, those should be tested to ensure they cover all the schemas below
- The default fallback should always be the IP address assigned by the network interface

#### Known Proxy IP Header Schemas

- [`RFC7239`](http://tools.ietf.org/html/rfc7239): The **ONLY** official standard.
- [`CF-Connecting-IP`](https://support.cloudflare.com/hc/en-us/articles/200170986-How-does-CloudFlare-handle-HTTP-Request-headers-): for applications behind Cloudflare's cloud proxies
- [`Fastly-Client-IP`](https://docs.fastly.com/guides/tls/how-can-i-find-the-original-ip-when-using-tls-termination): for applications using Fastly caching proxies
- `X-Cluster-Client-IP`: RiverBed Stingray (used by Rackspace)
- `X-Real-IP`: a common, nonstandard header, often used with Nginx.
- `X-Forwarded-For`: most common, nonstandard header, typically referred to as `XFF`

### Request

- The Agent should attempt to get the RAW Request as early as possible in the application lifecycle, before any processing *(decompression, modification, normalization, etc...)* by the application or application framework
  - This is to ensure all original headers and body are captured properly
- In many languages *(especially: PHP, Node.js)* reading the input stream is awarded to the first listener, the stream is then flushed, thus blocking any following listeners from reading
  - This is important to keep in mind for Agents attempting to parse the request bodies so not to prevent the user's application from getting the request data for its own processing

### Response

- The Agent should only attempt to process the response object at the time the application is ready to send it
  - Just as with the request scenario, this is to ensure all possible headers and final modifications to the response objects are captured.

## Body Size

- If the bodies *were* successfully captured, and in case the `Content-Length` header is not present, the Agent should attempt to calculate the request & response body size manually (in bytes)

## Headers

- When not readily available, the Agent should attempt to calculate headers sizes: *`ALF.har.log.entries[].request,headersSize`, `ALF.har.log.entries[].response.headersSize`* by reconstructing the [HTTP Message](http://httpwg.github.io/specs/rfc7230.html#http.message) from the start of the HTTP request/response message until (and including) the double CRLF before the body.

## Timings

- The Agent should attempt to capture start time: *`ALF.har.log.entries[].startedDateTime`* and measure the wait time: *`ALF.har.log.entries[].time`*
  - The Agent is best suited to do this because of its position in the [Lifecycle](#Lifecycle)

## Debugging

Agents should conditionally write to `stderr` based on the existence of the **agent name** in the environment variable: `MASHAPE_DEBUG`.

###### Example

```
MASHAPE_DEBUG=analytics-agent-node,analytics-server
```
