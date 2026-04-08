# Explainer for the Selective Permissions Intervention

This proposal is shipped in Chrome as of M146.

## Participate
- https://github.com/explainers-by-googlers/selective-permissions-intervention/issues


### Motivation 🤔

When a user grants a website permission to access a powerful API like their precise geolocation, microphone, camera, screen, or clipboard contents, their consent is intended for the site, not necessarily to every third-party script running on the page. In particular, embedded ad scripts can currently leverage the page's permission to opportunistically access this sensitive data. The user may not be aware that an advertisement is accessing their information.

This intervention aims to better align a granted permission with user intent by preventing ad script in a context with API permission from using it, reinforcing user trust and control over their data.


---


### The Intervention 🛡️

This intervention will cause the Permissions Policy checks for the given APIs to be denied when the browser determines that an “ad script” attempted to call the API.


#### Ad Classification

The classification of ads is left to the discretion of the user agent, but the intent is to cover all browsing contexts (e.g., including ad script in the top frame). For example, Chrome detects ads using its [AdTagging](https://chromium.googlesource.com/chromium/src/+/master/docs/ad_tagging.md) feature.


#### Protected APIs

This intent is to block access to APIs that reveal personal information about the user, as opposed to information about their browsing device. This is not an anti-fingerprinting intervention. As such, the initial features considered for this intervention are:

* Geolocation
* Bluetooth
* Microphone
* Camera
* Clipboard
* Display Capture
* USB

#### Actions Taken

When the intervention is triggered, Chrome will perform the following three actions simultaneously:

1. **Deny Permission**: The API request will be automatically denied. For promise-based APIs, the promise will be rejected with a `DOMException` named `'NotAllowedError'`.
2. **Console Issue**: A new console issue will be generated which shows the API that was blocked, the JavaScript stack at the time that it was blocked, and why that script was considered ad related by the the browser.
4. **Send Report**: An intervention report will be sent via the **Reporting API** to any registered reporting endpoints for the frame that initiated the call. This allows site owners to monitor the impact of the intervention on their properties. The fields of the report will look similar to the example console warning.

#### Proposed Specification
Here is the [proposed patch](https://github.com/w3c/webappsec-permissions-policy/pull/572) to Permissions Policy which allows the UA to deny permission policy requests for intervention purposes.

## Security and Privacy Considerations

The intervention is intended to improve user privacy by limiting access to privacy-sensitive APIs. 

One known security issue is that a child frame can test whether the browser considers its context to be ad related, by checking for the intervention. That an iframe is considered ad-related means that somewhere in the ancestor chain, the AdTracker considered the script that loaded the frame to be ad-script. This is an pre-existing issue with the other ads interventions in Chrome such as Heavy Ads, and preventing downloads from ads on the web.

---


### Developer Guidance 💻

This change will affect ads that attempt to use privacy sensitive APIs for features like location-based offers or interactive experiences.


#### How to Detect the Intervention



* **DevTools Console**: The easiest way to check for this intervention during development is to watch for the warning messages in the Chrome DevTools issues pane.
* **Reporting API**: For production monitoring, site owners should configure a reporting endpoint to receive `intervention` reports. The report body will look similar to this: \
```javascript
{
  "type": "intervention",
  "age": 60,
  "url": "https://publisher.example/page",
  "body": {
    "id": "AdPrivacySensitiveApi",
    "message": "Access to the Geolocation API was blocked because it was initiated by an ad script.\nJavaScript stack:\nat callGeolocation (https://adtech.example/adscript.js:1:24)\nat loadAds (https://publisher.example/main.js:123:12)\n\nMatched filterset rule:\n||adtech.example^",
    "sourceFile": "https://adtech.example/adscript.js",
    "lineNumber": 1,
    "columnNumber": 24,
  }
}

```


#### Recommended Solution

Functionality requiring sensitive APIs should be **initiated by the publisher's non-ad scripts**.

If an ad script requires user location, and the publisher consents, the publisher should make the geolocation call and pass that data to the ad script. 

If the script in question is not an ad script (e.g., the call would occur even with an ad blocker enabled) then please file an issue with the browser.

---


### **Chrome-Specific FAQ**

**1. How does Chrome define an "ad script"?** An ad script is any script whose request URL matches the patterns defined in Chrome’s public ad-blocking [filter list](https://github.com/chromium/chromium-ads-detection) or had an ad script on the JavaScript stack at the time that the script was fetched. This is the same mechanism Chrome uses for its [heavy ads intervention](https://developer.chrome.com/blog/heavy-ad-interventions).

**2. What if an ad's functionality legitimately requires a protected API?** The principle of this intervention is that user consent is granted to the top-level site. The site is responsible for brokering access to these powerful features. The ad should request that the publisher's page initiate the API call, rather than calling it directly.

**3. Will this intervention be enabled by default?** Yes, after a gradual rollout to monitor for breakage and gather feedback.


