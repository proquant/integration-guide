- [Introduction](#introduction)
- [Overview](#overview)
- [Requirements](#requirements)
- [What's an Integration](#whats-an-integration)
- [Creating an Integration](#creating-an-integration)
- [Handling Events](#handling-events)
- [Responding to Events](#responding-to-events)

# Introduction
Welcome to the ProQuant Integration guide!

This guide assumes the reader is a developer working at a company looking to integrate their product with ProQuant. We strongly encourage you to download the ProQuant app and get familiar with it as you're reading this guide.

We'll be covering things from the perspective of a single user. For the purpose of this guide, let's assume this user already has an account both in your system and in ProQuant.

Let's look at how to connect these accounts, so that the user's ProQuant strategy signals translate into actual executed trades in your system.


# Overview
Every time a ProQuant strategy generates a signal, a simulated position is opened/closed in ProQuant's system, but no actual trading is happening. As we're not an execution engine, all we can do is package certain Events happening in our system and send them to whatever Integrations the user has turned on for their strategy.

The goal of the guide is to show how to:
1. Create an Integration request
2. Handle received Events
3. Respond to received Events

# Requirements
The only requirement is that you provide us with a URL for us to send HTTP POST requests to. These requests are the mechanism for delivering Events to you, which is the essence of how the communication layer of a ProQuant integration works.

# What's an Integration?
An Integration is data that describes a destination for ProQuant Events to be delivered to:

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

All of these properties above are defined by you, at the moment of creating the Integration.

# Creating an Integration
A simple HTTP POST request to our API will create an *Integration request* which will appear in ProQuant's mobile app and must be approved by the user. Once approved, the user may *enable* this Integration for any of their running strategies. A running strategy will forward signals to all of its *enabled* Integrations.

The request for creating an *Integration request*:

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

Once the user has *approved* the Integration from inside ProQuant's mobile application, the setup is done and they may enable this Integration for one or more of their running strategies.

At this point, Events will start arriving at `webhook.url`.

# Handling Events
Let's first take a look at how *versioning* works.

### Versioning
In order to achieve backwards-compatibility, a simple versioning system must be adhered to when consuming ProQuant Events.

Every request we send to `webhook.url` includes a single Event, formatted in all currently supported version formats, alongside meta information about the status of every single version:

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

*DISCLAIMER: The above is just an example, you can find our currently supported versions and their deprecation status below*

In the example above we support two versions - `v1` and `v2`. `deprecation` signifies the status of the version - if `null`, this version is the latest and is actively supported and maintained by us. However, `deprecation` could be a date in the ISO format - in that case, this version is no longer supported by us, and, from the specified date onwards, data for this version will no longer be included in any requests we send your way. It is your responsibility to track the value of `deprecation` and make steps toward implementing the latest version so as not to break your client's Integrations.

New versions will be released when breaking, non-backwards-compatible changes are introduced. This should happen rarely, if at all, and support for old versions will be continued long enough to allow you to upgrade without rush.

### Currently supported versions
| Version | Status | Deprecated on |
| ------ | ------ | ------ |
| v1 | Active | - |

### v1 Event Structure
All Events have a common structure:
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
| INTEGRATION_APPROVED | The user approved the Integration request | `null` |
| INTEGRATION_REJECTED | The user rejected the Integration request | `null` |
| INTEGRATION_DELETED | The user deleted the previously approved Integration | `null` |
| STRATEGY_ENABLED | The user enabled the Integration for one of their strategies. Enabling an Integration means that whenever the strategy produces signals, they will be sent to that Integration as Events. | [StrategyEventData](#v1-strategy-event-data) |
| STRATEGY_DISABLED | The user disabled the Integration for one of their strategies. Disabling an Integration means that the Integration will stop receiving signal Events from this strategy | [StrategyEventData](#v1-strategy-event-data)
| SIGNAL_OPEN | One of the user's running strategies produced a signal to open a position. | [SignalEventData](#v1-signal-event-data) |
| SIGNAL_CLOSE | One of the user's running strategies produced a signal to close its opened position. | [SignalEventData](#v1-signal-event-data)
| SIGNAL_REOPEN | One of the user's running strategies produced a signal to close its opened position and then immediately open a new one in the opposite direction (the `direction` property in the data signifies the *new* position direction). | [SignalEventData](#v1-signal-event-data)

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

We've covered what Events are and what type of ProQuant Events you may receive. Finally, let's look at how to appropriately respond to the HTTP Requests we send your way.

# Responding to Events
If all goes well and the Event has been handled successfully, *or is irrelevant to you and you did not consume/handle it at all*, simply respond with status code 200 and an empty body.

In case of an error, ProQuant clients will receive a push notification informing them that something went wrong with the Integration. What this notification will be, depends on the way you respond to the request. We understand, handle and support a predefined set of responses, which translate into well-designed errors shown to clients. If you respond with a status code >= 400 and we do not support that type of response, we will fall back to a Generic error notification.

Implementing and returning these specific Event Errors is optional, but recommended, as it would give clients a better overview of what happened, should something go wrong with their Integration.

Here are the requirements for an error response to be correctly parsed by us:

1. Status code must be >= `400`
2. Content type must be `application/json`
3. The response headers must include the header `x-proquant-error: 1`
4. The response body must include a key `type` with one of the following values:

| Type | When to use |
| ------ | ------ |
| InsufficientFunds | The user's account in your system did not have enough funds to handle the SIGNAL_OPEN/SIGNAL_REOPEN Event. |
| QuantityTooLow | The requested quantity in the SIGNAL_OPEN/SIGNAL_REOPEN Event is lower than the minimum allowed in your system, for the requested ticker |
| QuantityTooHigh | The requested quantity in the SIGNAL_OPEN/SIGNAL_REOPEN Event is higher than the maximum allowed in your system, for the requested ticker  |
| InvalidQuantityPrecision | The requested quantity's precision in the SIGNAL_OPEN/SIGNAL_REOPEN Event is not supported by your system (e.g. buying 0.0001 units of XAUUSD) |
| UnsupportedInstrument | The requested instrument in the SIGNAL_OPEN/SIGNAL_REOPEN Event is not supported by your system |
| PositionAlreadyExists | A strategy can have up to 1 positions open at any given time. If you receive SIGNAL_OPEN from a strategy that already has an open position in your system, you should error out with PositionAlreadyExists. |
| MarketClosed | You received SIGNAL_OPEN/SIGNAL_CLOSE/SIGNAL_REOPEN, but the market is closed according to your system, and you cannot execute the trade |
| PositionNotFound | You received SIGNAL_CLOSE but there isn't any open position from this strategy in your system |
| InvalidCredentials | You weren't able to identify and/or authorize the user this Event was meant for |

# Thanks!

That's it for the guide, thank you for reading! If you have any suggestions or questions, feel free to either open an issue here, or send us an e-mail at support@proquant.com
