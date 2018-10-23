# PWA Bugs
This is is a general repo for PWA bugs in all browsers, likely most of them will be about iOS (Web.app)

- [iOS Safari](#ios-safari)
  * [Problem: in iOS 12 cache in Cache Storage magically disappears](#problem--in-ios-12-cache-in-cache-storage-magically-disappears)
  * [Problem: Cookie/Login isn't shared between Safari and standalone mode](#problem--cookie-login-isn-t-shared-between-safari-and-standalone-mode)
  * [Problem: Cross domain authorization in standalone mode doesn't work](#problem--cross-domain-authorization-in-standalone-mode-doesn-t-work)
  * [Problem: Sometimes PWA is added in normal mode and not standalone mode](#problem--sometimes-pwa-is-added-in-normal-mode-and-not-standalone-mode)
  * [Problem: PWA is added without a splashscreen](#problem--pwa-is-added-without-a-splashscreen)
  * [Problem: Navigation to a website has infinite loading](#problem--navigation-to-a-website-has-infinite-loading)
  * [Problem: Push Notifications are not supported](#problem--push-notifications-are-not-supported)


## iOS Safari

Web App Manfest and ServiceWorker API are supported on iOS since 11.3.  
Reference: https://twitter.com/rmondello/status/956256845311590400

### Problem: in iOS 12 cache in Cache Storage magically disappears

WebKit Bug: https://bugs.webkit.org/show_bug.cgi?id=190269

**Solution:**

This bug makes Cache Storage persist only until user closes Safari (or it's unloaded from memory).
Basically, the cache is some temporary place and is erased when Safari is closed.
There is really no workaround around this. It causes this situation.

- User visits a website
- ServiceWorker caches everything
- User closes Safari
- User visits a website again while offline
- Nothing loads

It also affects websites in standalone mode.
**This bug also prevents Safari from sharing the cache between standalone mode and normal Safari mode.**


### Problem: Cookie/Login isn't shared between Safari and standalone mode

Problem is in all iOS version.

**Solution:**

At the time of writing (iOS 12.0.1), there is only couple version of Safari where the workaround works.

The workaround is to transfer a session through Cache Storage API, which has shared cache in some Safari versions.  
Approximately how the workaround works:

- User visits a website
- User opens the Share Popup to add website to the homescreen
- When popup opens, Safari sends `destination='manifeset'` request to the server
- ServiceWorker intercepts the requests and send another to the server to generate temporary token
- ServiceWorker stores the token in special cache in Cache Storage API
- User taps "Add to Homescreen"
- User opens website from the homescreen
- ServiceWorker intercepts the opening and send another request to the server with the saved token
- Server checks the token and authorizes the user
- ServiceWorker allows navigation to finish
- User is automatically authorized in standalone mode

Now gotchas.

It works only in iOS 11.4.1 though. It's broken in iOS 12 and iOS 12.0.1, but supposed to be fixed in the next version.
WebKit Bug for iOS 12: https://bugs.webkit.org/show_bug.cgi?id=190269

Even though ServiceWorker was introduces in iOS 11.3, it's unknown if anything prior 11.4.1 can support this.
It either just doesn't work in iOS Emulator at all or doesn't work prioer 11.4.1 (latest 11 emulator version is 11.4.0).

Tested on real devices with these versions: 
- 11.4.1 -- works
- 12.0.0 -- doesn't work
- 12.0.1 -- doesn't work

### Problem: Cross domain authorization in standalone mode doesn't work

This issue happens since 11.3 when Web App Manifest is used.  
iOS requires `scope` property in Web App Manifest to determine which URLs allowed to be opened in standalone mode and which are not. When user website navigates to any URL outside of the scope (this includes any URL with different domain) -- it drops the navigation from standalone mode and opens it in Safari. This makes cross-domain authorization impossible.

**Solution:**

Use the iframe. 

Let's suppose there is a foo.com website and its login is on login.foo.com  
Here is approximately how this can be solved:

- Authorization request is sent to `foo.com`
- ServiceWorker intercepts this request and cloned request to the server, with `redirect: 'manual'`
- ServiceWorker tells the page to create `<iframe>` and request a fake url
- ServiceWorker responds to the `<iframe>` fake url request with the response from syntetic request from the step 2
- `<ifarme>` handles the response. The response must have a redirect 
- ServiceWorker intercepts the redirect from `<iframe>` and send another cloned request with `redirect: 'manual'`
- If type of the response is `basic`, the send the response to original authorization from the step 1
- If type of the response is `opaqueredirect`, repeat the steps from step 3

### Problem: Sometimes PWA is added in normal mode and not standalone mode
_(like there is no manifest at all)_

Reference: https://twitter.com/nekrtemplar/status/1040282198044291072

Safari fetches the manifest only when user is about to add the website to homescreen.
To be preceice, manifest is fetched exactly when Share Popup is opened.

So if user performs all the steps very fast: 1) Open popup; 2) Tap "Add to Home Screen"; 3) Tap "Add" on the adding popup;
Then in some cases the manifest might be fetched yet, so website will be added like there is no manifest.

**Solution:**

Thankfully, when Safari fetches the manifest,
that request goes to the ServiceWorker with `request.destination` being `manifest`, as per spec.
This means that it's possible to intercept such request and return manifest contents from the cache.
So the solution would be to precache the manifest on ServiceWorker `install` and serve it to the browser from the cache.


### Problem: PWA is added without a splashscreen

Even if all correct splashscreen elements are added to the page, after adding the website to the home screen,
it may not have the splashscreen. 

**Solution:**

When website is added only with Web App Manifest, Safari doesn't not take splashscreen elements into the account. The solution would be adding `<meta name="apple-mobile-web-app-capable" content="yes">` element to the page alongside with the manifest.

_**Note:** It might be better adding `apple-mobile-web-app-capable` dynamically and only when in browser mode.
When in standalone mode, it may affects URLs behavior since Web App Manifest installed PWA and `apple-mobile-web-app-capable`
installed PWA have different URL behavior on iOS._

### Problem: Navigation to a website has infinite loading

If update of a ServiceWorker happens when there is a page navigation in-flight and ServiceWorker `install` event resolves
sooner than the navigation page is finished loading, then Safari kills old ServiceWorker immediately even without calling `self.skipWaiting()` and that break loading on the page and hence it never loads.

**Solution:**

Check for navigation requests when `install` event happens and do not resolve the promise if there are any. When page loads, send message to the installing ServiceWorker that it may try again to resolve the installation promise (check again for navigation request, repeat all the steps until no navigation requests, then resolve the promise).

### Problem: Push Notifications are not supported

Both Push subscription (Push API) and displaying notifications (Notifications API) are not supported in iOS. There is no pushManager in ServiceWorker registration object - so don't forget to do a feature detection in your code.

**Solution:**

There is no good workaround/polyfill as this feature fully depends on the 3rd party Messaging Service infrastructure (Apple Push Notifications Service in that case) which is not supporting the standartized (VAPID, Push API, Notifications API) way to subscribe for/receive/display notifications. There are no signals from Apple/Safari about this feature is under consideration/development. There is a Twitter thread with discussing the possible solution to have a kind of notifications on iOS: https://twitter.com/webmaxru/status/1034882612978966528. Some outcomes:
- For Safari on Mac you can set up Push notificatious using a proprietary implementation: https://developer.apple.com/notifications/safari-push-notifications/
- The working (but cumbersome) solution is to wrap your PWA app into a native iOS one using, for example, Cordova.
- The closest (but still not a "real") pure web solution is to set up a PassBook that can receive push. But this way your notifications are disconnected from the app itself.
