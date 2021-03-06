# Server API documentation

The Eos server is written in Go, a modern, free, compiled language developed by Google. As a result of this, however, there are specific quirks to the Eos server's handling of certain API calls, owing to the precise nature of Go's `structure` handling.

The Eos server actually operates 2 (in a non-production environment) to 3 (production) servers, specifically:

In a non-production environment:

- A HTTP server operating on port 80 to serve the PWA files (in the `/webclient` folder)
- A WebSocket server operating on a specified port, serving as the back-end of the Eos service.

In a production environment:

- A HTTP server operating on port 80 to redirect requests to the HTTPS server
- A HTTPS server (with valid TLS certificate) operating on port 443 to serve PWA files in an encrypted fashion
- A WebSocket server secured with TLS operating on a specified port, serving as the back-end of the Eos service.

API calls should be sent via a WebSocket packet to the WebSocket server, from the HTTP(S) server operating **on the same host** (for security purposes).

## Typical user flow

```mermaid
graph TD;
    Load(Loading) --> |connecion established| Login{Login screen};
    Login --> |login| Log[Mood logging];
    Login --> |sign up| Email[Provide email address];
    Email --> |email received| Verify[Verify email address and supply password];
    Verify --> Log;
    Log --> Chat[Chat service];
    Chat --> End(End);
```

### Network requests

```mermaid
sequenceDiagram
    Client --> Server: Inititate websocket connection;
    Server ->> Client: version;
    alt New account
        Client ->> Server: signup;
        Server ->> Client: signup;
        Note over Client, Server: Email with verification <br />token sent to client;
        opt Client decision
            Client ->> Server: verifyEmail;
            Server ->> Client: verifyEmail;
        end
        Client ->> Server: createAccount;
    else Existing user
        Client ->> Server: login;
    end
    Server ->> Client: login;
    Client ->> Server: mood;
    Client ->> Server: comment;
    Client ->> Server: chat:start;
    alt User banned
        Server ->> Client: chat:banned;
    else
        Server ->> Client: chat:ready;
    end
    loop Each message
        Client ->> Server: chat:send;
        Server ->> PerspectiveAPI: (Test message against AI);
        PerspectiveAPI ->> Server: (ATTACK_ON_AUTHOR rating);
        alt rating is good || API failure
            Server ->> Client: chat:message;
        else rating is bad
            Server ->> Client: chat:rejected;
            opt User confirms
                Client ->> Server: chat:verify;
                Server ->> Client: chat:message;
            end
        end
    end
    opt Reported by user
        Client ->> Server: chat:report;
    end
    Client ->> Server: chat:close;
```

## API Methods

### Connection

Connection to the WebSocket server follows standard WebSocket specification. For example, a JavaScript connection to the WebSocket server (`new WebSocket('address here')`).

Upon successfully connecting to the WebSocket server, the server **must** respond with the following data packet:

```javascript
{
    "type": "version",
    "data": "" // version number of the server
}
```

### Logging in (login)

```javascript
{
    "type": "login",
    "emailAddress": "", // email address goes here
    "password": "", // password goes here
}
```

Once the login has been processed, the server will return the following:

```javascript
{
    "type": "login",
    "flag": true, // true if user authentication success, false otherwise.
    "user": {} // User object
}
```

**Notice:** This method used to create a new account if the requested one did not already exist. This has changed in `v2.0:staging-rc3`, instead breaking this functionality into its own API method.

[Documentation on the User object](user.md)

This method **MUST** be called before calling any subsequent method within the server's functionality; non-logged-in method calls will be discarded by the server automatically.

### Resetting a password (resetPassword)

```javascript
{
    "type": "resetPassword",
    "emailAddress": "", // email address goes here
}
```

If a login token was successfully sent to the user's email address, then the server will respond:

```javascript
{
    "type": "resetPassword",
    "flag": true
}
```

Otherwise, `flag` will be `false`. The client is not expected to do anything else following this.

### Creating a new account (signup)

```javascript
{
    "type": "signup",
    "emailAddress": "", // email address goes here
}
```

This API method is a *flow*, meaning that it does not operate independantly and makes use of additional methods for the same purpose.

If there is no existing account with the email address supplied, the server will respond:

```javascript
{
    "type": "signup",
    "flag": true, // if an account already exists, this will be false
}
```

If `flag` is `true`, this represents that the server (through SendGrid) has issued a verification email to the email address supplied. To continue, the user should check their emails.

### Verifying email address (verifyEmail)

```javascript
{
    "type": "verifyEmail",
    "data": "", // the verification token received via email should go here
}
```

After the user follows the instructions in the email to verify their email, they should either (have a verification token) or (have clicked a link handling this for them).

The server should respond to this request with:

```javascript
{
    "type": "verifyEmail",
    "data": "", // the email address associated with the verification token
}
```

This supplied email address should be used to pre-populate the user's email address field in the signup form (**DO NOT ALLOW THE USER TO CHANGE THIS MANUALLY**).

### Completing the signup process (createAccount)

```javascript
{
    "type": "createAccount",
    "data": "", // verification token supplied via email earlier
    "password": "", // the user's chosen password
}
```

The server will automatically log the user in to the newly created account:

```javascript
{
    "type": "login",
    "flag": true, // true if successful, false if the email address already has an account
    "user": {}, // User object -- this is a blank User object if the account creation was unsuccessful
}
```

### Submitting new Mood data (mood)

```javascript
{
    "type": "mood",
    "day": 0-6, // day of the week, 0 (Sunday) to 6 (Saturday)
    "month": "MM", // month of the year, 0 (January) to 11 (December)
    "year": "yyyy", // current year in UTC time
    "mood": -2-2, // relative mood, -2 to 2
}
```

## Submitting a new comment for a mood (comment)

```javascript
{
    "type": "comment",
    "mood": -1-1, // integer, either negative (-1), neutral (0), or positive (1)
    "data": "" // string containing comment data
}
```

## Updating user details (details)

```javascript
{
    "type": "details",
    "emailAddress": "", // new email address
    "password": "", // new password
    "data": "" // new name
}
```

Any values which remain unchanged **MUST** still be sent, but should be sent as an empty string (`""`), to indicate no change is needed.

The only details required for an account is the **email address and password**, the user's name is not required and will default to "friend".

**If a new email address is supplied, there is an additional email flow, beginning with the following server response:**

```javascript
{
    "type": "changeEmailVerification"
}
```

This indicates that a verification token has been sent to the new email address, and that the user should look there. The UI should respond by asking the user for this token.

## Verifying an email change (changeEmail)

```javascript
{
    "type": "changeEmail",
    "data": "" // the authorisation token should go here
}
```

**If there is a pending request matching this auth token**, the server will respond:

```javascript
{
    "type": "changeEmail",
    "emailAddress": "" // the new email address which is now set
}
```

Else, it will disregard the request.


## Sending a deletion request (delete)

```javascript
{
    "type": "delete"
}
```

This action, depending on how the server's setup, will usually result in instantaneous data deletion. This should remove the user file, and purge the in-memory user data (nullify the variable). **Report data will be unaffected by this request, accounting for the fact that report data may be critical to a law enforcement investigation.**

Handling of this request should begin immediately; finality warnings should be addressed by the client.

## Chat API: Start a new chat (chat:start)

This method must be called, and a ChatID must be provided, before any other chat API methods can be called.

```javascript
{
    "type": "chat:start"
}
```

If the user is banned from online chat functionality, processing of this command will be ignored and the server will respond with a simple notice of the form:

```javascript
{
    "type": "chat:banned"
}
```
Otherwise, handling of setting up chat details, and connecting users together (usually via a first-come-first-serve queue system) is handled by the server. If the user is awaiting a partner, the server should send this as response:

```javascript
{
    "type": "chat:ready",
    "flag": false
}
```

Otherwise, if a chat has been initiated, this should be sent:

```javascript
{
    "type": "chat:ready",
    "flag": true,
    "cid": "" // chat ID goes here (see below)
}
```

The `cid` field should contain a string **unique chat identification**, which SHOULD NOT be an incremental counter. Treat this like a user ID; this will be used by the clients to address the chat which they are in to the server. If an invalid ChatID is sent, just ignore the request; it's malformed and likely a spoof attack.

**UPON CHAT TERMINATION BY EITHER PARTY, THE SERVER WILL SEND A `"chat:closed"` NOTICE TO THE OTHER PARTY.**

## Chat API: Send a message (chat:send)

```javascript
{
    "type": "chat:send",
    "data": "", // Message content goes here, in string format
    "cid": "" // the ChatID provided by the server goes here
}
```

This will be handled by the server and sent to the other party of the conversation. If the server has a provided Google Cloud API Key, then the message will first be sent through the Perspective API Neural Network. A toxicity probability above 0.9 will block the message (see below), whereas a lower probability **or an unsuccessful (non-200 return value) request** will allow the message to go straight through.

If a message is rejected, the server will send the client a JSON request of the following format:

```javascript
{
    "type": "chat:rejected",
    "mid": "" // incremental message ID goes here relating to the message's index position in the RAM log.
}
```

To authorise a message, see the `verify` Chat API method below.

Upon a successful message send, the sender will receive the following JSON message from the server:

```javascript
{
    "type": "chat:message",
    "flag": false,
    "data": "" // chat message
}
```

Whereas the recipient will receive:

```javascript
{
    "type": "chat:message",
    "flag": true,
    "data": "" // chat message
}
```

## Chat API: Verify a rejected message (chat:verify)

Message rejections are not final, acknowledging the unreliability of AI. Thus, if a message is rejected, the client should ask the user. If the user insists, this method can be used to 'force' the message through.

```javascript
{
    "type": "chat:verify",
    "cid": "", // the chat id goes here
    "mid": "" // the message id goes here
}
```

Sending will then occur as if the message had been allowed through the filter automatically.

## Chat API: Report a chat (chat:report)

```javascript
{
    "type": "chat:report",
    "cid": "" // the chat id to report
}
```

## Chat API: Close a chat (chat:close)

```javascript
{
    "type": "chat:close"
}
```

Different servers may handle chat reports differently. This is not addressed by this specification.

## Admin API: Access a report (admin:access)

**ADMIN API METHODS WILL ONLY RETURN A RESPONSE IF THE LOGGED-IN EOS ACCOUNT'S "ADMIN" PARAMETER IS SET TO TRUE.**

```javascript
{
    "type": "admin:access",
    "cid": "" // ID of the chatlog to access (aka chat ID or report ID)
}
```

=> This will return the following response:
```javascript
{
    "type": "admin:chatlog",
    "chatlog": [
        {
            "aiDecision": true, // true if the message was delivered, false otherwise. Messages may have two objects here, one true and one false, if a rejected message was forced through. This aids moderation.
            "sender": "", // user ID of the message's sender.
            "message": "" // message content.
        }, ...
    ]
}
```

## Admin API: Decide a report (admin:decision)

**ADMIN API METHODS WILL ONLY RETURN A RESPONSE IF THE LOGGED-IN EOS ACCOUNT'S "ADMIN" PARAMETER IS SET TO TRUE.**

```javascript
{
    "type": "admin:decision",
    "cid": "", // ID of the chatlog you are deciding on.
    "data": "" // userID of the offending user to ban. if not applicable, leave as a blank string.
}
```

Upon a decision being made, the chatlog should therefore be deleted immediately. **It is important to note that, if a chatlog contains illegal content or is part of an ongoing investigation, the matter should be escalated to the server admin and a copy of all relevant data should be made by them. The report should NOT be decided upon. This should be achieved via `admin:flag`.**

Upon completion, the server will provide this response:

```javascript
{
    "type": "admin:success"
}
```

## Admin API: Flag a report for escalation (admin:flag)

**ADMIN API METHODS WILL ONLY RETURN A RESPONSE IF THE LOGGED-IN EOS ACCOUNT'S "ADMIN" PARAMETER IS SET TO TRUE.**

```javascript
{
    "type": "admin:flag",
    "cid": "" // ID of the chatlog you are deciding on.
}
```

Upon completion, the server will provide this response:

```javascript
{
    "type": "admin:success"
}
```