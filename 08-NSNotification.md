# NSNotification

When you define your own `NSNotification` you should define your notification's name as a string constant. Like any string constant that you want to make available to other classes, it should be declared as `extern` in the public interface, and defined privately in the corresponding implementation. 
Because you're exposing this symbol in the header you should follow the usual namespace rule prefixing the notification name with the class name that belongs to.
It's also good practice to name the notification using the verb Did/Will and terminate the name with the word "Notifications".

```obj-c
// Foo.h
extern NSString * const ZOCFooDidBecomeBarNotification

// Foo.m
NSString * const ZOCFooDidBecomeBarNotification = @"ZOCFooDidBecomeBarNotification";
```
