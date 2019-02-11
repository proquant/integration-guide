# Introduction
Welcome to the ProQuant Integration guide!

This guide assumes the reader is a developer working at a company looking to integrate their product with ProQuant. We strongly encourage you to download the ProQuant app and get familiar with it as you're reading this guide.

We'll be covering things from the perspective of a single user. For the purpose of this guide, let's assume this user already has an account both in your system and in ProQuant.

Let's look at how to connect these accounts, so that the user's ProQuant strategy signals translate into actual executed trades in your system.


# Overview
Every time a ProQuant strategy generates a signal, a simulated position is opened/closed in ProQuant's system, but no actual trading is happening. As we're not an execution engine, all we can do is package certain Events happening in our system and send them to whatever Integrations the user has turned on for their strategy.

The goal of the guide is to show how to:
1. Create an integration request from your system to ProQuant's, for a user's account
2. Handle received events
3. Respond to received events

# Requirements
The only requirement is that you provide us with a reachable URL for us to send HTTP POST requests to. These requests are the mechanism for delivering events to you, which is the essence of how the communication layer of a ProQuant integration works.

# What's an Integration?
Here's what an Integration looks like:
```javascript
    {
        userEmail, // The e-mail the user created their ProQuant account with.
        name, // used for visualisation
        description, // used for visualisation
        imageUrl, // used for visualisation
        webhook: {
            url, // where we send Events to
            headers // the headers we include in every POST request we make to the url above
        }
    }
```

All of these properties above are defined by you, at the moment of creating the integration.

# Creating an integration
A simple HTTP POST request to our API will create an *integration request* which will appear in ProQuant's mobile app and must be approved by the user. Once approved, the user may *enable* this integration for any of their running strategies. A running strategy will forward signals to all of its *enabled* integrations.

An *integration* in ProQuant (actually called a *Connected Account* in the app), includes a *name*, *description*, and *avatar*. However, the most important piece of information in an *integration* is the  *webhook* property. It contains a *url* and some *headers*, and we use these ... TODO.

The request for creating an *integration request*:

`POST https://www.proquant.com/api/integrations/v1/integrations`
```javascript
{
    name, // The "title" of the integration, used for visualisation purposes only. We suggest setting it to the e-mail of the user in your system, or some other identifier (e.g. "user1@example.com")
    description, // The "subtitle" of the integration. Your product name would fit best here (e.g. "Binance") 
    imageUrl, // This is used as an "Avatar" for the integration. We suggest using your company/product logo here.
    userEmail, // The e-mail the user created their ProQuant account with.
    webhook: {
        url, // We'll be sending events this way. Could be unique for each user, depending on your identification & authentication strategy.
        headers // Optional. Anything you put in here will be included, as headers, in the POST requests we make to the webhook url.
    }
}
```

Once the user has *approved* the integration from inside ProQuant's mobile app, the setup is done and they may enable this integration for one or more of their running strategies.

At this point, Events will start arriving at the `webhook.url` address.

# Handling Events
Before we talk about the Events themselves, let's first take a look at *versioning*.

### Versioning
Every request we send to `webhook.url` includes a single event, formatted in all currently supported version formats, alongside meta information about the status of every single version. It is your responsibility, as a developer, to track when the version you're using is going to be deprecated, and upgrade to a newer one before that happens.

Here's how this looks in practice:

POST `webhook.url` Request payload
```
{
    "v1": {
        "deprecation": "2020-01-01T00:00:00.000Z",
        "event": {
            /* Event in version 1 format */
        }
    },
    "v2": {
        "deprecation": null,
        "event": {
            /* Event in version 2 format */
        }
    }
}
```

*DISCLAIMER: The above is just an example, you can find our currently supported versions and their deprecation status further in the guide*

In the example above, we support 2 versions - v1 and v2. The deprecation parameter signifies the status of the version - if deprecation is null, that means this version is the latest and is actively supported and maintained by us. However, deprecation could be a date in the ISO format - in that case, this version is no longer supported by us, and, after the specified date has passed, data for this version will no longer be included in any requests we send your way. It is your responsibility to check for the value of the deprecation parameter for the version of our API you've currently implemented, and make an effort to implement the latest version so as not to break your client's integrations.

New versions will be released when breaking, non-backwards-compatible changes are introduced. This should happen rarely, if at all, and support for old versions will be continued long enough to allow you to upgrade without rush.

### Currently supported versions
| Version | Status | Deprecated on |
| ------ | ------ | ------ |
| v1 | Active | - |

### v1 Event Structure
All Events have common structure and properties, so we'll begin by taking a look at that.

Basic Event structure (JSON):
```
{
    "timestamp": 1549623690730, // UNIX-timestamp in milliseconds
    "type": "SIGNAL_OPEN",
    "integration": { ... } // Integration
    "data": { ... } // Specific for each event type
}
```

### v1 Event types and data

| Type | What happened | data |
| ------ | ------ | ------ |
| INTEGRATION_APPROVED | The user approved the integration request | `null` |
| INTEGRATION_REJECTED | The user rejected the integration request | `null` |
| INTEGRATION_DELETED | The user deleted a previously approved integration | `null` |
| STRATEGY_ENABLED | The user enabled the integration for one of their strategies. Enabling an integration means that whenever the strategy produces signals, they will be sent to that integration as Events. | [Strategy Event Data](#v1-strategy-event-data) |
| STRATEGY_DISABLED | The user disabled the integration for one of their strategies. Disabling an integration means that the integration will stop receiving signal Events from this strategy | [Strategy Event Data](#v1-strategy-event-data)
| SIGNAL_OPEN | One of the user's running strategies produced a signal to open a position. | [Signal Event Data](#v1-signal-event-data) |
| SIGNAL_CLOSE | One of the user's running strategies produced a signal to close its opened position. | [Signal Event Data](#v1-signal-event-data)
| SIGNAL_REOPEN | One of the user's running strategies produced a signal to close its opened position and then immediately open a new one in the opposite direction (the `direction` property in the data signifies the *new* position direction). | [Signal Event Data](#v1-signal-event-data)

##### v1 Strategy Event Data
```
{
   id, // unique identifier of the strategy
   name // strategy name, given by the user
}
```

##### v1 Signal Event Data
```
{
    direction: BUY|SELL,
    ticker,
    quantity, // Expressed in units
    strategy: {
        id // The unique id of the strategy that produced the signal
        name // Name of the strategy that produced the signal
    },
    proquantPositionId // Unique id of the corresponding position in ProQuant
}
```

### v1 Request Example
Here's an example request payload you may receive at `webhook.url`, taking everything we've covered so far into consideration:

```
{
    "v1": {
        "deprecation": null,
        "event": {
            "timestamp": 1549623690730,
            "type": "SIGNAL_OPEN",
            "integration": {
                "name": "user@example.com",
                "description": "Foo Bar FX",
                "imageUrl": "https://foo.bar/logo.png",
                "webhook": {
                    "url": "https://foo.bar/api/proquant",
                    "headers": {
                        "x-foo-bar-api-token": "baz"
                    }
                }
            },
            "data": {
                "direction": "BUY",
                "ticker": "XAUUSD",
                "quantity": 10,
                "strategy": {
                    "id": "6bbc583bf6d70605bbbc892f66367ab7",
                    "name": "Gold High-Risk"
                },
                "proquantPositionId": "960fe1a63d264d0f4e19a0ebf385497e"
            }
        }
    }
}
```

We've covered what Events are and what type of Events you may receive from ProQuant. All that's left is to see how to respond to these HTTP requests.

# Responding to Events
If all goes well and the Event has been handled successfully, *or is irrelevant to you and you did not consume/handle it at all*, simply respond with status code 200 and an empty body.

In case of an error, ProQuant clients will receive a push notification informing them that something went wrong with the integration. What this notification will be, depends on the way you respond to the request. We understand, handle and support a predefined set of responses, which translate into well-designed errors shown to clients. If you respond with a status code >= 400 and we do not support that type of response, we will fall back to a Generic error notification.

Implementing and returning these specific Event Errors is optional, but recommended, as it would give clients a better overview of what went wrong with their integration.

Here are the requirements for an error response to be correctly parsed by us:

1. Status code must be >= `400`
2. Content type must be `application/json`
3. The response headers must include the header `x-proquant-error: 1`
4. The response body must include a key `type` with one of the following values:
a. InsufficientFunds
b. QuantityTooLow
c. QuantityTooHigh
d. InvalidQuantityPrecision
e. UnsupportedInstrument
f. PositionAlreadyExists
g. MarketClosed
h. PositionNotFound
i. InvalidCredentials

 
If you have a type of error in mind that we do not support, feel free to contact us, or open an issue, and we'll look into adding support for it.

# Thanks!

That's it for the guide, thank you for reading! If you have any questions, feel free to either open an issue here, or send us a mail at support@proquant.com
