# iOS OM SDK Documentation

These are the implementation instructions for the iOS Open Measurement SDK.

## Table of Contents

- [Initial Setup](#initial-setup)
  * [Importing the SDK](#importing-the-sdk)
  * [Initializing the SDK](#initializing-the-sdk)
    + [1. Activate the SDK](#1-activate-the-sdk)
    + [2. Fetch the OMID JS library](#2-fetch-the-omid-js-library)
    + [3. Identify your integration.](#3-identify-your-integration)
- [Ad Format Implementation Steps](#ad-format-implementation-steps-)
  * [WebView Display](#webview-display)
    + [1. Retrieve the ad response.](#1-retrieve-the-ad-response-)
    + [2. Inject the OMID JS library into the ad response.](#2-inject-the-omid-js-library-into-the-ad-response-)
    + [3. Create and configure the ad session.](#3-create-and-configure-the-ad-session-)
      - [Create context](#create-context)
      - [Designate event layer](#designate-event-layer)
      - [Create session](#create-session)
    + [4. Set the view on which to track viewability.](#4-set-the-view-on-which-to-track-viewability-)
      - [Set the view](#set-the-view)
      - [Register obstructions](#register-obstructions)
      - [Update view reference](#update-view-reference)
    + [5. Start the session.](#5-start-the-session)
    + [6. Signal the impression event.](#6-signal-the-impression-event)
    + [7. Stop the session.](#7-stop-the-session)
  * [WebView Video](#webview-video)
    + [1. Create a SessionClient.](#1-create-a-sessionclient)
    + [2. Retrieve the ad response.](#2-retrieve-the-ad-response)
    + [3. Inject the OMID JS library into the ad response.](#3-inject-the-omid-js-library-into-the-ad-response)
    + [4. Create and configure the ad session.](#4-create-and-configure-the-ad-session)
    + [5. Set the view on which to track viewability.](#5-set-the-view-on-which-to-track-viewability)
    + [6. Prepare the measurement resources.](#6-prepare-the-measurement-resources-)
    + [7. Initialize the JS ad session.](#7-initialize-the-js-ad-session)
      - [Create the session](#create-the-session)
      - [Set the video element](#set-the-video-element)
        * [Top window](#top-window)
        * [Cross-domain iframe](#cross-domain-iframe)
      - [Create the event publishers](#create-the-event-publishers)
    + [8. Start the session.](#8-start-the-session)
    + [9. Signal ad load.](#9-signal-ad-load)
    + [10. Signal the impression event.](#10-signal-the-impression-event)
    + [11. Signal playback progress events.](#11-signal-playback-progress-events)
      - [Playback events](#playback-events)
      - [Volume events](#volume-events)
      - [State changes](#state-changes)
  * [Native Display](#native-display)
    + [1. Retrieve the ad response.](#1-retrieve-the-ad-response)
    + [2. Prepare the measurement resources.](#2-prepare-the-measurement-resources)
    + [3. Create and configure the ad session.](#3-create-and-configure-the-ad-session-)
      - [Create context](#create-context-1)
      - [Designate event layer](#designate-event-layer-1)
      - [Create session](#create-session-1)
    + [4. Set the view on which to track viewability.](#4-set-the-view-on-which-to-track-viewability)
    + [5. Start the session.](#5-start-the-session-1)
    + [6. Signal the impression event.](#6-signal-the-impression-event-1)
    + [7. Stop the session.](#7-stop-the-session-1)
  * [Native Video](#native-video-)
    + [1. Retrieve the ad response.](#1-retrieve-the-ad-response-1)
    + [2. Prepare the measurement resources.](#2-prepare-the-measurement-resources-)
    + [3. Create and configure the ad session.](#3-create-and-configure-the-ad-session)
    + [4. Set the view on which to track viewability.](#4-set-the-view-on-which-to-track-viewability-1)
    + [5. Create the event publisher instances.](#5-create-the-event-publisher-instances)
    + [6. Start the session.](#6-start-the-session)
    + [7. Register the ad load event.](#7-register-the-ad-load-event)
    + [8. Register the impression.](#8-register-the-impression)
    + [9. Signal playback progress events.](#9-signal-playback-progress-events)
      - [Playback events](#playback-events-1)
      - [Volume events](#volume-events-1)
      - [State changes](#state-changes-1)
    + [10. Complete the session.](#10-complete-the-session)
- [Validating your OM SDK Implementation](#validating-your-om-sdk-implementation)
- [Distribute](#distribute)
  * [With a static library](#with-a-static-library)
  * [With a dynamic library](#with-a-dynamic-library)
- [FAQ](#faq)

# Initial Setup

Please implement the following setup steps before moving on to the specific ad format instructions.

## Importing the SDK

The first step is, of course, to add the OM SDK to your app or ads SDK.

Drag the static library (packaged as a framework) into your Project and make sure that it has been
added in the "Link Binary With Libraries" phase of your target. Then import the header:

```objective-c
#import <OMSDK/OMSDK.h>
```

## Initializing the SDK

You should implement these steps as early as possible in your app or SDK lifecycle.  

**Note** that OM SDK may only be used on the main UI thread.  Make sure you are on the main thread when you initialize the SDK, create its objects, and invoke its methods.

### 1. Activate the SDK

The first step is to initialize the OM SDK:

```objective-c
NSError *error;
BOOL sdkStarted = [[OMIDSDK sharedInstance] activateWithOMIDAPIVersion:OMIDSDKAPIVersionString
    error:&error];
```

If the SDK fails to initialize, the `error` instance will have information about the cause of
failure. All subsequent steps will fail as well if the SDK does not initialize properly.

### 2. Fetch the OMID JS library

As the next step, you must fetch the OMID JS service library and save the response.

```objective-c
NSURL *url = [NSURL URLWithString:@"https://my.server.com/omsdk.js"];
__weak typeof(self) weakSelf = self;
NSURLSessionDataTask *dataTask = [defaultSession dataTaskWithURL:url
    completionHandler:^(NSData *data, NSURLResponse
        *response, NSError *error) {
        if (error || weakSelf == nil) {
            return;
        }
        weakSelf.omidJS = [[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding];
}];
[dataTask resume];
```

We recommend that this step is performed right after you've initialized the SDK because you
will need the JS library to be available before you create any tracking session.

**Note** that this step is only necessary if you are injecting the OMID JS library client-side,
which may not necessarily be true if you use only WebView ad formats. This is because WebView ad
formats (not native, however) permit the injection server-side. If you are indeed injecting the JS
library server-side (i.e., within the ad response itself), you can skip this step.

### 3. Identify your integration.

Create a `Partner` object to identify your integration. The IAB Tech Lab will assign a unique
partner name to you at the time of integration, so this is the value you should use here.

The version string should represent the integration version in semantic versioning format. For an
ads SDK, this should be the same as your SDK's semantic version. For an app publisher, this should
be the same as your app version.

```objective-c
OMIDPartner *partner = [[OMIDPartner alloc] initWithName:PARTNER_NAME
    versionString:PARTNER_SDK_OR_APP_VERSION];
```

You do not need to use the object at this time, but it will be required each time you track a new
ad.

# Ad Format Implementation Steps <a name="ad-formats"></a>

The following sections describe how to implement measurement for various ad formats. They assume
you've already imported the library and implemented the initialization code.

## WebView Display

The steps below describe how to create a tracking session for a WebView (HTML) ad.

### 1. Retrieve the ad response. <a name="webview-ad-response"></a>

Retrieve the ad response as you normally would. For the purposes of the subsequent steps, the ad
response should be an HTML string.

```objective-c
NSString *adResponseHtmlString = @"<html>...</html>";
```

### 2. Inject the OMID JS library into the ad response. <a name="webview-inject-omid"></a>

After you've retrieved the ad response, inject the JS service into the ad response and load
inside the ad WebView:

```objective-c
NSError *error;
String *htmlString = [OMIDScriptInjector injectScriptContent:self.omidJS
    intoHTML:adResponseHtmlString error:&error];
[adWebView loadHTMLString:htmlString baseURL:[[NSBundle mainBundle] bundleURL]];
```

**Note**, as mentioned in Step 2 of Initialization, if you have injected the JS service library
server-side, this step is not necessary.

### 3. Create and configure the ad session. <a name="ad-session-config"></a>

Create the session in the following sequence of steps.

Please note that each class's initializer has its own input error checking, so it is important to
pass through an error object and validate that each instance initialized successfully.

**Note**: in order to prevent issues with starting the session later, you must wait until the
WebView finishes loading OM SDK JavaScript before creating the `OMIDAdSession`. Creating the session
sooner than that may result in an inability to signal events (impression, etc.) to verification
scripts inside the WebView. The easiest way to avoid this issue is to create the `OMIDAdSession`
in a webview delegate callback (`-[WKNavigationDelegate webView:didFinishNavigation:]` or
`-[UIWebViewDelegate webViewDidFinishLoad:]`). Alternatively, if an implementation can receive an
HTML5 DOMContentLoaded event from the WebView, it can create the `OMIDAdSession` in a message
handler for that event.

```
// In an implementation of WKNavigationDelegate:
- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation {
  if (self.omidSession != nil) return;

  /*
   * Create the session.
   * Eliding the steps for brevity. Please reference for full steps below.
   * /
  self.omidSession = [[OMIDAdSession alloc] initWithConfiguration:config
      adSessionContext:context error:nil];

  // Set the view on which to track viewability
  self.omidSession.mainAdView = webView;

  // Start session
  [self.omidSession start];
}
```

#### Create context

First, create a context with a reference to the partner object you created in the setup step and
the ad's WebView.

```objective-c
NSError *ctxError;
// the custom reference ID may not be relevant to your integration in which case you may pass an
// empty string.
NSString *customRefId = ...;
OMIDAdSessionContext *context = [[OMIDAdSessionContext alloc] initWithPartner:partner
    webView:adWebView customReferenceIdentifier:customRefId error:&ctxError];
```

#### Designate event layer

Then designate which layer is responsible for signaling the impression event. For WebView display
ads this is generally the native layer. We'll get to the actual signaling of the event in a
subsequent step.

```objective-c
NSError *cfgError;
OMIDAdSessionConfiguration *config = [[OMIDAdSessionConfiguration alloc]
    initWithImpressionOwner:OMIDNativeOwner videoEventsOwner:OMIDNoneOwner
    isolateVerificationScripts:NO error:&cfgError];
```

**Note** that it is important that the videoEventsOwner parameter should be set to `OMIDNoneOwner`
for display formats. Setting to anything else will cause the `mediaType` parameter passed to
verification scripts to be set to `video`.

#### Create session

Finally, create the session itself.

```
NSError *sessError;
self.omidSession = [[OMIDAdSession alloc] initWithConfiguration:config
    adSessionContext:context error:&sessError];
```

You will want to store a strong reference to the session either as a property or in a mapping
dictionary (if you expect multiple concurrent ad sessions, in which case you would map from an ad
instance to its associated session).

The subsequent steps assume you were able to initialize the session successfully. Of course, if the
session failed to initialize with an error, it will return a nil reference and you should not use
it subsequently.

### 4. Set the view on which to track viewability. <a name="set-session-view"></a>

#### Set the view

Set the view on which to track viewability. For a WebView ad, this will be the WebView itself. For
a native ad, this will be the native view that contains all the relevant elements of the ad.

```objective-c
self.omidSession.mainAdView = adView;
```

#### Register obstructions

If there are any native elements which you would consider to be part of the ad, such as a close
button, some logo text, or another decoration, you should register them as friendly obstructions
to prevent them from counting towards coverage of the ad. This applies to any ancestor or peer 
views in the view hierarchy (all sub-views of the adView will be automatically treated as part of
the ad):

```objective-c
[self.omidSession addFriendlyObstruction:view];
```

#### Update view reference

If the view changes at a subsequent time due to a fullscreen expansion or for a similar reason, you
should always update the `mainAdView` reference to whatever is appropriate at that time.

### 5. Start the session.

Starting the session does not trigger an impression yet -- it only prepares the session for
tracking. It is important to start the session before dispatching any events.

```objective-c
[self.omidSession start];
```

As described in the [previous step](#ad-session-config), this should occur after the WebView has
loaded.

### 6. Signal the impression event.

The definition of an "impression" is generally accepted to be on ad render so this is likely when
you will want to dispatch the event. The event should be dispatched only once and attempting to
trigger it multiple times is an error.

```objective-c
NSError *adEvtsError;
OMIDAdEvents *adEvents = [[OMIDAdEvents alloc] initWithAdSession:omidSession error:&adEvtsError];
NSError *impError;
[adEvents impressionOccurredWithError:&impError];
```

### 7. Stop the session.

Stop the session when the impression has completed and the ad will be destroyed. Note, that
after you've stopped the session it is an error to attempt to start it again or trigger an
impression on the finished session.

**Note** that ending an OMID ad session sends a message to the verification scripts running inside the webview supplied by the integration.  So that the verification scripts have enough time to handle the `sessionFinish` event, the integration must maintain a strong reference to the webview for at least 1.0 seconds after ending the session.

```objective-c
[self.omidSession finish];
self.omidSession = nil;
```

## WebView Video

The implementation instructions for WebView Video are similar in many respects to WebView Display
on the native side. The primary point of difference is that WebView Video will generally require
some additional implementation within the JavaScript layer inside the WebView. This additional
JavaScript configuration will likely be performed to a large degree in your ad server so that the
necessary components are embedded within the ad response by the time your app or SDK receives it.

The following implementation instructions assume that the JavaScript layer is responsible for these
actions:
* parsing of the ad response to load measurement scripts
* impression registration
* playback progress notifications

Impression events and playback progress can be handled from the native layer as well. You can
indicate which layer is responsible for event handling at the time that you create the
`OMIDAdSessionConfiguration` instance by passing through the appropriate owner (Native or JavaScript)
for the respective events. As mentioned previously, this guide assumes you will be implementing the
responsibilities cited above in the JavaScript layer. If you would like instructions on how to do
the same in the native layer, please reference the [Native Video](#native-video) implementation instructions.

### 1. Create a SessionClient.

Within the HTML ad response, please create a SessionClient. You will interact with classes from
the SessionClient to signal events and pass metadata.

```javascript
var sessionClient;

try {
  sessionClient = OmidSessionClient[SESSION_CLIENT_VERSION];
} catch (e) {}

if (!sessionClient) {
  return;
}

const AdSession = sessionClient.AdSession;
const Partner = sessionClient.Partner;
const Context = sessionClient.Context;
const VerificationScriptResource = sessionClient.VerificationScriptResource;
const AdEvents = sessionClient.AdEvents;
const VideoEvents = sessionClient.VideoEvents;
```

### 2. Retrieve the ad response.

See [this step](#webview-ad-response) of WebView Display. This guide assumes that the ad response
will contain HTML (which will render the video player) as well as a VAST component.

### 3. Inject the OMID JS library into the ad response.

See [this step](#webview-inject-omid) of WebView Display.

### 4. Create and configure the ad session.

See [this step](#ad-session-config) of WebView Display.

**Note** that you will want to designate which layer is responsible for signaling events differently
than for WebView Display. Generally for WebView video the JavaScript layer will be signaling both
the impression and video events. If this is not done correctly, the OMID service will not pass
events to any verification scripts.

As with WebView display, you should ensure the session set up and creation should happen only after
receiving the WebView loaded event. Please reference [this step](#ad-session-config) of the WebView
Display instructions for further detail.

Furthermore, you should decide on the appropriate value for the isolateVerificationScripts
parameter. The effect of a `YES` value is that measurement resources will be placed in a sandboxed
iframe where they cannot access the video ad element. If you specify `NO`, they will be placed into
a same-origin iframe. The [FAQ](#faq) has further detail on this setting.

```objective-c
NSError *cfgError;
OMIDAdSessionConfiguration *config = [[OMIDAdSessionConfiguration alloc]
    initWithImpressionOwner:OMIDJavaScriptOwner videoEventsOwner:OMIDJavaScriptOwner
    isolateVerificationScripts:YES|NO error:&cfgError];
```

### 5. Set the view on which to track viewability.

See [this step](#set-session-view) of WebView Display.

### 6. Prepare the measurement resources. <a name="webview-parse-scripts"></a>

The following instructions assume the ad response uses the VAST 4.1 Verification node, either per
the 4.1 spec exactly or in the Extensions node. The exact details are outside the scope of this
document and are addressed by other Tech Lab guidance and documentation.

If you are not able to use VAST or the 4.1 node, you must find an alternative way to embed this
information within the ad response and will likely need to work with each measurement vendor
individually to determine the correct mechanism for loading their tags.

Using this 4.1 Verification node as an example:

```xml
<Verification vendor="vendorKey">
  <JavaScriptResource apiFramework="omid">
    <![CDATA[https://measurement.domain.com/tag.js]]>
  </JavaScriptResource>
  <VerificationParameters>
    <![CDATA[{...}]]>
  </VerificationParameters>
</Verification>
```

First create an array to hold one or more measurement resources that you will parse in the next
step:

```javascript
var resources = [];
```

Then, for each Verification node in the ad response, create a `VerificationScriptResource` instance
as follows:

```javascript
var vendorKey = ...; // parsed from "vendor" attribute
var params = ...;    // parsed from VerificationParameters as a string
var url = ...;       // parsed from JavaScriptResource
var resource = new VerificationScriptResource(url, vendorKey, params);
resources.add(resource);
```

**Note** that the vendorKey must match the `vendor` attribute from the Verification node in order
for OM SDK to pass any impression specific metadata in the VerificationParameters to the correct
verification script.

### 7. Initialize the JS ad session.

Next, create the JS ad session and pass through the measurement resources you parsed from the ad
response in the previous step. You will need to use this session instance in order to subscribe
to the native session start event as well as to load the resources.

#### Create the session

```javascript
var partner = new Partner(PARTNER_NAME, PARTNER_SDK_OR_APP_VERSION);
var context = new Context(partner, resources);
var adSession = new AdSession(context);
```

Note that the parameters for `Partner` here should match those you've passed in the native layer.

#### Set the video element

In order to ensure the ad is measured correctly, you should provide a reference to the video
element when it is available. The right steps will depend on whether the video element is in the
top window or inside a cross-domain iframe.

##### Top window

Simply provide the video element reference using the `Context` object you created previously:

```javascript
const videoElement = document.getElementById(...);
context.setVideoElement(videoElement);
```

##### Cross-domain iframe

There are two possible scenarios when the video element is in a cross-domain iframe:

1. The `Session` and the element are inside the cross-domain iframe.
2. You are able to create a `Session` in the top window as well as inside the cross-domain iframe
    with the ad element.

In the first scenario, you should mark up the iframe with a predefined class name: `omid-element`.
This will ensure the OMID JS service running in the top level is able to locate the iframe. The next
step is to indicate the position of the element within the iframe. This can be done by passing the
element's offset to the `Session` instance you created inside the iframe:

```javascript
// elementBounds is a rect {x, y, width, height} that indicates the position of the video element
// inside the iframe.
adSession.setElementBounds(elementBounds)
```

In the second scenario, you should pass the reference to the iframe to the `Session` and `Context`
instance in the top level:

```javascript
const iframeElement = document.getElementById(...);
context.setSlotElement(iframeElement);
```

And within the iframe, provide the element's offset to the `Session` instance inside the iframe:

```javascript
adSession.setElementBounds(elementBounds)
```

#### Create the event publishers

In addition, you should also at this time create the event notification objects which you will use
to signal the impression and playback events.

```javascript
const adEvents = new AdEvents(adSession);
const videoEvents = new VideoEvents(adSession);
```

### 8. Start the session.

Start the session in the native layer before signaling any events in the JS layer.

```objective-c
[self.omidSession start];
```

### 9. Signal ad load.

When the video has loaded, signal the load event and pass through the following metadata:

```javascript
adSession.registerSessionObserver((event) => {
  if (event.type === "sessionStart") {
    // setup code
    ...
    // load event
    videoEvents.loaded({ isSkippable: skippable, isAutoPlay: autoplay, position: position });
    // other event code
    ...
  } else if (event.type === "sessionError") {
    // handle error
  } else if (event.type === "sessionFinish") {
    // clean up
  }
});
```

**Note** that we are dispatching the event within the session observer callback. This is to ensure
that we do not dispatch any events until we've received session start. All events in the JS layer
must be dispatched only after the session start event. You should also check the event type to
ensure you handle each event type appropriately.

### 10. Signal the impression event.

When you are ready, signal the impression event using the events object you created in the
previous step. The standard time to signal an impression is on ad render, and right before the ad
starts playing for video. As noted in the previous step, you must also ensure that you don't
dispatch the impression event until **after** you've received the session start event.

```javascript
// this should be within the same callback as the load event from the previous step
adSession.registerSessionObserver((event) => {
  if (event.type === "sessionStart") {
    ...
    adEvents.impressionOccurred();
    ...
  } else if (event.type === "sessionError") {
    // handle error
  } else if (event.type === "sessionFinish") {
    // clean up
  }
})
```

Note that we are signaling this event from the JavaScript rather than native layer since we
designated the JavaScript layer as the impression event owner.

### 11. Signal playback progress events.

As playback progresses, dispatch notifications for the playback progress events. You should
at a minimum signal the following events, as appropriate:

* start
* first quartile [25%]
* midpoint [50%]
* third quartile [75%]
* complete [only if ad reaches 100%]
* pause [user initiated]
* resume [user initated]
* bufferStart [playback paused due to buffering]
* bufferEnd [playback resumes after buffering]
* player volume change
* skipped [any early termination of playback]

#### Playback events

Monitor video playback to signal the progress events at appropriate times (reference bullet list
above). Generally the timing of the events corresponds to the industry defined standards VPAID and
VAST.

Note that the start event is distinct from others because it requires the duration of the ad as
well as the creative volume. Please ensure that video player duration is available at the time you
dispatch this event.

```javascript
if (!isNaN(player.duration)) {
  videoEvents.start(player.duration, player.volume);
} else {
  // wait until duration is available to start
}
```

Note that the `videoPlayerVolume` parameter should only be the volume of the player element. The
device volume is detected by the SDK automatically. The player volume should be normalized between
0 and 1. If the creative volume only supports muted or unmuted, it is sufficient to use the 
following for the videoPlayerVolume parameter:

```javascript
  videoEvents.start(player.duration, player.muted ? 0 : 1);
```

#### Volume events

With respect to volume changes, you are responsible for notifying only about creative volume
changes. See the note about `videoPlayerVolume` in the section above.

```javascript
videoEvents.volumeChange(player.volume);
```

#### State changes

Finally, you are also responsible for signaling any player state changes. If the player can
expand to fullscreen as well as exit fullscreen, you will want to signal these state changes as
follows:

```javascript
// entering fullscreen
videoEvents.playerStateChange("fullscreen");

// exiting fullscreen
videoEvents.playerStateChange("normal");
```

## Native Display

For clarity, when we refer to Native Display, we refer to non-WebView display ad formats where the
components of the ad are native (non-HTML) UI elements.

### 1. Retrieve the ad response.

Retrieve the ad response as you normally would. For a native ad the ad response may generally be in
the form of a JSON which includes some metadata and URLs to the ad assets.

### 2. Prepare the measurement resources.

The steps here are conceptually similar to the [same step](#native-parse-scripts) of Native Video. Unlike video, there is no
standard ad response format for display, so you must find an alternative way to determine which
measurement resources should be tracking a given ad impression, but, in any case, you will most
likely return this information as part of the ad response one way or another.

### 3. Create and configure the ad session. <a name="native-ad-session-config"></a>

Create the session in 3 steps.

Please note that each class's initializer has its own input error checking, so it is important to
pass through an error object and validate that each instance initialized successfully.

#### Create context

First, create a context with a reference to the partner object you created in the setup step, the
OMID JS, and the measurement resources.

```objective-c
NSError *ctxError;
OMIDAdSessionContext *context = [[OMIDAdSessionContext alloc] initWithPartner:partner
    script:self.omidJS resources:scripts customReferenceIdentifier:nil error:&ctxError];
```

#### Designate event layer

Then designate which layer is responsible for signaling the impression event. For Native display
this will generally be the native layer.

```objective-c
NSError *cfgError;
OMIDAdSessionConfiguration *config = [[OMIDAdSessionConfiguration alloc]
    initWithImpressionOwner:OMIDNativeOwner videoEventsOwner:OMIDNoneOwner
    isolateVerificationScripts:NO error:&cfgError];
```

**Note** that it is important that the videoEventsOwner parameter should be set to `OMIDNoneOwner`
for display formats. Setting to anything else will cause the `mediaType` parameter passed to
verification scripts to be set to `video`.

#### Create session

Finally, create the session itself.

```
NSError *sessError;
self.omidSession = [[OMIDAdSession alloc] initWithConfiguration:config
    adSessionContext:context error:&sessError];
```

### 4. Set the view on which to track viewability.

See [this section](#set-session-view) of WebView display.

### 5. Start the session.

Starting the session does not trigger an impression yet -- it only prepares the session for
tracking. It is important to start the session before dispatching any events.

Generally you should start the session as soon as you've completed the previous steps.

```objective-c
[self.omidSession start];
```

### 6. Signal the impression event.

The definition of an "impression" is generally accepted to be on ad render so this is likely when
you will want to dispatch the event. The event should be dispatched only once and attempting to
trigger it multiple times is an error.

```objective-c
NSError *adEvtsError;
OMIDAdEvents *adEvents = [[OMIDAdEvents alloc] initWithAdSession:omidSession error:&adEvtsError];
NSError *impError;
[adEvents impressionOccurredWithError:&impError];
```

### 7. Stop the session.

Stop the session when the impression has completed and the ad will be destroyed. Note, that
after you've stopped the session it is an error to attempt to start it again or trigger an
impression on the finished session.

```objective-c
[self.omidSession finish];
self.omidSession = nil;
```

## Native Video <a name="native-video"></a>

Please follow the instructions below for how to correctly track a native video ad.

### 1. Retrieve the ad response.

Retrieve the ad response as you would normally. Generally this will be a VAST document.

### 2. Prepare the measurement resources. <a name="native-parse-scripts"></a>

In this step, you will determine which measurement resources should be tracking the ad. Overall the
instructions are similar to [this step](#webview-parse-scripts) of the WebView Video instructions.
As before, we will assume that the ad response contains one or more Verification nodes as specified
in VAST 4.1.

First we create an array to hold the scripts:

```objective-c
NSMutableArray *scripts = [NSMutableArray new];
```

Then parse each Verification node and add the associated `OMIDVerificationScriptResource` to the
array of resources:

```objective-c
NSURL *url = [NSURL URLWithString:stringUrl];
NSString *vendorKey = ...;
NSString *params = ...;

[scripts addObject:[[OMIDVerificationScriptResource alloc] initWithURL:url vendorKey: vendorKey
    parameters:params]];
```

### 3. Create and configure the ad session.

See [this step](#native-ad-session-config) of the Native Display implementation instructions.

**Note** that you will want to specify a video event owner that is different from the example
provided with native display.

```objective-c
NSError *cfgError;
OMIDAdSessionConfiguration *config = [[OMIDAdSessionConfiguration alloc]
    initWithImpressionOwner:OMIDNativeOwner videoEventsOwner:OMIDNativeOwner
    isolateVerificationScripts:NO error:&cfgError];
```

### 4. Set the view on which to track viewability.

See [this step](#set-session-view) of WebView Display.

### 5. Create the event publisher instances.

You will use these instances to signal impression and playback events, so you will want to hold on
to them after you've created them.

```objective-c
// to signal impression event
NSError *aErr;
self.omidAdEvents = [[OMIDAdEvents alloc] initWithAdSession:self.omidSession error:&aErr];

// to signal video events
NSError *vErr;
self.omidVideoEvents = [[OMIDVideoEvents alloc] initWithAdSession:self.omidSession error:&vErr];
```

Be sure to check the errors and ensure the event instances were created properly before using them
subsequently. If there was an error, they will be nil.

### 6. Start the session.

As noted previously, signaling the start of the session does not yet trigger the impression. This
only prepares the session for tracking. You will want to do this as close to completing the
previous steps as possible.

```objective-c
[self.omidSession start];
```

### 7. Register the ad load event.

Dispatch the loaded event to signal that the ad has loaded and is ready to play. Please reference
the `OMIDVASTProperties` header for the appropriate values.

```objective-c
OMIDVASTProperties vastProperties = ...;
[self.omidVideoEvents loadedWithVastProperties:vastProperties];
```

### 8. Register the impression.

Signal the start of the impression. Generally this should occur right before the ad has started to
play.

```objective-c
NSError *impError;
[self.omidAdEvents impressionOccurredWithError:&impError];
```

### 9. Signal playback progress events.

As playback progresses, dispatch notifications for the playback progress events. You should
at a minimum signal the following events, as appropriate:

* start
* first quartile [25%]
* midpoint [50%]
* third quartile [75%]
* complete [only if ad reaches 100%]
* pause [user initiated]
* resume [user initated]
* bufferStart [playback paused due to buffering]
* bufferEnd [playback resumes after buffering]
* player volume change
* skipped [any early termination of playback]

#### Playback events

Monitor video playback to signal the progress events at appropriate times (reference bullet list
above). Generally the timing of the events corresponds to the industry defined standards VPAID and
VAST.

Note that the start event is distinct from others because it requires the duration of the ad as
well as the creative volume:

```objective-c
[self.omidVideoEvents startWithDuration:VIDEO_DURATION videoPlayerVolume:PLAYER_VOLUME_0_TO_1];
```

#### Volume events

With respect to volume changes, you are responsible for notifying only about creative volume
changes. Device volume is detected by the SDK automatically.

```objective-c
[self.omidEvents volumeChangeTo:PLAYER_VOLUME_0_TO_1]
```

#### State changes

Finally, you are also responsible for signaling any player state changes. If the player can
expand to fullscreen as well as exit fullscreen, you will want to signal these state changes as
follows:

```objective-c
// enter fullscreen
[self.omidVideoEvents playerStateChangeTo:OMIDPlayerStateFullscreen];

// exit fullscreen
[self.omidVideoEvents playerStateChangeTo:OMIDPlayerStateNormal];
```

**Special note for MPMoviePlayerController**: if your playback happens via a fullscreen
`MPMoviePlayerController` instance, you must ensure that you dispatch the fullscreen event to ensure
that viewability is tracked correctly.

### 10. Complete the session.

Stop the session on completion or termination of ad playback.

```objective-c
[self.omidSession finish];
self.omidSession = nil
```

# Validating your OM SDK Implementation

Validating your steps is an important part of the integration process. Follow the instructions [here](validation.md).

# Distribute

This section is directed at ads SDKs that will want to distribute their SDK after integrating the OM
SDK in order to provide measurement across their network of publishers. While the ads SDK may very
well choose to distribute the OM SDK as a separate component, this will generally provide a poorer
usability experience compared to embedding the OM SDK within. The following instructions detail how
to embed the OM SDK when possible. Note that the OM SDK does use namespacing so that it can be
independently included within multiple ads SDKs in a single app without issues.

## With a static library

[TBD]

## With a dynamic library

Distribution as a dynamic library may be supported in a future release, but the current version is
distributed as a static library only.

# FAQ

FAQ available [here](faq.md).
