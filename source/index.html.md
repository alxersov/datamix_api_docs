---
title: Datamix API Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - javascript

toc_footers:
  - <a href='https://github.com/lord/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true
---

# Introduction to Datamix

Welcome to the Datamix API! The Datamix API is used to log data for all new Mixbook applications.

You can view code examples in the dark area to the right, and you can switch the programming language of the examples with the tabs in the top right.

## Logging Architecture

Mixbook is split into multiple systems and applications. Each application further has multiple environments. Together these denote a logging destination. For example, Datamix itself falls in the 'mixbook' system, 'datamix' application with 'development', 'staging' and 'production' environments.

To append logs, you'll need to create a logging session for a given destination. Logging sessions are intended to be small, logical seperations of log files. Don't share sessions across servers, processes or browsers. Sessions are cheap but have a theoritical maximum rate limit (~ 5 MB/s per session).

Each destination will have one or more API keys that allow authentication and session creation. They will expire regularly (perhaps yearly) and will have to be rotated.

## Client Logging

Since API keys should never make it to a client in any form whatsoever. You'll need to do some extra work if you want to log from a client browser. The proper steps are:

1. The browser makes an authenticated request to your server-side application for a session token. The authentication is whatever is appropriate for your browser application.
2. Your server, having the appropriate API keys, makes a request for a new session from Datamix.
3. The server responds to the browser client application with the new session token, expiration and destination information.
4. The browser application can log data directly to https://datamix.mixbook.com for the duration of the session.

## Client Libraries

We are developing client libraries to simplify the process of using Datamix. Currently only ruby (coming soon) and [JavaScript/TypeScript](https://github.com/Mixbook/mixbook_logger) libraries are available.

# General API Usage

## Authenticate a Session

> To authorize, use something like this:

```javascript
//jQuery example
data = { "key": "<your_api_key>" }

$.post("/sessions", function(data, status) {
  let system = data["system"] // The target system
  let application = data["application"] // The target application
  let environment = data["environment"] // The target environment

  let token = data['token'] // The new session token
  // The expires value is returned as an integer (milliseconds)
  let session_expires = new Date(data['expires'])
});
```

> Make sure to replace `<your_api_key>` with your API key.

Datamix uses API keys to allow access to the API. You have to get someone to share the current API key with you for the given system, application and environment.

Posting logs requires the use of the session token. Typically logging sessions are good for an hour, you should renew the session (by creating a new one) before your existing one expires.

<aside class="warning">
Never share or expose your API keys with those outside the organization. They are intended to be used server-side only.
</aside>

<aside class="notice">
API Keys will typically also have an expiration. You'll need to configure new keys before the current ones expire.
</aside>

## Sending Logs

```javascript
//jQuery example
var log_data = {
  "session": session_token,
  "messages": [
    {
      "level": "info",
      "timestamp": new Date().now(),
      "message": "This is a simple info message"
    },
    {
      "level": "warn",
      "timestamp": new Date().now(),
      "message": "This is a simple warning message"
    }
  ]
}

destination = system + "." + application + "." + environment

var callback = function(data, status) {
  let system = data["system"] // The target system
  let application = data["application"] // The target application
  let environment = data["environment"] // The target environment

  let processed = data["processed"] // The number of processed logs (e.g. 2)
  let rejected = data["rejected"] // Data about any rejected logs

  for(var i = 0; i < rejected.length; i++) {
    var index = rejected[i]["index"] // The index of the rejected log
    var message = rejected[i]["message"] // The reason for the rejection.
  }

  // There are better ways to do this, but if you have more logs:
  next_logs = get_next_logs() // implemented elsewhere ...
  // Note the session will have changed, you must use the new token
  next_logs["session"] = data["session"]
  $.post("/destinations/" + destination + "/logs", next_logs, callback);
}

$.post("/destinations/" + destination + "/logs", log_data, callback);
```

> An example JSON returned by the logs endpoint:

```json
{
  "session": "signed-session-string-here",
  "system": "batch",
  "application": "publisher",
  "environment": "staging",
  "rejected": [
    {
      "index": "3",
      "message": "Message timestamp is too old: 2017-01-01 00:12:32 UTC"
    }
  ]
}
```

Logs one or more messages to the backend. Besides being a valid JSON format, the logs messages must:

* Be logged at an acceptable log level (`debug`, `info`, `warn`, `error` or `fatal`)
* Have a timestamp in the last 10 minutes
* Have a timestamp no more than 10 seconds in the future

<aside class="notice">
Non-rejected logs were already sent to the logging backend. Retrying rejected logs will not resolve the issue since they are invalid. Resending non-rejected logs again will cause duplicate log entries.
</aside>

<aside class="warning">
The logs response will return a new session token. You **must** use this new token on the next request or the logs may be dropped. The signed session token may contain state information necessary to correctly pass messages to the backend systems. This implies a single session can only support a single request at a time. Do not make parallel requests with the same session.
</aside>

There is not currently a log message size limit. However, large log messages may be split into multiple messages on the backend before reaching their final destination. The theoretical maximum sustained log throughput is about 5 MB/s. If you are even coming close to that, you'll need to split your logs across multiple sessions.

# HTTP Request Specification

## Create a Session

`POST https://datamix.mixbook.com/sessions`

### Parameters

```json
{
  "key": <api_key:string>
}
```

Parameter | Description
--------- | -----------
key | The API key. You must pass the correct API key for your system, application and environment.

<aside class="success">
Either query parameters or JSON content are acceptable
</aside>

## Add Logs to a Session

`POST https://datamix.mixbook.com/destinations/<destination>/logs`

The `destination` is a string in the following format:

`<system>.<application>.<environment>`

### Parameters

```json
{
  "session": <session_token:string>,
  "messages": [
    {
      "level": <level:string>,
      "timestamp": <milliseconds_since_epoch:integer>,
      "message": <log_message:string>
    },
    ...
  ]
}
```

The log content should be sent to the server as a JSON string. See the message format on the right for details.
