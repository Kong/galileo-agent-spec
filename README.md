# Mashape Analytics Agent Spec

Agents are libraries that can act as a middleweare / helper utility in reading request & response objects at the start of the request before processing, and at the end of the response before sending, in order to construct [ALF objects](https://github.com/Mashape/api-log-format) which can be used with the [Galileo](https://www.apianalytics.com/) service.

## Connectivity

There are 2 ways to communicate with the Galileo infrastructure: ZMQ and HTTP.

If the agent can be installed WITHOUT having to install another system-wide library for ZMQ, then use ZMQ. Otherwise use HTTP. For example, Java offers a complete implementation of ZMQ in pure Java, which means that it'll be installed automatically by maven and it doesn't need any `sudo apt-get install`.

### ZMQ

```js
var zmq = require('zmq');
var socket = zmq.socket('push');
socket.connect('tcp://socket.analytics.mashape.com:5500');

// The space between the type and the payload is mandatory
socket.send('alf_1.0.0 ' + JSON.stringify(alfObject));
```

The agent MUST:

- Connect to `tcp://socket.analytics.mashape.com:5500` in PUSH mode
- Send ALFs automatically on the socket as they arrive
- Listen to the flush event. If an ALF cannot be flushed within 20 seconds, the socket must be destroyed and a new one created. Don't lose any ALFs in the process.

### HTTP

In the ALF format, there's 2 ways to batch data. The first is to create an array of ALF objects `[{ALF}, {ALF}]`. The second is to make one ALF have multiple entries. Option 2 means that they'll share the ALF header, so it's only acceptable if they share the same `serviceToken` and `environment`, for example.

The agent MUST:

- Bulk the data, see above for the 2 ways to do it
- If the ALFs are batched into an array (Option 1), flush the array to `http://socket.analytics.mashape.com/1.0.0/batch` **every 2 seconds AND every time the array length reaches 1000`. Those settings should be adjustable by the user.
- If Option 2 is chosen, it means there's only one ALF to send (even though it might have more than one entry). Send it to `http://socket.analytics.mashape.com/1.0.0/single`.
- Monitor the response of the server. If it isn't `200 OK`, then save it to the disk and save the error somewhere (stderr or the error logs, for example).


## Format

The Agent will use [API Log Format](https://github.com/Mashape/api-log-format) *(ALF)* to create log entries.

Most of the fields in ALF are self-explanatory. Check out the [HAR spec](http://www.softwareishard.com/blog/har-12-spec/) for additional information. The most common gotchas and questions are answered here:

### clientIPAddress

Use, in order of priority: `Forwarded` header (RFC7239), `X-Real-IP` header, `X-Forwarded-For` header, raw socket IP.

### Request

- The Agent should attempt to get the RAW Request as early as possible in the application lifecycle, before any processing *(decompression, modification, normalization, etc...)* by the application or application framework
  - This is to ensure all original headers and body are captured properly
- In many languages *(especially: PHP, Node.js)* reading the input stream is awarded to the first listener, the stream is then flushed, thus blocking any following listeners from reading
  - This is important to keep in mind for Agents attempting to parse the request bodies so not to prevent the user's application from getting the request data for its own processing

### Response

- The Agent should only attempt to process the response object at the time the application is ready to send it
  - Just as with the request scenario, this is to ensure all possible headers and final modifications to the response objects are captured.

### Body Size

- If the bodies *were* successfully captured, the Agent should attempt to calculate the request & response body size manually (in bytes). Otherwise, rely on the Content-Length header (this will fail for Chunked connections, for those the body needs to be counted manually).

### Headers

- When not readily available, the Agent should attempt to calculate headers sizes: *`ALF.har.log.entries[].request,headersSize`, `ALF.har.log.entries[].response.headersSize`* by reconstructing the [HTTP Message](http://httpwg.github.io/specs/rfc7230.html#http.message) from the start of the HTTP request/response message until (and including) the double CRLF before the body. This means calculating the length of the headers as they appeared "on the wire", including the colon, space and CRLF between headers.

### Timings

There are 3 mandatory fields:

- send: How long did it take between the request was received and the time it was forwarded to the service that can process it?
- wait: How long did it take between the time the request was forwarded to the service and the time the service STARTED sending back a response?
- receive: How long did it take between the time the service STARTED sending the response and the time the body was fully flushed?

### Bodies

When the request/response bodies are captured, they must be base64-encoded.

For request bodies: `request.postData = {"text":BASE64_BODY}`

For response bodies: `response.content = {"encoding":"base64", "text":BASE64_BODY}`
