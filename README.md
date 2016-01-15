# Galileo Analytics Agent Spec

Agents are libraries that can act as a middleweare / helper utility in reading incoming requests & outgoing responses data in order to construct [ALF objects][api-log-format] which can be used with the [Galileo][galileo] service.

## Lifecycle

Agents need to appropriately be injected at the appropriate point in the request-response lifecycle, at the start of the request before processing any business logic, and at the end of the response before sending back to the client.

```
   ┌──────────────────────────────────────┐   ┌────────────────────────────┐
   │                                      │   │                            │
   │ [server] last request byte received  │──▶│  [agent] log request data  │──┐
   │                                      │   │                            │  │
   └──────────────────────────────────────┘   └────────────────────────────┘  │
┌─────────────────────────────────────────────────────────────────────────────┘
│  ┌──────────────────────────────────────┐                                         ┌────────────────────────────┐
│  │                                      │                                         │                            │
└─▶│     [application] business logic     │──┐                                 ┌───▶│    [agent] add to queue    │──┐
   │                                      │  │                                 │    │                            │  │
   └──────────────────────────────────────┘  │                                 │    └────────────────────────────┘  │
┌────────────────────────────────────────────┘                                 │ ┌──────────────────────────────────┘
│  ┌──────────────────────────────────────┐   ┌────────────────────────────┐   │ │  ┌────────────────────────────┐
│  │                                      │   │                            │   │ │  │                            │
└─▶│  [server] last response byte ready   │──▶│ [agent] log response data  │──┬┘ └─▶│  [agent] send to Galileo   │──┐
   │                                      │   │                            │  │     │                            │  │
   └──────────────────────────────────────┘   └────────────────────────────┘  │     └────────────────────────────┘  │
┌─────────────────────────────────────────────────────────────────────────────┘  ┌──────────────────────────────────┘
│  ┌──────────────────────────────────────┐                                      │  ┌────────────────────────────┐
│  │                                      │                                      │  │                            │
└─▶│        [server] send response        │                                      └─▶│    [agent] flush queue     │
   │                                      │                                         │                            │
   └──────────────────────────────────────┘                                         └────────────────────────────┘
```

## Connectivity

There are 2 ways to communicate with the Galileo infrastructure: [ZMQ](http://zeromq.org/) and HTTP.

If the programming language provides access to native bindings for ZMQ, and the agent can declare a direct dependency for ZMQ lib (through package managers), then ZMQ is preferred. Otherwise use HTTP.

###### Examples

- :+1: Java: [jeromq](https://github.com/zeromq/jeromq)
- :+1: Python: [pyzmq](https://github.com/zeromq/pyzmq)
- :-1: Node.js: [zeromq.node](https://github.com/JustinTulloss/zeromq.node)

## Agent Configuration

The Agent should expose the following optional configurations to the user, with fallback to default values when none are provided:

| name                      | type     | description                                                                  | default                      |
| ------------------------- | -------- | ---------------------------------------------------------------------------- | ---------------------------- |
| **`HOST`**                | [`RFC 3986 Host`][rfc3986-host] | DNS Host / IP Address of Galileo Socket Service       | `socket.galileo.mashape.com` |
| **`PORT`**                | [`RFC 3986 Port`][rfc3986-port] | Port for Galileo Socket Service                       | `5500`                       |
| **`FAIL_LOG`**            | [`RFC 3986 Path`][rfc3986-path] | file system path, storage location for failed message | `/dev/null`                  |
| **`RETRY_COUNT`**         | `integer`                       | Number of retries in case of failures                 | `0`                          |

### ZMQ

#### Options

In addition to the [Agent Configuration](#agent-configuration), the Agent should also expose the following optional configurations to ZMQ users, with fallback to default values when none are provided:

| name                      | description                                                                   | default                      |
| ------------------------- | ----------------------------------------------------------------------------- | ---------------------------- |
| **`CONNECTION_TIMEOUT`**  | Timeout in minutes before recycling the connection to Galileo Socket Service  | `5`                          |
| **`FLUSH_TIMEOUT`**       | Timeout in seconds before abort sending current message                       | `5`                          |

#### Message Format

```
alf_1.0.0 {ALF_OBJECT}
```

where `{ALF_OBJECT}` is a `UTF-8` encoded String of the [ALF JSON Object][api-log-format]

#### Behavior

The agent MUST:

- [ ] Connect to `tcp://HOST:PORT` in `PUSH` mode
- [ ] Send ALFs automatically on the socket as they arrive
- [ ] On an Interval of `CONNECTION_TIMEOUT`, replace the socket with a fresh one.
- [ ] Manage timeouts / failures:
  - message cannot be flushed within `FLUSH_TIMEOUT`
  - message rejected from Galileo Socket Server
  - retry as per `RETRY_COUNT` or send to `FAIL_LOG`.

### HTTP

#### Options

In addition to the [Agent Configuration](#agent-configuration), the Agent should also expose the following optional configurations to ZMQ users, with fallback to default values when none are provided:

| name                      | unit    | description                                         | default                      |
| ------------------------- | ------- | --------------------------------------------------- | ---------------------------- |
| **`CONNECTION_TIMEOUT`**  | seconds | timeout before aborting the current HTTP connection | `30`                         |
| **`FLUSH_TIMEOUT`**       | seconds | timeout before flushing the current queue           | `5`                          |
| **`QUEUE_SIZE`**          | integer | total queue size                                    | `100`                        |

The Galileo Socket Service provides two methods to batch data.

1. Group multiple ALF Objects into an array: `[{ALF_OBJECT}, {ALF_OBJECT}]`.
2. Construct a single ALF Object with multiple [HAR][har-spec] [entries](http://www.softwareishard.com/blog/har-12-spec/#entries).

###### Note:

The latter batching method requires sharing the same ALF log entry, meaning all entries share the same ALF [root properties](https://github.com/Mashape/api-log-format/blob/master/versions/1.0.0.md#properties) *(e.g. `version`, `clientIPAddress`)*. This introduces an implicit requirement to group by `clientIPAddress`, and therefore is a complex scenario that is best avoided.

#### Behavior

The agent MUST:

- [ ] Group the data into before sending as batches
- If the ALFs are batched into an array (Option 1), flush the array to `http://socket.analytics.mashape.com/1.0.0/batch` **every 2 seconds AND ALSO every time the array length reaches 1000 elements**. The flush interval (2 seconds) and the queue length (1000) should be configurable by the user.
- If Option 2 is chosen, it means there's only one ALF to send (even though it might have more than one entry). Send it to `http://socket.analytics.mashape.com/1.0.0/single`.  Make sure it doesn't exceed 500 MB.
- Monitor the response of the server. If it isn't `200 OK`, then save it to the disk and save the error somewhere (stderr or the error logs, for example).

#### Response
- Successful, 200 would be retured with body (Valid ALFs: Saved ALFs/total ALFs) for request containing all valid ALFs and if request not breaching rate limit
- Partial Sucess, 207 would be retured with body (Valid ALFs: Saved ALFs/total ALFs) for request containing partial valid ALFs or if request breaching rate limit
- Failure, 400, if request size is more than 500 mb or if you enter a type that doesn’t exist ( `'alf_1.0.0', 'batch_alf_1.0.0'`) 

## Capturing Data

The Agent will use [API Log Format](https://github.com/Mashape/api-log-format) *(ALF)* to create log entries.

Most of the fields in ALF are self-explanatory. Check out the [HAR spec][har-spec] for additional information.

The following rules are beyond the scope of HAR and **MUST** be applied to all agents:

### `clientIPAddress`

- [ ] Parse Headers to obtain true client IP *(see [reference table](#client-ip-headers) below)*
- [ ] fallback to capturing the raw socket client IP

###### Client IP Headers

| header                | priority | description                                                            |
| --------------------- | -------- | ---------------------------------------------------------------------- |
| `Forwarded`           | 1        | [RFC 7239](https://tools.ietf.org/html/rfc7239) Standard               |
| `X-Real-IP`           | 2        | mostly used in [proxies](http://bit.ly/1Jj9yu6)                        |
| `X-Forwarded-For`     | 3        | common, [non-standard](https://en.wikipedia.org/wiki/X-Forwarded-For)  |
| `Fastly-Client-IP`    | 4        | [Fastly](http://bit.ly/1Rm8pdA)                                        |
| `CF-Connecting-IP`    | 4        | [CloudFlare](http://bit.ly/22ZZ53c)                                    |
| `X-Cluster-Client-IP` | 4        | [Rackspace](http://bit.ly/1KdMKNc), X-Ray                              |
| `Z-Forwarded-For`     | 5        | Z Scaler                                                               |
| `WL-Proxy-Client-IP`  | 5        | Oracle Web Logic                                                       |
| `Proxy-Client-IP`     | 5        | no-references                                                          |

### Request

- [ ] Agent should attempt to get the **RAW** Request as early as possible *(as soon as the last byte is received and before application business logic)*
  - Should be triggered prior to any processing *(decompression, modification, normalization, etc...)* by the application or application framework
  - In many languages *(especially: `PHP`, `Node.js`)* reading the input stream is awarded to the **first listener**, the stream is then flushed, thus blocking any following listeners from reading
  - This is to ensure all original headers and body state are captured properly
- [ ] Agents cannot obstruct the application natural flow
  - should not prevent the application from getting the request data for its own processing
  - this is likely framework dependent, or in the case of `PHP`, `Node.js`, the input stream can only be read once, and thus the agent must re-institute the stream so the application logic can continue un-interrupted.

### Response

- The Agent should only attempt to process the response object at the time the application is ready to send it. *(as soon as the last byte is ready to send)*.
  - Just as with the [request](#request) scenario, this is to ensure all possible headers and final modifications to the response objects are captured.
  - some languages *(such as `PHP`)* would terminate as soon as as the last byte is sent, it is important to trigger the agent logic, before sending the response is started, but not before constructing the response is completed.

### Body Size

- [ ] Agent should attempt to calculate the request & response body size manually *(in bytes)* or Otherwise, rely on the `Content-Length` header when available.
  - this is true regardless whether bodies are flagged to be sent or not.

### Headers

- [ ] When not readily available, the Agent should attempt to calculate headers sizes: *`ALF.har.log.entries[].request.headersSize`, `ALF.har.log.entries[].response.headersSize`*
  - this can be achieved by reconstructing the [HTTP Message](http://httpwg.github.io/specs/rfc7230.html#http.message) from the start of the HTTP request/response message until (and including) the double `CRLF` before the body.
  - This means calculating the length of the headers as they appeared "on the wire", including the colon, space and `CRLF` between headers.

### Timings

There are 3 mandatory fields that require to be manually calculated (where applicable):

- [ ] `ALF.har.log.entries[].timings.send`: duration in milliseconds, between the first byte of request received and *processing time*
- [ ] `ALF.har.log.entries[].timings.wait`: duration in milliseconds, between the *processing time* and the first byte of response sent time
- [ ] `ALF.har.log.entries[].timings.receive`: duration in milliseconds, between the first byte of the response time and the last byte sent time

###### Note:

The term *"processing time"* can refers to different meanings given the agent type:

1. Proxy Agents: sending the last byte to the upstream service
2. Native Agents: the start of the application business logic

### Bodies

When the request/response bodies are captured, they must be base64-encoded:

###### Example
For request bodies: `request.postData = {"encoding": "base64", "text": "BASE64_BODY"}`
For response bodies: `response.content = {"encoding": "base64", "text": "BASE64_BODY" }`

[api-log-format]: https://github.com/Mashape/api-log-format
[galileo]: https://getgalileo.io/
[har-spec]: http://www.softwareishard.com/blog/har-12-spec/
[rfc3986-host]: https://tools.ietf.org/html/rfc3986#section-3.2.2
[rfc3986-port]: https://tools.ietf.org/html/rfc3986#section-3.2.3
[rfc3986-path]: https://tools.ietf.org/html/rfc3986#section-3.3
