## Exploring iOS 14: Crash Reporting using MetricKit
MetricKit was introduced last year with iOS 13, offering a relatively simple API to access a wide range of on-device power and performance metrics for your app. With the introduction of iOS 14, MetricKit added 3 new trackable metrics: CPU instructions, animation hitches, and exit reasons.

Out of those 3 new metrics, the exit reasons metric is the most interesting to me. Exit reasons track the various ways an app could be terminated while running in either the foreground or the background. Those include things like terminations due to memory pressure, watchdog terminations, as well as crashes.

Why is this interesting? Because crash reports on iOS right now can be collected by either using a 3rd party SDK, or relying on Apple's limited Xcode Crashes organizer. This is the first time Apple has offered a solution to build complex crash reporting pipelines with data provided by a 1st-party framework, and without relying on a bunch of messy techniques.

This blog post is an attempt to answer some of the questions that I think are going to be common for anyone looking at MetricKit and comparing it to any of the current crash reporters.

Before getting started, I recommend reading [this blog post](https://www.chimehq.com/blog/metrickit-crash-reporting) by [Matt Massicotte](https://twitter.com/mattie) first. It talks about a lot of the implementation details of crash reporting in MetricKit. Here are some of the key points Matt made that are relevant to this blog post.

* In-process crash reporting, which is how all current commercial and open-source crash reporters are implemented, is messy and error-prone.
* MetricKit offers raw diagnostics data. It’s up to you to do any analysis on that data.
* MetricKit crash reporting is designed to support multiple consumers of the diagnostics data, unlike methods currently used by in-process crash reporting which can leave multiple crash reporters competing for the data.
* Stack traces collected by MetricKit are unsymbolicated and will require further processing before being human-readable.

### How is the data from MetricKit different from Xcode Organizer?
When it comes to crashes, in theory, data shown in Xcode Organizer, should be identical to that returned by MetricKit. The only difference is that data in Xcode Organizer will be presented in a human-readable format and fully symbolicated.

### Should I start collecting crashes using MetricKit in my app?
Probably not. Unless you have a very good reason to build a completely custom crash reporting pipeline (guess what? It’s a lot of work!), you should just rely on crash reports coming from the in-process crash reporter of your choice plus crash reports in Xcode Organizer.

### Who is this for then?
Apple doesn’t specifically say this in the documentation or any of the WWDC videos talking about MetricKit, but I’m presuming that it’s mainly targeted at services offering crash reporting and other performance and diagnostics tools.

### Is this the end of in-process crash reporting?
I highly doubt it. MetricKit is very promising, but it also currently has a few limitations that I think are going to curb its adoption.

MetricKit will only collect data for users that have opted-in to share Diagnostics & Usage data with developers. This is great for privacy, yet it also means that you will not be getting the full picture of how a certain crash impacts your users since you only have a (supposedly small) percentage of the diagnostics reports. More obscure crashes affecting a small percentage of your user base may go completely unnoticed.

Another limitation with MetricKit is that you can request diagnostics data about the previous 24 hours only once per day. This means that [sudden spikes in crashes](https://www.theverge.com/2020/5/7/21250689/facebook-sdk-bug-ios-app-crash-apple-spotify-venmo-tiktok-tinder) may go unnoticed for a few hours too long.

Finally, crash reporting in MetricKit requires iOS 14. Depending on how fast iOS 14 gets adopted by users, this limitation alone could mean that we will have to wait at least 2 to 3 more years before MetricKit can see meaningful levels of adoption.

Apple could address some of those limitations in future versions of MetricKit, but I highly doubt that anybody will be ditching current in-process crash reporters for MetricKit any time soon.

### Is Apple planning on limiting the use of in-process crash reporting?
It’s not entirely implausible that MetricKit could be part of a plan to limit the use of methods currently used by in-process crash reporters and have tighter control over the crash reporting lifecycle and data. It’s very hard to speculate on this, but I doubt that this is what Apple is planning to do.

If this were to happen, it would give Apple the ability to enforce the privacy of user’s data at an API level. While the idea, in theory, sounds appealing, the amount of private data in a crash report is very little, if not non at all.

If Apple was to take such a heavy-handed approach, I believe it would start with SDKs and services that collect a lot more sensitive data, like analytics SDKs, where this would have a much higher impact.

### Conclusion
It’s great to see Apple offering a 1st-party framework for reporting crashes and other diagnostic information that’s designed to be usable directly by an app or by a 3rd-party service.

I think there’s a lot of work that needs to be done for MetricKit crash reporting to fully replace in-process crash reporting, but for now, I think it’s a welcome addition to the several ways we can collect and consume app diagnostic information.

I’m excited to see what other features Apple adds to MetricKit next year, and if they are accompanied by any policy/App Store review changes designed to push developers to use MetricKit instead of alternative solutions.

**Disclaimer: I work for Instabug and we currently offer a crash reporting SDK. My opinions might be biased due to that.**
