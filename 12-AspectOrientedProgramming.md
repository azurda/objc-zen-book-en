# Aspect Oriented Programming

Aspect Oriented Programming (AOP) is something not well-known in the Objective-C community but it should be as the runtime is so powerful that AOP should be one of the first things that comes to the mind. Unfortunately, as there is no standard de facto library, nothing comes ready to use out-of-the-box from Apple and the topic is far from being trivial, developers still don't think of it in nowadays. 

Quoting the [Aspect Oriented Programming](http://en.wikipedia.org/wiki/Aspect-oriented_programming) Wikipedia page:

> An aspect can alter the behavior of the base code (the non-aspect part of a program) by applying advice (additional behavior) at various join points (points in a program) specified in a quantification or query called a pointcut (that detects whether a given join point matches).


In the world of Objective-C this means using the runtime features to add *aspects* to specific methods. The additional behaviors given by the aspect can be either:

* add code to be performed before a specific method call on a specific class
* add code to be performed after a specific method call on a specific class
* add code to be performed instead of the original implementation of a specific method call on a specific class

There are many ways to achieve this we are not digging into deep here, basically all of them leverage the power of the runtime. 
[Peter Steinberger](https://twitter.com/steipete) wrote a library, [Aspects](https://github.com/steipete/Aspects) that fits the AOP approach perfectly. We found it reliable and well-designed and we are going to use it here for sake of simplicity.
As said for all the AOP-ish libraries, the library does some cool magic with the runtime, replacing and adding methods (further tricks over the method swizzling technique).
The API of Aspect are interesting and powerful:

```obj-c
+ (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;
- (id<AspectToken>)aspect_hookSelector:(SEL)selector
                      withOptions:(AspectOptions)options
                       usingBlock:(id)block
                            error:(NSError **)error;
```

For instance, the following code will perform the block parameter after the execution of the method `myMethod:` (instance or class method that be) on the class `MyClass`.

```obj-c
[MyClass aspect_hookSelector:@selector(myMethod:)
                 withOptions:AspectPositionAfter
                  usingBlock:^(id<AspectInfo> aspectInfo) {
            ...
        }
                       error:nil];
```

In other words: the code provided in the block parameter will always be executed after each call of the `@selector` parameter on any object of type `MyClass` (or on the class itself if the method is a class method).

We added an aspect on `MyClass` for the method `myMethod:`.

Usually AOP is used to implement cross cutting concern. Perfect example to leverage are analytics or logging.

In the following we will present the use of AOP for analytics. Analytics are a popular "feature" to include in iOS projects, with a huge variety of choices ranging from Google Analytics, Flurry, MixPanel, etc.
Most of them have tutorials describing how to track specific views and events including a few lines of code inside each class.

On Ray Wenderlich's blog there is a long [article](http://www.raywenderlich.com/53459/google-analytics-ios) with some sample code to include in your view controller in order to track an event with [Google Analytics](https://developers.google.com/analytics/devguides/collection/ios/):

```obj-c
- (void)logButtonPress:(UIButton *)button {
    id<GAITracker> tracker = [[GAI sharedInstance] defaultTracker];
    [tracker send:[[GAIDictionaryBuilder createEventWithCategory:@"UX"
                                                          action:@"touch"
                                                           label:[button.titleLabel text]
                                                           value:nil] build]];
}
```

The code above sends an event with context information whenever a button is tapped. Things get worse when you want to track a screen view:

```obj-c
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];

    id<GAITracker> tracker = [[GAI sharedInstance] defaultTracker];
    [tracker set:kGAIScreenName value:@"Stopwatch"];
    [tracker send:[[GAIDictionaryBuilder createAppView] build]];
}
```

This should look like a code smell to the most of the experienced iOS developers. We are actually making the view controller dirty adding lines of code that should not belong there as it's not responsibility of the view controller to track events. You could argue that you usually have a specific object responsible for analytics tracking and you inject this object inside the view controller but the problem is still there and no matter where you hide the tracking logic: you eventually end up inserting some lines of code in the `viewDidAppear:`.

We can use AOP to track screen views on specific `viewDidAppear:` methods, and moreover, we could use the same approach to add event tracking in other methods we are interested in, for instance when the user taps on a button (i.e. trivially calling the corresponding IBAction).

This approach is clean and unobtrusive:

* the view controllers will not get dirty with code that does not naturally belongs to them
* it becomes possible to specify a SPOC file (single point of customization) for all the aspects to add to our code
* the SPOC should be used to add the aspects at the very startup of the app
* if the SPOC file is malformed and at least one selector or class is not recognized, the app will crash at startup (which is cool for our purposes) 
* the team in the company responsible for managing the analytics usually provides a document with the list of *things* to track; this document could then be easily mapped to a SPOC file
* as the logic for the tracking is now abstracted, it becomes possible to scale with a grater number of analytics providers
* for screen views it is enough to specify in the SPOC file the classes involved (the corresponding aspect will be added to the `viewDidAppear:` method), for events it is necessary to specify the selectors. To send both screen views and events, a tracking label and maybe extra meta data are needed to provide extra information (depending on the analytics provider).

We may want a SPOC file similar to the following (also a .plist file would perfectly fit as well):

```obj-c
NSDictionary *analyticsConfiguration()
{
    return @{
        @"trackedScreens" : @[
            @{
                @"class" : @"ZOCMainViewController",
                @"label" : @"Main screen"
                }
             ],
        @"trackedEvents" : @[
            @{
                @"class" : @"ZOCMainViewController",
                @"selector" : @"loginViewFetchedUserInfo:user:",
                @"label" : @"Login with Facebook"
                },
            @{
                @"class" : @"ZOCMainViewController",
                @"selector" : @"loginViewShowingLoggedOutUser:",
                @"label" : @"Logout with Facebook"
                },
            @{
                @"class" : @"ZOCMainViewController",
                @"selector" : @"loginView:handleError:",
                @"label" : @"Login error with Facebook"
                },
            @{
                @"class" : @"ZOCMainViewController",
                @"selector" : @"shareButtonPressed:",
                @"label" : @"Share button"
                }
             ]
    };
}
```

The architecture proposed is hosted on GitHub on the [EF Education First](https://github.com/ef-ctx/JohnnyEnglish/blob/master/CTXUserActivityTrackingManager.m) profile.

```obj-c
- (void)setupWithConfiguration:(NSDictionary *)configuration
{
    // screen views tracking
    for (NSDictionary *trackedScreen in configuration[@"trackedScreens"]) {
        Class clazz = NSClassFromString(trackedScreen[@"class"]);

        [clazz aspect_hookSelector:@selector(viewDidAppear:)
                       withOptions:AspectPositionAfter
                        usingBlock:^(id<AspectInfo> aspectInfo) {
            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
                NSString *viewName = trackedScreen[@"label"];
                [tracker trackScreenHitWithName:viewName];
            });
        }];

    }

    // events tracking
    for (NSDictionary *trackedEvents in configuration[@"trackedEvents"]) {
        Class clazz = NSClassFromString(trackedEvents[@"class"]);
        SEL selektor = NSSelectorFromString(trackedEvents[@"selector"]);

        [clazz aspect_hookSelector:selektor
                       withOptions:AspectPositionAfter
                        usingBlock:^(id<AspectInfo> aspectInfo) {
            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
                UserActivityButtonPressedEvent *buttonPressEvent = [UserActivityButtonPressedEvent eventWithLabel:trackedEvents[@"label"]];
                [tracker trackEvent:buttonPressEvent];
            });
        }];

    }
}
```