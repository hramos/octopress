--- 
name: drop-in-reachability-check
layout: post
title: Drop In Reachability Check
time: 2010-12-14 15:15:00 -04:00
---

Every iOS application that requires an Internet connection should make use of the Apple [Reachability](http://developer.apple.com/library/ios/#samplecode/Reachability/Introduction/Intro.html) classes. These allow you to easily check if there is a WAN connection available and/or if a specific host can be reached.
<!-- more -->
You never know when your application will run in Airport mode or in a dead area with no cell connection at all. Furthermore, the user might not have 3G service but might be connected through a Wi-Fi hotspot. Perhaps in a future OS update, an additional WAN connection might become available through Bluetooth or a completely new protocol. For these reasons, I can&#8217;t advise you to create your own Reachability code, when Apple&#8217;s will suffice: you only care if there is access to a WAN, or at the very least, if Host X is reachable (who cares about the pathway to it?).

In iPhone OS 2, the Reachability classes were kind of obscure, hidden away in the Apple Sample Code and with no easy way of dropping them in to a project. By default they were synchronous and would need some modification to work properly asynchronously - I don&#8217;t really remember the details, I just remember I had to do this a couple of times and it was very annoying. So I was very happy to find [this answer in Stack Overflow](http://stackoverflow.com/questions/1083701/how-to-check-for-an-active-internet-connection-on-iphone-sdk/3597085#3597085) which provides a step by step to use the updated Reachability classes along with NSNotifications to asynchronously determine reachability.

You should always [visit Stack Overflow to get the latest code and comments](http://stackoverflow.com/questions/1083701/how-to-check-for-an-active-internet-connection-on-iphone-sdk/3597085#3597085), but I&#8217;m saving a local copy here for posterity:

An updated answer:

1) Add `SystemConfiguration` framework to the project but don&#8217;t worry about including it anywhere

2) Add `Reachability.h` and `Reachability.m` to the project

3) Add `@class Reachability;` to the .h file of where you are implementing the code

4) Create a couple instances to check in the `interface` section of the .h file:
{% highlight objectivec %}
Reachability* internetReachable;
Reachability* hostReachable;
{% endhighlight %}	


5) Add a method in the .h for when the network status updates:

{% highlight objectivec %}
- (void)checkNetworkStatus:(NSNotification *)notice;
{% endhighlight %} 
	

6) Add `#import "Reachability.h"` to the .m file where you are implementing the check

7) In the .m file of where you are implementing the check, you can place this in one of the first methods called (`
init` or `viewWillAppear` or `viewDidLoad`, etc.):

{% highlight objectivec %}
- (void)viewWillAppear:(BOOL)animated {
  // check for internet connection
  [[NSNotificationCenter defaultCenter] 
    addObserver:self 
	selector:@selector(checkNetworkStatus:) 
	name:kReachabilityChangedNotification 
	object:nil];

  internetReachable = [[Reachability reachabilityForInternetConnection] retain];
  [internetReachable startNotifier];

  // check if a pathway to your host exists
  hostReachable = [[Reachability reachabilityWithHostName: @"www.hectorramos.com"] retain];
  [hostReachable startNotifier];

  // now patiently wait for the notification
}
{% endhighlight %}
	
	
8) Set up the method for when the notification gets sent and set whatever checks or call whatever methods you may have set up (in my case, I just set a BOOL)

{% highlight objectivec %}
// called after network status changes
- (void)checkNetworkStatus:(NSNotification *)notice {

  NetworkStatus internetStatus = [internetReachable currentReachabilityStatus];
 
  switch (internetStatus) {
    case NotReachable: {
      NSLog(@"The internet is down.");
      self.internetActive = NO;
      break;
    }
    case ReachableViaWiFi: {
      NSLog(@"The internet is working via WIFI.");
      self.internetActive = YES;
      break;
    }
    case ReachableViaWWAN: {
      NSLog(@"The internet is working via WWAN.");
      self.internetActive = YES;
      break;
    }
  }

  NetworkStatus hostStatus = [hostReachable currentReachabilityStatus];
  
  switch (hostStatus) {
    case NotReachable: {
      NSLog(@"A gateway to the host server is down.");
      self.hostActive = NO;
      break;
    }
    case ReachableViaWiFi: {
      NSLog(@"A gateway to the host server is working via WIFI.");
      self.hostActive = YES;
      break;
    }
    case ReachableViaWWAN: {
      NSLog(@"A gateway to the host server is working via WWAN.");
      self.hostActive = YES;
      break;
    }
  }
}
{% endhighlight %}


9) In your `dealloc` or `viewWillDisappear` or similar method, remove yourself as an observer

{% highlight objectivec %}
- (void)viewWillDisappear:(BOOL)animated {
  [[NSNotificationCenter defaultCenter] removeObserver:self];
}
{% endhighlight %}


Note: There might be an instance using `viewWillDisappear` where you receive a memory warning and the observer never gets unregistered so you should account for that as well.
