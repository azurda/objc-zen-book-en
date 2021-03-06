# Class

## Class name

You should always prefix your class with **three** capital-case letters (two letters are reserved for Apple classes), this looking-weird practice is done to mitigate the notable absence of namespaces in our favorite language.
Some developers don't follow this practice for model objects (we observed this practice especially for Core Data objects), we advice to strictly follow this convention also in the case of Core Data objects because you might end up merging your Managed Object Model with other MOMs, maybe coming from third party library. 
As you may already have noticed, in this book the class (but not only) prefix is `ZOC`.

There is another good practice that you might want to follow while choosing the name for your classes: when you're creating a subclass, you should put the specifying name part between the class prefix and the superclass name. This is better explained with an example: if you have a class named `ZOCNetworkClient`, example of subclass name will be `ZOCTwitterNetworkClient` (note "Twitter" between "ZOC" and "NetworkClient"); or, following the same rule, a `UIViewController` subclass would `ZOCTimelineViewController`.


## Initializer and dealloc

The recommended way to organize the  code is to  have `dealloc` method placed at the top of the implementation (directly after the `@synthesize` and `@dynamic` statements) and the `init` should be placed directly below the `dealloc` methods of any class. In case of multiple initializer the designated initializer should be placed as first method, because is the one where most likely there is more logic, and the secondary initializers should follow.
In these days with ARC, it is less likely that you will need to implement the dealloc method, but still the rationale is that by having the dealloc very close to your init methods, you are visually emphasizing the pairing between the methods. Usually in the dealloc method you will undo things that you have done in the init methods.

`init` methods should be structured like this:

```obj-c
- (instancetype)init
{
    self = [super init]; // call the designated initializer
    if (self) {
        // Custom initialization
    }
    return self;
}
```

It's interesting to understand why do we need to set the value of `self` with the return value of `[super init]` and what happens if we don't do that.

Let's do a step back: we are so used to type expressions like `[[NSObject alloc] init]` that the difference between `alloc` and `init` fades away. A peculiarity of Objective-C  is the so called *two stage creation*. This means that the allocation and the initialization are two separate steps, and therefore two different methods need to be called: `alloc` and `init`.
- `alloc` is responsible for the object allocation. This process involve the allocation of enough memory to hold the object from the application virtual memory, writing the `isa` pointer, initializes the retain count, and zeroing all the instance variables. 
- `init` is responsible for initializing the object, that means brings the object in an usable state. This typically means set the instance variable of an object to reasonable and useful initial values.

The `alloc` method will return a valid instance of an uninitialized object. Every message sent to this instance will be translated into an `objc_msgSend()` call where the parameter named `self` will be the pointer returned by `alloc`; in this way `self` is implicitly available in the scope of every methods. 
To conclude the two step creation the first method sent to a newly allocated instance should, by convention, be an `init` method. Notably the `init` implementation of `NSObject` is not doing more than simply return `self`. 

There is another important part of the contract with `init`: the method can (and should) signal to the caller that it wasn't able to successfully finish the initialization by returning `nil`; the initialization can fail for various reasons such as an input passed in the wrong format or the failure in successfully initialize a needed object.
This is lead us to understand why you should always call `self = [super init]`, if your superclass is stating that it wasn't able to successfully initialize itself, you must assume that you are in an inconsistent state and therefore do not proceed with your own initialization and return `nil` as well in your implementation. If you fail to do so you might end up dealing with an object that is not usable, that will not behave as expected and that might eventually lead to crash your app.

The ability to re-assign `self` can also be exploited by the `init` methods to return a different instance than the one they have been called on. Examples of this behavior are [Class cluster](#class-cluster) or some Cocoa classes that returns the same instance for identical (immutable) objects.

### Designated and Secondary Initializers

Objective-C have the concept of designated and secondary initializers. 
The designated initializer is the initializer that takes the full complement of initialization parameters, the secondary initializers are one or more initializer methods that calls the designated initializer providing one or more default values for the initialization parameter.

```obj-c
@implementation ZOCEvent

- (instancetype)initWithTitle:(NSString *)title
                         date:(NSDate *)date
                     location:(CLLocation *)location
{
    self = [super init];
    if (self) {
        _title    = title;
        _date     = date;
        _location = location;
    }
    return self;
}

- (instancetype)initWithTitle:(NSString *)title
                         date:(NSDate *)date
{
    return [self initWithTitle:title date:date location:nil];
}

- (instancetype)initWithTitle:(NSString *)title
{
    return [self initWithTitle:title date:[NSDate date] location:nil];
}

@end
```

Given the above example `initWithTitle:date:location:` is the designated initializer. The other two init methods are the secondary initializers because they are just calling the designated initializer of the class where they are implemented.

#### Designated Initializer

A class should always have one and only one designated initializer, all other init methods should call the designated one (even though there are an exception to this case).
This dichotomy does not dictate any requirement about which initializer should be called.
It should rather be valid to call any designated initializer in the class hierarchy, and it should be guaranteed that *all* the designated initializer in the class hierarchy are called starting from the furthest ancestor (typically `NSObject`) down to your class. 
Practically speaking this means that the first initialization code executed is the furthest ancestor, and then going down to the class hierarchy; giving to all the classes in the hierarchy the chance to do their specific part of initialization. This totally make sense: you want that everything you inherit from your superclass is in an usable state before doing your actual work.
Even though it isn't explicitly stated, all the classes in Apple's frameworks are guaranteed to respect this contract and your classes should do the same to be sure to be a good citizen and behave correctly and as expected.

There are three different situations that may present when defining a new class:

1. No need to override any initializers
2. Overriding designated initializer
3. Define a new designated initializer

The first case is the most trivial: you don't need to add any specific logic at the initialization of your class you simply rely on you parent designated initializer.
When you want to provide additional initialization logic you can decide to override the designated initializer. You should only override your immediate superclass's designated initializer and be sure that your implementation calls the super of the method you're overriding.  
A typical example is whether you create a `UIViewController` subclass overriding `initWithNibName:bundle:`:

```obj-c
@implementation ZOCViewController

- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    // call to the superclass designated initializer
    self = [super initWithNibName:nibNameOrNil bundle:nibBundleOrNil];
    if (self) {
        // Custom initialization
    }
    return self;
}

@end
```

In case of `UIViewController` subclass it would be an error to override `init` as, in the case that the caller will try to initialize your class by calling `initWithNib:bundle`, your implementation will not be called. This also contradicts the rule that states that it should be valid to call any designated initializer.

In case you want to provide your own designated initializer there are three steps that you need to follow in order to guarantee the correct behavior:

1. Declare your designated initializer, being sure to call your immediate superclass's designated initializer.
2. Override the immediate superclass's designated initializer calling your new designated initializer.
3. Document the new designated initializer. 

Lots of developers often miss the last two steps, this is not only a sign of little care, but in the case of the step two is clearly against the contract with the framework and can lead to very non-deterministic behaviors and bugs.
Let's see an example of the correct way to implement this:

```obj-c
@implementation ZOCNewsViewController

- (id)initWithNews:(ZOCNews *)news
{
    // call to the immediate superclass's designated initializer
    self = [super initWithNibName:nil bundle:nil];
    if (self) {
        _news = news;
    }
    return self;
}

// Override the immediate superclass's designated initializer
- (id)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil
{
    // call the new designated initializer
    return [self initWithNews:nil];
}

@end
```

In case you don't override `initWithNibName:bundle:` and the caller decides to initialize you class with this method (that would be a perfectly valid option) the method `initWithNews:` will never get called and this will bring to an incorrect initialization sequence where the specific initialization logic of your class is not executed.

Even though it should be possible to infer what method is the designate initializer, it is always good to be clear and explicit (the future you or other developers that will work on your code will thank you). There are two strategies (non mutually exclusive) that you can decide to use: the first one you is to clearly state in the documentation which initializer is the designated one, but better yet you can be nice with your compiler and by using the compiler directive `__attribute__((objc_designated_initializer))` you can signal your intent.
Using that directive will also helps the compiler helping you and in fact the compiler will issue a warning if in your new designate initializer you don't call your superclass's designated initializer.
There are, though, cases in which not calling the class designated initializer (and in turn providing the required parameters) and calling another designated initializer in the class hierarchy will bring the class in an useless state. Referring to the previous example, there is no point in instantiating a `ZOCNewsViewController` that should present a news, without the news itself. In this case you can enforce even more the need to call a very specific designated initializer by simply making all the other designated initializers not available. It is possible to do that by using another compiler directive `__attribute__((unavailable("Invoke the designated initializer"))) `, decorating a method with this attribute will make the compiler issuing an error if you try to call this method.

Here the header relative to the implementation of the previous example (note the use of macros to don't repeat the code and being less verbose).

```obj-c

@interface ZOCNewsViewController : UIViewController

- (instancetype)initWithNews:(ZOCNews *)news ZOC_DESIGNATED_INITIALIZER;
- (instancetype)initWithNibName:(NSString *)nibNameOrNil bundle:(NSBundle *)nibBundleOrNil ZOC_UNAVAILABLE_INSTEAD(initWithNews:);
- (instancetype)init ZOC_UNAVAILABLE_INSTEAD(initWithNews:);

@end

```

A corollary of what described above is that you should never call a secondary initializer from within the designated one (if the secondary initializer respects the contract, it will call the designated one). Doing so, the call is very likely to invoke one of the subclass's overridden init methods and it will result in infinite recursion.

There is however an exception to all the rules laid out before that is whether an object conforms to the `NSCoding` protocol and it is initialized through the method `initWithCoder:`.
We should distinguish between the case where the superclass is adopting `NSCoding` and when not. 
In the former case, if you just call `[super initWithCoder:]` you will probably have some shared initialization code with the designated initializer. A good way to handle this is to extract this code in a private method (i.e.  `p_commonInit`).
When your superclass does not adopt `NSCoding` the recommendation is to threat `initWithCoder:` as a secondary initializer and therefore call the `self` designated initializer. Note that this is against what suggested by Apple in the [Archives and Serializations Programming Guide](https://developer.apple.com/library/mac/documentation/cocoa/Conceptual/Archiving/Articles/codingobjects.html#//apple_ref/doc/uid/20000948-BCIHBJDE) where is stated:

> the object should first invoke its superclass's designated initializer to initialize inherited state

following this will actually lead to a non-deterministic behavior that will change if your class is or not a direct subclass of `NSObject`.

#### Secondary Initializer

As stated in the previous paragraph, a secondary initializer is a sort of convenience method to provide default values / behaviors to the designated initializer.
That said, it seems clear that you should not do any mandatory initialization in such method and you should never assume that this method will gets called. Again, the only methods that we are guaranteed to get called are the designated initializer.
This imply that in your designated initializer you should always call another secondary initializer or your `self` designated initializer.  Sometimes, by mistake, one can type `super`; doing this will cause not to respect the aforementioned sequence of initialization (in this specific case by skipping the initialization of the current class).

##### References

- https://developer.apple.com/library/ios/Documentation/General/Conceptual/DevPedia-CocoaCore/ObjectCreation.html
- https://developer.apple.com/library/ios/documentation/General/Conceptual/CocoaEncyclopedia/Initialization/Initialization.html
- https://developer.apple.com/library/ios/Documentation/General/Conceptual/DevPedia-CocoaCore/MultipleInitializers.html
- https://blog.twitter.com/2014/how-to-objective-c-initializer-patterns



### instancetype

One often doesn't realize that Cocoa is full of conventions, and they can help the compiler being a little bit more smart. Whether the compiler encounters `alloc` or `init` methods, it will know that, even though the return type of both method is `id`, those methods will always return objects that are an instance of the receiving class's type. As a consequence, this allows the compiler to provide and enforce the type checking (i.e. check that the methods called on the returned instance are valid). This kind of Clang smartness is due to what is called [related result type](http://clang.llvm.org/docs/LanguageExtensions.html#related-result-types), meaning that 

> messages sent to one of alloc and init methods will have the same static type as the instance of the receiver class

To have more information about the convention that allow to automatically identify the related result type please refer to the [appropriate section]((http://clang.llvm.org/docs/LanguageExtensions.html#related-result-types)) in the Clang Language Extensions guide.
A related result type can be explicitly stated using the `instancetype` keyword as return type and this can be very helpful in situation where a factory method or convenience constructor is used. This will hint the compiler with the correct type checking and, probably more important, will behave correctly also when subclassing.

```obj-c
@interface ZOCPerson
+ (instancetype)personWithName:(NSString *)name;
@end
```

Even though, according to the clang specification, `id` can be promoted to `instancetype` by the compiler. In the case of `alloc` or `init` we strongly encourage to use the return type `instancetype` for all class and instance methods that returns an instance of the class.
This is mostly to form the habit and to be consistent (and possibly having a more readable interface) in all your APIs. Once again, by making small adjustments to your code you can improve the readability: with a simple glance you will be able to distinguish which methods are returning an instance of your class. Sort of details you will appreciate in the long run.

##### Reference
- http://tewha.net/2013/02/why-you-should-use-instancetype-instead-of-id/
- http://tewha.net/2013/01/when-is-id-promoted-to-instancetype/
- http://clang.llvm.org/docs/LanguageExtensions.html#related-result-types
- http://nshipster.com/instancetype/



### Initialization Patterns

#### Class cluster
Class cluster as described in the Apple's documentation is
> an architecture that groups a number of private, concrete subclasses under a public, abstract superclass.

If this description sounds familiar probably your instinct is correct. Class cluster is the Apple lingo for the [Abstract Factory](http://en.wikipedia.org/wiki/Abstract_factory_pattern) design pattern.
The idea with class cluster is very simple: you typically have an abstract class that during the initialization process uses information, generally provided as parameters of the initializer method or available in the environment, to apply a logic and instantiate a concrete subclass. This "public facing" class should internally have a good knowledge of its subclass to be able to return the private subclass that best suited the task.
This pattern is very useful because it removes the complexity of this initialization logic from the caller that only knows about the interface to comunicate with the object, without actually caring about the underlying  implementation.
Class clusters are widely used in the Apple's frameworks; some of the most notably examples are `NSNumber` that can return an appropriate subclass depending of the type of number provided (Integer, Float, etc...) or `NSArray` that will return a concrete subclass with the best storage policy.
The beauty of this pattern is that the caller can be completely unaware of the concrete subclass; in fact it can be used when designing a library to be able to swap the underlaying returned class without leaking any implementation detail as long as is respectful of the contract established in the abstract class.

In our experience the use of a Class Cluster can be very helpful in removing a lot of conditional code.
A typical example of this is when you have the same UIViewController subclass for both iPhone and iPad, but the behavior is slightly different depending on the the device.
The naïve implementation is to put some conditional code checking the device in the methods where the different logic is needed, even though initially the place where this conditional logic is applied may be quite few they naturally tend to grow producing an explosion of code paths.
A better design can be achieved by creating an abstract and generic view controller that will contain all the shared logic and then two specializing subclass for every device. 
The generic view controller will check the current device idiom and depending on that it will return the appropriate subclass.


```obj-c
@implementation ZOCKintsugiPhotoViewController

- (id)initWithPhotos:(NSArray *)photos
{
    if ([self isMemberOfClass:ZOCKintsugiPhotoViewController.class]) {
        self = nil;

        if ([UIDevice isPad]) {
            self = [[ZOCKintsugiPhotoViewController_iPad alloc] initWithPhotos:photos];
        }
        else {
            self = [[ZOCKintsugiPhotoViewController_iPhone alloc] initWithPhotos:photos];
        }
        return self;
    }
    return [super initWithNibName:nil bundle:nil];
}

@end
```

The previous code example show how to create a class cluster. First of all the `[self isMemberOfClass:ZOCKintsugiPhotoViewController.class]` is done to prevent the necessity to override the init method in the subclass in order to prevent an infinite recursion. 
When `[[ZOCKintsugiPhotoViewController alloc] initWithPhotos:photos]` will get called the previous check will be true, the `self = nil` is to remove every reference to the instance of `ZOCKintsugiPhotoViewController` that it will be deallocated , following there is the logic to choose which subclass should be initialized. 
Let's assume that we are running this code on an iPhone and that `ZOCKintsugiPhotoViewController_iPhone` is not overriding `initWithPhotos:`; in this case, when executing `self = [[ZOCKintsugiPhotoViewController_iPhone alloc] initWithPhotos:photos];` the `ZOCKintsugiPhotoViewController` will be called and here is when the first check comes handy, given that now is not exactly  `ZOCKintsugiPhotoViewController` the check will be false calling the `return [super initWithNibName:nil bundle:nil];` this will make continue the initialization following the correct initialization path highlighted in the previous session.

#### Singleton

Generally avoid using them if possible, use dependency injection instead.
Nevertheless, unavoidable singleton objects should use a thread-safe pattern for creating their shared instance. As of GCD, it is possible to use the `dispatch_once()` function to

```obj-c
+ (instancetype)sharedInstance
{
   static id sharedInstance = nil;
   static dispatch_once_t onceToken = 0;
   dispatch_once(&onceToken, ^{
      sharedInstance = [[self alloc] init];
   });
   return sharedInstance;
}
```

The use of dispatch_once(), which is synchronous, replaces the following, yet obsolete, idiom:

```obj-c
+ (instancetype)sharedInstance
{
    static id sharedInstance;
    @synchronized(self) {
        if (sharedInstance == nil) {
            sharedInstance = [[MyClass alloc] init];
        }
    }
    return sharedInstance;
}
```

The benefits of `dispatch_once()` over this are that it's faster, it's also semantically cleaner, because the entire idea of dispatch_once() is "perform something once and only once", which is precisely what we're doing. This approach will also prevent [possible and sometimes prolific crashes][singleton_cocoasamurai].

Classic examples of acceptable singleton objects are the GPS and the accelerometer of a device. Even though Singleton objects can be subclassed, the cases where this comes useful are rare. The interface should put in evidence that the given class is intended to be used as a Singleton. Therefore, often a single public `sharedInstance` class method would suffice and no writable properties should be exposed.

Trying to use a Singleton as a container for objects to share across the code or layer of your application is ugly and nasty, and should be interpreted as a design smell.

[singleton_cocoasamurai]: http://cocoasamurai.blogspot.com/2011/04/singletons-your-doing-them-wrong.html

## Properties

Properties should be named as descriptively as possible, avoiding abbreviation, and camel-case with the leading word being lowercase. Luckily our tool of choice is able to autocomplete everything we type (well... almost everything. Yes, I'm looking at you Xcode's Derived Data) so there is no reason to save a couple of chars, and it's better to convey as much information as possible in your source code.

**For example:**
```obj-c
NSString *text;
```

**Not:**
```obj-c
NSString* text;
NSString * text;
```
(Note: this preference is different for constants. This is actually a matter of common sense and readability: C++ developers often prefer to separate the type from the name of the variable, and as the type in its pure form would be `NSString*` (for objects allocated on the heap, as in C++ it'd be possible to allocate objects on the stack) the format `NSString* text;` is used.)

Use auto-synthesize for properties rather than manual `@synthesize` statements unless your properties are part of a protocol rather than a concrete class. If Xcode can automatically synthesize the variable, then let it do so; moreover it'd be a part of the code that is just redundant and that you have to maintain without a real benefit.

You should always use the setter and getter to access the property, except for the `init` and `dealloc` methods. Generally speaking, using the property gives you an increased visual clue that what you're accessing is outside you current scope and therefor is subject to side-effect.

You should always prefer the setter because:

* Using the setter will respect the defined memory-management semantics (`strong`, `weak`, `copy` etc...). This was definitely more relevant before ARC but it's still relevant; think, for example, the `copy` semantic, every time you use the setter the passed value is copied without any additional operation;
* KVO notifications (`willChangeValueForKey`, `didChangeValueForKey`) are fired automatically;
* It is easier to debug: you can set a breakpoint on the property declaration and the breakpoint will fire every time the getter/setter is called, or you can set a breakpoint inside a custom setter/getter;
* It allows to add extra logic when setting a value in a single place.

You should alway prefer the getter:

* It is resilient to future change (e.g. the property is dynamically generated);
* It allows subclassing;
* It is easier to debug (i.e. Is possible to put a breakpoint in the getter and see who's accessing that specific getter);
* It makes the intent clear and explicit: by accessing an ivar `_anIvar` you are actually accessing `self->_anIvar`. This can lead to problems, for instance, accessing the iVar inside a block (you're capturing and retaining self even if you don't explicitly see the keyword `self`);
* It automatically fires KVO notifications;
* The overhead introduced by the message sending is very low and in the most of the case is negligible. For more information on the performance penalty introduced by the property you may find an interesting performance rundown  [Should I Use a Property or an Instance Variable?](http://blog.bignerdranch.com/4005-should-i-use-a-property-or-an-instance-variable/)

#### Init and Dealloc

There is however an exception to what stated before: you must never use the setter (or the getter) in the `init` (and other initializer method), and instead you should always access directly the variable using the instance variable. This is to be defensive against subclassing: eventually a subclass can override the setter (or getter) and trying to call other methods, access properties or iVars that aren't in a consistent state or fully-initialized. Remember that an object is considered fully initialized and in a consistent state only after the init returns. The same applies for the `dealloc` method (during the `dealloc` method an object can be in a inconsistent state). This is also clearly stated many times over time:

* [Advanced Memory Management Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmPractical.html#//apple_ref/doc/uid/TP40004447-SW6) under the self-explanatory section "Don't Use Accessor Methods in Initializer Methods and dealloc";
* [Migrating to Modern Objective-C](http://adcdownload.apple.com//wwdc_2012/wwdc_2012_session_pdfs/session_413__migrating_to_modern_objectivec.pdf) at WWDC 2012 at slide 27;
* in a [pull request](https://github.com/NYTimes/objective-c-style-guide/issues/6) form Dave DeLong's. 

Moreover, using the setter in the init will not play nicely with `UIAppearence` proxy (please refer to [UIAppearance for Custom Views](http://petersteinberger.com/blog/2013/uiappearance-for-custom-views/) for additional informations).

#### Dot-Notation

When using the setter/getter always prefer the dot notation.
Dot-notation should *always* be used for accessing and mutating properties.

**For example:**
```obj-c
view.backgroundColor = [UIColor orangeColor];
[UIApplication sharedApplication].delegate;
```

**Not:**
```obj-c
[view setBackgroundColor:[UIColor orangeColor]];
UIApplication.sharedApplication.delegate;
```
Using the dot notation will ensure a visual clue to help distinguish between a property or a typical method call.

### Property Declaration

The preferred way to declare a property is the following format

```obj-c
@property (nonatomic, readwrite, copy) NSString *name;
```

The property attributes should be ordered as following: atomicity, read/write and memory-managements storage. By doing this your attributes are more likely to change in the right most position and it's easier to scan with your eyes.

You must use `nonatomic` attribute, unless strictly necessary. On iOS, the locking introduced by `atomic` can significantly affect the performances.

Properties that stores a block, in order to keep it alive after the end of the declaration scope, must be `copy` (the block is initially created on the stack, calling `copy` cause the block to be copied on the heap).

In order to achieve a public getter and a private setter, you can declare the public property as `readonly` and re-declare the same property in the the class extension as `readwrite`:

```obj-c
@interface MyClass : NSObject
@property (nonatomic, readonly) NSObject *object
@end

@implementation MyClass ()
@property (nonatomic, readwrite, strong) NSObject *object
@end
```

If the name of a `BOOL` property is expressed as an adjective, the property can omit the "is" prefix but specifies the conventional name for the get accessor, for example:

```obj-c
@property (assign, getter=isEditable) BOOL editable;
```
Text and example taken from the [Cocoa Naming Guidelines](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingIvarsAndTypes.html#//apple_ref/doc/uid/20001284-BAJGIIJE).

In the implementation file avoid the use of `@synthesize`, Xcode is already adding this for you.

#### Private Properties

Private properties should be declared in class extensions (anonymous categories) in the implementation file of a class. Named categories (e.g. `ZOCPrivate`) should never be used unless extending another class.

**For example:**

```obj-c
@interface ZOCViewController ()
@property (nonatomic, strong) UIView *bannerView;
@end
```

### Mutable Object

Any property that potentially can be set with a mutable object (e.g. `NSString`,`NSArray`,`NSURLRequest`) must have the memory-management type to `copy`. This is done in order to ensure the encapsulation and prevent that the value is changed after the property is set without that the object know it.

You should also avoid to expose mutable object in the public interface, because this allows clients of your class to change your own internal representation and break the encapsulation. You can provide a read-only property that returns an immutable copy of your object:

```obj-c
/* .h */
@property (nonatomic, readonly) NSArray *elements

/* .m */
- (NSArray *)elements {
  return [self.mutableElements copy];
}
```

### Lazy Loading

There are cases when instantiating an object can be expensive and/or needs to be configured once and has some configuration involved that you don't want to clutter the caller method.

In this case, instead of allocating the object in the init method one could opt for overriding the property getter for lazy instantiation. Usually the template for this kind of operation is the following:

```obj-c
- (NSDateFormatter *)dateFormatter {
  if (!_dateFormatter) {
    _dateFormatter = [[NSDateFormatter alloc] init];
        NSLocale *enUSPOSIXLocale = [[NSLocale alloc] initWithLocaleIdentifier:@"en_US_POSIX"];
        [dateFormatter setLocale:enUSPOSIXLocale];
        [dateFormatter setDateFormat:@"yyyy-MM-dd'T'HH:mm:ss.SSSSS"];
  }
  return _dateFormatter;
}
```

Even though this can be beneficial in some circumstance we advice to be thoughtful when you decide to do, and actually this should be avoided. The following are the arguments against the use of property with lazy initialization:

* The getter should not have side effect. By looking a getter on the right-hand side you're not thinking that this is causing an object to be allocated or will lead to side effect. In fact, trying to call a getter without using the returned value the compiler will emit the warning "getter should not used for side effect";
* You're moving the cost of the initialization as side effect of the first access, this will lead to difficulties to optimize performance issues (that are also hard to instrument);
* The moment of instantiation can be non deterministic: for example you're expecting that this property is first accessed by a method but as soon as you change the class implementation, the accessor gets called before your original expectation. This can cause problems especially if the instantiation logic has dependencies on other state of the class that may be different. In general is better to explicitly express the dependency;
* This behavior might not be KVO friendly. If the getter changes the reference it should fire a KVO notification for notify the change, it can be weird to receive a change notification when accessing a getter.


## Methods

### Parameter Assert
Your method may require some parameter to satisfy certain condition (i.e. not to be nil): in such cases it's a good practice to use `NSParameterAssert()` to assert the condition and eventually throwing an exception.

### Private methods

Never prefix your private method with a single underscore `_`, this prefix is reserved by Apple, doing otherwise expose you to the risk of overriding an existing Apple's private method.

## Equality

In case you need to implement equality remember the contract: you need to implement both the `isEqual` and the `hash` methods. If two objects are considered equal through `isEqual`, the `hash` method must return the same value, but if `hash` returns the same value the object are not guaranteed to be equals.

This contracts boils down to how the lookup of those objects is done when are stored in collections (i.e. `NSDictionary` and `NSSet` use hash table data structure underneath).

```obj-c
@implementation ZOCPerson

- (BOOL)isEqual:(id)object {
    if (self == object) {
        return YES;
    }

    if (![object isKindOfClass:[ZOCPerson class]]) {
        return NO;
    }

    // check objects properties (name and birthday) for equality
    ...
    return propertiesMatch;
}

- (NSUInteger)hash {
    return [self.name hash] ^ [self.birthday hash];
}

@end
```

It is important to notice that the hash method must not return a constant. This is a typical error and causes serious problems as it will cause 100% of collisions in the hash table as the value returned by the hash method is actually used as key in the hash table.

You should also always implement a typed equality check method with the following format `isEqualTo<#class-name-without-prefix#>:`
If you can, it's always preferable to call the typed equal method in order to avoid the type checking overhead.

A complete pattern for the isEqual* method should be as so:

```obj-c
- (BOOL)isEqual:(id)object {
    if (self == object) {
      return YES;
    }

    if (![object isKindOfClass:[ZOCPerson class]]) {
      return NO;
    }

    return [self isEqualToPerson:(ZOCPerson *)object];
}

- (BOOL)isEqualToPerson:(Person *)person {
    if (!person) {
        return NO;
    }

    BOOL namesMatch = (!self.name && !person.name) ||
                       [self.name isEqualToString:person.name];
    BOOL birthdaysMatch = (!self.birthday && !person.birthday) ||
                           [self.birthday isEqualToDate:person.birthday];

  return haveEqualNames && haveEqualBirthdays;
}
```

Given an object instance the computation of the `hash` should be deterministic, this is extremely important if the object is added to a container object (i.e. `NSArray`, `NSSet`, or `NSDictionary`) otherwise the behavior will be undefined (all container objects use the object's hash to do the lookup and enforce specific property like the uniqueness of the objects contained). That said, the calculation of the hash should always be made only by using immutable properties or, better yet, by guarantee the immutability of the objects.

