# Mashape Analytics Agent Spec

Agents are libraries that can act as a middleweare / helper utility in reading request & response objects at the start of the request before processing, and at the end of the response before sending, in order to construct [ALF objects](https://github.com/Mashape/api-log-format) which can be used with the [Mashape Analytics](ttps://www.apianalytics.com/) service.

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

- **Request Processing**
  The agent should *-when possible-* process the RAW request object prior to any processing by application or framework used by application. this is to ensure all original headers and bodies are captured properly.

- **Response Processing**
  The agent should only attempt to process the response object at the time the application is ready to send it, just as with the request scenario, this is to ensure all possible headers and final modifications to the response objects are captured.
