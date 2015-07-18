# Naming

## General conventions

Apple naming conventions should be adhered to wherever possible, especially those related to [memory management rules](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/MemoryMgmt/Articles/MemoryMgmt.html) ([NARC](http://stackoverflow.com/a/2865194/340508)).

Long, descriptive method and variable names are good.

**Preferred:**

```obj-c
UIButton *settingsButton;
```

**Not Preferred:**
```obj-c
UIButton *setBut;
```

## Constants

Constants should be camel-case with all words capitalized and prefixed by the related class name for clarity.

**Preferred:**
```obj-c
static const NSTimeInterval ZOCSignInViewControllerFadeOutAnimationDuration = 0.4;
```

**Not Preferred:**
```obj-c
static const NSTimeInterval fadeOutTime = 0.4;
```

Constants are preferred over in-line string literals or numbers, as they allow for easy reproduction of commonly used variables and can be quickly changed without the need for find and replace. Constants should be declared as `static` constants and not `#define`s unless explicitly being used as a macro.

**Preferred:**
```obj-c
static NSString * const ZOCCacheControllerDidClearCacheNotification = @"ZOCCacheControllerDidClearCacheNotification";
static const CGFloat ZOCImageThumbnailHeight = 50.0f;
```

**Not Preferred:**

```obj-c
#define CompanyName @"Apple Inc."
#define magicNumber 42
```

Constants exposed externally should use this pattern in the interface file:

```obj-c
extern NSString *const ZOCCacheControllerDidClearCacheNotification;
```
and the previously defined assignment should be in the implementation file.

You only need to add the namespace to a constant that is made public. Even though constants used within an implementation could follow a different pattern, there is no need not to be consistent with the rule.


## Methods

In method signatures, there should be a space after the method type (`-`/`+` symbol). There should be a space between the method segments (matching Apple's style).  Always include a keyword and be descriptive with the word before the argument which describes the argument.

The usage of the word "and" is reserved.  It should not be used for multiple parameters as illustrated in the `initWithWidth:height:` example below.

**Preferred:**
```obj-c
- (void)setExampleText:(NSString *)text image:(UIImage *)image;
- (void)sendAction:(SEL)aSelector to:(id)anObject forAllCells:(BOOL)flag;
- (id)viewWithTag:(NSInteger)tag;
- (instancetype)initWithWidth:(CGFloat)width height:(CGFloat)height;
```

**Not Preferred:**

```obj-c
- (void)setT:(NSString *)text i:(UIImage *)image;
- (void)sendAction:(SEL)aSelector :(id)anObject :(BOOL)flag;
- (id)taggedView:(NSInteger)tag;
- (instancetype)initWithWidth:(CGFloat)width andHeight:(CGFloat)height;
- (instancetype)initWith:(int)width and:(int)height;  // Never do this.
```

## Literals

`NSString`, `NSDictionary`, `NSArray`, and `NSNumber` literals should be used whenever creating immutable instances of those objects. Pay special care that `nil` values not be passed into `NSArray` and `NSDictionary` literals, as this will cause a crash.

**For example:**

```obj-c
NSArray *names = @[@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul"];
NSDictionary *productManagers = @{@"iPhone" : @"Kate", @"iPad" : @"Kamal", @"Mobile Web" : @"Bill"};
NSNumber *shouldUseLiterals = @YES;
NSNumber *buildingZIPCode = @10018;
```

**Not:**

```obj-c
NSArray *names = [NSArray arrayWithObjects:@"Brian", @"Matt", @"Chris", @"Alex", @"Steve", @"Paul", nil];
NSDictionary *productManagers = [NSDictionary dictionaryWithObjectsAndKeys: @"Kate", @"iPhone", @"Kamal", @"iPad", @"Bill", @"Mobile Web", nil];
NSNumber *shouldUseLiterals = [NSNumber numberWithBool:YES];
NSNumber *buildingZIPCode = [NSNumber numberWithInteger:10018];
```

For the mutable counterparts of these classes we recommend, instead, the use of the explicit classes such as `NSMutableArray`, `NSMutableString`, and so on. 

The following **should be avoided**:

```obj-c
NSMutableArray *aMutableArray = [@[] mutableCopy];
```

The problems with the previous notation are both of efficiency and readability. 
On the efficiency side, an unnecessarily immutable object is created and immediately thrown away; this unlikely will slow down your app (unless the method here is called frequently) but there is really no reason to do this just to save some characters. 
Regarding the readability, we can see two problems here: the first is that when scanning through the code and seeing `@[]` your mind is immediately connected to and instance of `NSArray`, but in this case you need to stop and check more thoughtfully. Another aspect to take into account is that it would be very likely that someone with less experience will see your code and depending on his background he might not be very comfortable with the dichotomy between mutable and immutable objects. He or she could not be very familiar with the meaning of creating a mutable copy (obviously we are not suggesting that this knowledge is not necessary). Again, this is not something absolutely wrong but is more about code usability (that includes readability).
