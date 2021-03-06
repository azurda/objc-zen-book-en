# Categories

It is ugly, we know, but categories should always be prefixed with your lower case prefix and an underscore i.e. `- (id)zoc_myCategoryMethod`. This practice is also [recommended by Apple](https://developer.apple.com/library/ios/documentation/cocoa/conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html#//apple_ref/doc/uid/TP40011210-CH6-SW4).

This is absolutely needed because implementing a category method with a name already existing in the extended object or in another category may result in an undefined behavior. Practically, what is going to happen is that the implementation of the last category loaded will be the one that gets called. 

In case you want to be sure that you're not replacing any implementation with your own category you can set the environment variable `OBJC_PRINT_REPLACED_METHODS` to `YES`, this will print in the console the names of the methods that have been replaced. 
At the time of writing LLVM 5.1 does not emit any warning or error for this, so be careful and don't override methods in categories.

A good practice is to use prefix also for category names.

**Example:**

```obj-c
@interface NSDate (ZOCTimeExtensions)
- (NSString *)zoc_timeAgoShort;
@end
```

**Not:**

```obj-c
@interface NSDate (ZOCTimeExtensions)
- (NSString *)timeAgoShort;
@end
```

Category can be used to group related method in a header file. This is a very common practice in Apple's framework (nearby is proposed an extract from `NSDate` header) and we strongly encourage to do the same in your code. 
In our experience creating this groups can be helpful in further refactoring: when the interface of a class starts growing can be a signal that your class is doing to much and therefore violating the Single Responsibility Principle, the previously created groups be used to better understand the different responsibilities and help in breaking down the class in more self-contained components.

```obj-c

@interface NSDate : NSObject <NSCopying, NSSecureCoding>

@property (readonly) NSTimeInterval timeIntervalSinceReferenceDate;

@end

@interface NSDate (NSDateCreation)

+ (instancetype)date;
+ (instancetype)dateWithTimeIntervalSinceNow:(NSTimeInterval)secs;
+ (instancetype)dateWithTimeIntervalSinceReferenceDate:(NSTimeInterval)ti;
+ (instancetype)dateWithTimeIntervalSince1970:(NSTimeInterval)secs;
+ (instancetype)dateWithTimeInterval:(NSTimeInterval)secsToBeAdded sinceDate:(NSDate *)date;
// ...
@end
```