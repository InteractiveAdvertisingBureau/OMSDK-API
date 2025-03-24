# Validating your OM SDK Implementation

## Table of Contents

- [Summary](#summary)
- [Steps](#steps)
  * [1. Load and inject the script.](#1-load-and-inject-the-script)
  * [2. Follow the standard implementation steps for your ad format.](#2-follow-the-standard-implementation-steps-for-your-ad-format)
  * [3. Start your proxy session.](#3-start-your-proxy-session)
  * [4. Build and run your app.](#4-build-and-run-your-app)
  * [5. Load ads and observe logs from the Validation Client.](#5-load-ads-and-observe-logs-from-the-validation-client)
    + [All Ads](#all-ads)
    + [Display Ads](#display-ads)
    + [Video Ads](#video-ads)

# Summary

While the SDK signals critical implementation issues via errors and exceptions, you should perform
additional validation steps to ensure that your implementation works end to end. You can do this by
using a JavaScript script that is bundled with the SDK distribution called the Validation
Verification Client.

Below we describe how to execute the script and monitor for events from it to confirm correct
implementation. Please note that while you should certainly perform this validation yourself, you
should seek further guidance from the IAB Tech Lab to ensure that your implementation is
independently certified.

# Steps

## 1. Load and inject the script.

The exact method will vary depending on whether you are validating a WebView or native ad.

If you are validating a WebView ad, you will want to embed the Validation Verification Client
script in the ad response.

If you are validating a native ad, you will want to ensure that you inject the Validation Client
as one of the measurement resources. You can host the script on a remote server or proxy it
locally.

An example in iOS:

```
NSMutableArray *scripts = [NSMutableArray new];
NSURL *url = [NSURL urlWithString:@"127.0.0.1/omid-validation-verification-script-v1.js"];
NSString *vendorKey = @"dummyVendor"; // you must use this value as is
NSString *params = @"{\"k\":\"v\"}"
[scripts addObject:[[OMIDVerificationScriptResource alloc] initWithURL:url vendorKey:vendorKey
    parameters:params]];
```

The script is generally named `omid-validation-verification-script-v1.js` out of the box.

## 2. Follow the standard implementation steps for your ad format.

Make sure that you've implemented the validation steps as specified in the implementation
instructions depending on the ad format you are validating.

## 3. Start your proxy session.

The Validation Verification Client will "log" via HTTP requests to localhost. These requests will
be sent in response to various events (start of a session, impression event, playback progress,
viewability update, etc.) and will include the parameters included with those events. In order to
see these events, you will need to have a proxy session running to capture the pings.

## 4. Build and run your app.

Then move on to the next step where you will load and interact with ads in your app.

## 5. Load ads and observe logs from the Validation Client.

As you interact with the app and load ads, monitor logs from the Validation Client in your proxy. By
default they will go to `localhost:66`. e.g.,

```
http://localhost:66/sendMessage?msg=[url encoded message content]
```

The exact content of the messages will vary depending on what event is being logged, but, with few
exceptions, you can expect the event to be serialized as a JSON object with the following structure:

```json
{
  "adSessionId": "5CAE70B9-2D92-4F09-A10F-44358F316B40",
  "timestamp": "[timestamp]",
  "type": "[eventType]",
  "data": {}
}
```

A few notes on the above:

* The ad session ID will be the same across all events for a given ad session.
* The `"data"` property may not be present on all events.
* The above JSON object will be serialized and URL quoted, since the event will be pinged over HTTP.

Below are the events you should check for.

### All Ads

* Initialization.

```
OmidSupported[true]
```

* Session start. Note, event `"type"` is `"sessionStart"`. You should also confirm the content of
the `"data"` property contains expected values.

```json
{
  "adSessionId": "A811D9AE-947E-49FB-9572-BCB13B0F9FC8",
  "timestamp": 1513975492522,
  "type": "sessionStart",
  "data": {
    "context": {
      "environment": "app",
      "omidNativeInfo": {
        "partnerName": "partner",
        "partnerVersion": "1.0"
      },
      "deviceInfo": {
        "deviceType": "iPhone7,1",
        "osVersion": "11.1.0",
        "os": "iOS"
      },
      "adSessionType": "html",
      "app": {
        "appId": "com.partner.TestApp",
        "libraryVersion": "1.0.0"
      },
      "customReferenceData": "",
      "supports": [
        "clid",
        "vlid"
      ],
      "omidJsInfo": {
        "serviceVersion": "1.0.1"
      }
    }
  }
}
```

* Impression. Should be triggered only once for a given ad session.

```json
{
  "adSessionId": "A811D9AE-947E-49FB-9572-BCB13B0F9FC8",
  "timestamp": 1515021621358,
  "type": "impression",
  "data": {
    "mediaType": "video",
    "viewport": {
      "width": 414,
      "height": 736
    },
    "adView": {
      "percentageInView": 100,
      "reasons": [],
      "geometry": {
        "width": 300,
        "height": 250,
        "x": 57,
        "y": 164
      },
      "onScreenGeometry": {
        "width": 300,
        "height": 250,
        "x": 57,
        "y": 164,
        "obstructions": []
      }
    }
  }
}
```

* Geometry change (viewability update). Please confirm that the events are triggered as the ad is
scrolled (if it can be scrolled in and out of view) and that the viewable percentage reflected in
`adView.percentageInView` is accurate.

```json
{
  "adSessionId": "A811D9AE-947E-49FB-9572-BCB13B0F9FC8",
  "timestamp": 1513978723380,
  "type": "geometryChange",
  "data": {
    "viewport": {
      "width": 375,
      "height": 667
    },
    "adView": {
      "percentageInView": 0,
      "reasons": [
        "clipped"
      ],
      "geometry": {
        "width": 300,
        "height": 300,
        "x": 0,
        "y": 1100
      },
      "onScreenGeometry": {
        "width": 300,
        "height": 0,
        "x": 0,
        "y": 1100,
        "obstructions": []
      }
    }
  }
}
```

### Display Ads

The events described in the previous section should encapsulate all of the events that you would
need to monitor. Please ensure that the events are dispatched at appropriate times for a given ad
session.

For example, if you are seeing multiple session start events for an inline ad as you are
scrolling it in and out of view, there is likely a problem in your native implementation and you
should ensure you are maintaining a single ad session throughout.

If you are not seeing viewability updates there was likely a prior issue in initialization, such as
not setting the ad view.

And, as a final example, if the viewable percentages are incorrect, make sure to check for any
obstructions that could be getting in the way of the ad.

For native display please ensure that the verification parameters for the test script are passed
through correctly. You can do so by checking the `"verificationParameters"` property in the
`"data"` attribute of the session start event:

```json
{
  "adSessionId": "A811D9AE-947E-49FB-9572-BCB13B0F9FC8",
  "timestamp": 1513975492522,
  "type": "sessionStart",
  "data": {
      "verificationParameters": "..."
    }
  }
}
```

### Video Ads

For video ads you should look for playback events in addition to the events outlined previously.

As noted with regards to Native Display in the Display section, you should also check that the
`verificationParameters` are passed through correctly in the session start event.

You should check that all of the required video events are fired as expected at the correct time and
in correct order. Here is an example print out of the `start` event:

```json
{
  "adSessionId": "A811D9AE-947E-49FB-9572-BCB13B0F9FC8",
  "timestamp": 1515021630122,
  "type": "start",
  "data": {
    "duration": 30.022,
    "videoPlayerVolume": 1,
    "deviceVolume": 0.6
  }
}
```

If the video player can be muted, make sure that the appropriate volume change events are being
recorded with appropriate values. Check the `"data"` dictionary for the device and player volume
values.

```json
{
  "adSessionId": "A811D9AE-947E-49FB-9572-BCB13B0F9FC8",
  "data": {
    "deviceVolume": 0.06666667,
    "videoPlayerVolume": 1
  },
  "timestamp": 1515699112351,
  "type": "volumeChange"
}
```

Note that not every playback event will contain the data property. For example:

```json
{
  "adSessionId": "A811D9AE-947E-49FB-9572-BCB13B0F9FC8",
  "timestamp": 1515021638157,
  "type": "firstQuartile"
}
```
