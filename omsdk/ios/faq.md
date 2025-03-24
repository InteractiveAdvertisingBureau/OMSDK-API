# Implementation FAQ

- [Questions](#questions)
    + [Can I inject the OMID JS library into the ad response server-side?](#can-i-inject-the-omid-js-library-into-the-ad-response-server-side)
    + [When is it appropriate to signal the impression event?](#when-is-it-appropriate-to-signal-the-impression-event)
    + [What is the lifecycle of an ad session?](#what-is-the-lifecycle-of-an-ad-session)
      - [Note on native display](#note-on-native-display)
    + [Where do I get the measurement resources for native ad formats?](#where-do-i-get-the-measurement-resources-for-native-ad-formats)
    + [Can I reuse a WebView to load multiple display ads in sequence?](#can-i-reuse-a-webview-to-load-multiple-display-ads-in-sequence)
    + [How do I make sure parts of the ad that are not the main view don't impact my viewability?](#how-do-i-make-sure-parts-of-the-ad-that-are-not-the-main-view-dont-impact-my-viewability)
    + [What is the purpose of the event owner?](#what-is-the-purpose-of-the-event-owner)
    + [I have only native ad formats. Does the OM SDK use WebViews internally?](#i-have-only-native-ad-formats-does-the-om-sdk-use-webviews-internally)
    + [What is the purpose of isolating verification scripts?](#what-is-the-purpose-of-isolating-verification-scripts)

# Questions

Below is a list of frequently asked questions in relation to implementing the OM SDK.

### Can I inject the OMID JS library into the ad response server-side?

Yes. The implementation docs assume you will fetch the ad response and inject the OMID JS library
client-side, but you can also do the same server-side. The key is that the library must be injected
and available **before** any third party measurement scripts load.

### When is it appropriate to signal the impression event?

On ad render for display, per industry guidelines, and right before start of playback for video. To
minimize discrepancies, send the OMID impression event at the same time your ad platform signals an
impression.

### What is the lifecycle of an ad session?

The lifecycle of the session, from start to finish, should match the lifecycle of the ad. A single
session should be maintained for a single ad from the time the ad is created until it is completely
dismissed or destroyed.

One common question is what should you do with an ad session for an ad that the user scrolls
out of view. If the ad can be scrolled into view again and you would count this as the same, rather
than a new, impression, you should maintain the ad session you originally created for the ad;
you do not need to call any methods on the session to notify it that the ad has scrolled out of
view.

Similarly, if the user can interrupt ad playback by backgrounding the app and then returns to
the app to resume the same impression, you want to make sure that you maintain only a single ad
session; do not attempt to destroy the ad session when the user backgrounds the app and create a
new one when they reopen the app and the ad resumes. You should certainly, however, make sure to
signal the pause and resume events appropriately in this instance.

#### Note on native display

For native display ads the lifecycle we described above is challenging to do in practice since they
are often implemented with view reuse (i.e., the views are created and destroyed dynamically as the
user scrolls, for example, through a feed). In this case, it's difficult to predict the lifecycle of
the ad *itself* making the management of the ad session challenging as well.

What this means from a practical standpoint is that you may have no other choice than to create and
destroy multiple ad sessions for a single impression. The issue is that this will result in
impression discrepancies and other measurement problems since the impression is no longer tracked
as a single continuous session. You will need to speak with your preferred measurement companies to
determine how to ensure proper impression counting in this scenario. The capabilities may differ
but at a minimum will likely require the use of an impression ID to "tie together" several ad
sessions to a single impression.

### Where do I get the measurement resources for native ad formats?

Please see the relevant implementation steps for specifics.

For video, you will typically want to leverage VAST 4.1 but you can technically embed the
necessary information (vendor URL, parameters, vendor identifier) in the ad response in any other
manner convenient to you. The advantage of using the VAST 4.1 spec is standardization. If your
video player does not support VAST 4.1, you must still use the VAST 4.1 verification node and place
it within the Extensions section.

For display, VAST 4.1 is not available, so you must embed the metadata in the response in an
alternative format.

### Can I reuse a WebView to load multiple display ads in sequence?

Reusing the WebView to load multiple ads is fine, as long as you replace the contents with the
new ad response each time and of course repeat the steps outlined in the WebView Display sections.

### How do I make sure parts of the ad that are not the main view don't impact my viewability?

Make sure to pass any blocking views that are part of the ad (e.g., video player controls, close
buttons, etc.) as friendly obstructions. Providing the container view that contains all of the
elements of the ad is another option.

### What is the purpose of the event owner?

The event owner for impression and video events indicates which layer, native or JavaScript, will be
responsible for signaling these events so it is important to select the right values.

For WebView display, the ad event owner will almost always be native, since the native layer knows
when the WebView will start rendering.

For WebView video, ad event owner and video event owner will generally be JavaScript though there
may be exceptions depending on your specific setup.

For native ad formats, both the ad and video events owner will be native.

### I have only native ad formats. Does the OM SDK use WebViews internally?

The OM SDK does need to create a JavaScript execution environment in order to be able to measure the
ads. On iOS it uses JavaScriptCore and on Android it uses a plain WebView.

### What is the purpose of isolating verification scripts?

This setting is only applicable to HTML (WebView) video. When isolation is enabled measurement
resources will be placed in a sandboxed iframe where they cannot access the video ad element. If
isolation is disabled, they will be placed into a same-origin iframe.

You should isolate if you have strict requirements for isolation of 3rd party code from the video
player. Please note that doing so may place more stringent quality assurance requirements on your
integration.
