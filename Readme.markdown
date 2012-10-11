# CrashLog API Documentation

CrashLog consists of two APIs, the collection API known as `stdin` and the query API.

## stdin API

CrashLog accepts events over HTTP to `stdin.crashlog.io`. Events are authenticated with HMAC (see below) using an `api_key` and `secret`. Upon successfully authenticating the `payload` will be consumed.

### HMAC authenticated requests

All payloads must be sent along with a HMAC `Authorization` header, this should be structured like this:

`Authorization: CrashLog <api-token>:<HMAC Signature>`

In a nutshell this signature is constructed from the headers you are sending with the payload, example:

```
hmac_string     = HTTP-Verb    + "\n" +
                  Content-Type + "\n" +
                  Content-MD5  + "\n" +
                  Date         + "\n" +
                  request-uri;

"Authorization: CrashLog " + API-Token + ":"  + base64(hmac-sha1(hmac_string))
```

The service ID `CrashLog` is to be used for authenticating against the production App at `http://crashlog.io` and `Staging` is used for authenticating and sending events to the staging App.

To see exactly how this is constructed take a look at the source of [crashlog-auth-hmac](https://github.com/crashlog/auth-hmac) gem and [crashlog](https://github.com/crashlog/crashlog).

The `api_key` and `secret` pair can be found on the settings page of your project. You should never send the `secret` down the wire and only expose it to your application for obvious security reasons.

If these credentials become compromised you can revoke then within the settings page as well.

### Endpoints

CrashLog supports two endpoints, one for announcing application launch or deployment, and one for consuming events.

### Announce

```
Host: stdin.crashlog.io

POST /announce
```

Announce simply returns the application name so you can check that your application is configured correctly. In the Ruby gem this will write a line to your production log file with the name of the project.

Typically you would send the `hostname` and a `timestamp` in this payload and it will be displayed in the project overview page.

### Events

```
Host: stdin.crashlog.io

POST /events
```

### Event payload

You can see the event payload example below, but here are the basics:

The payload can contain multiple interfaces, of which only 2 are required:

- notifier (required)
- event (required)
- backtrace
- environment
- context
- data

### Notification response

A successfully lodged exception will return a hash like:

`"{"location_id": "<128bit UUID>"}`

`location_id` can be used to generate a URL to link directly to the exception or fetch information about it from the CrashLog public API.

**Generating URLs**

The standard location for locating exceptions within the CrashLog UI is `http://crashlog.io/locate/:location_id`

This will either return `404` if the exception has not yet been loaded (< T+1 second) or `301` with a redirection to the actual error page.

# Example Payload Schema

```json
{
  "payload" : {

    // Interface: Notifier
    "notifier" : {
      "name": "crashlog-nodejs",
      "version": "1.0.0"
    },

    // Interface: Event
    "event" : {
      "message" : "Some one line error description",
      "type": "RuntimeError",
      "timestamp" : "2012-08-03T12:34:19+10:00"
    }

    // Interface: Backtrace
    "backtrace" : [
      {
        // Required
        "file": "somewhere.js",
        "number": 14, // Line number
        "method": "raiseTheRoof",

        // Optional
        "context_line": "source code for affected line",
        "pre_context" : ["line -1 before affected line", "line -2 before affected line", "line ..."],
        "post_context" : ["line +1 after affected line", "line ..."]
      },

      {
        // Required
        "file": "somewhere_else.js",
        "number": 192, // Line number
        "method": "setup",

        // Optional
        "context_line": "source code for affected line",
        "pre_context" : ["line -1 before affected line", "line -2 before affected line", "line ..."],
        "post_context" : ["line +1 after affected line", "line ..."]
      }
    ],

    // Interface: Environment
    // Optional
    //
    // Any key value pairs you want to be able to view later on
    //
    // At the moment this is unstructured, this will likely change in the furure.
    "environment": {
      "project_root" : "/home/deploy/my_app/current",
      "platform" : "Node.js 0.6.x",
      "name": "production",
      "system": {
        "distribution": "Ubuntu 12.04 LTS"
      },
      "browser": {
        "ident": "Usual browser tag"
      },

      "plugins": []

      // TODO: Insert more attributes here

    },

    // Interface: Context
    // Optional
    // Any contextual information relating to the cause of this error
    // This is typically a serialisation of the current user
    //
    // The attributes can be used to correlate errors
    "context" : {
      "stage" : "production",
      "hostname": "app01.myhostname.com",
      "current_user": {
        "id": 1,
        "email": "user@example.com"
      },

      "controller": {
        "name": "usersController",
        "action": "index"
      }
    }
  }
}
```
