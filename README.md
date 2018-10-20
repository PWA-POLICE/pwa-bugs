# PWA Bugs
This is is a general repo for PWA bugs in all browsers, likely most of them will be about iOS (Web.app)

## iOS

Web App Manfest and ServiceWorker API are supported on iOS since 11.3.  
Reference: https://twitter.com/rmondello/status/956256845311590400

### Problem: Sometimes PWA is added in normal mode and not standalone mode (like there is no manifest at all)

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
