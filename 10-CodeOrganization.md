# Code Organization

[Quoting](http://nshipster.com/pragma/) Mattt Thompson

> code organization is a matter of hygiene

We could not agree more. Having you code clearly organized in a clean and defined manner is a way to show respect for yourself and other people that will read and change you code (consider the future you included).

## Exploit Code Block

A very obscure GCC behavior that it is also supported by Clang is the ability of a code block to return the value of the latest statement if enclosing in round bracket.

```obj-c
NSURL *url = ({
    NSString *urlString = [NSString stringWithFormat:@"%@/%@", baseURLString, endpoint];
    [NSURL URLWithString:urlString];
});
```
This feature can be nicely organized to group small chunk of code that usually are necessary only for the sole purpose of setting up a class.
This gives a reader an important visual clue and help reduce the visual noise focusing on the most important variable in the function.
Additionally, this technique has the advantage that all the variables declared inside the code block, as one might expect, are valid only inside that scope, this mean that the your not polluting the method's stack trace and you can reuse the variable name without having duplicated symbols.

## Pragma 

### Pragma Mark

`#pragma mark -` is a great way to organize the code inside a class and helps you grouping methods implementation.
We suggest to use `#pragma mark -`to separate:

- methods in functional groupings 
- protocols implementations.
- methods overridden from a superclass


```obj-c

- (void)dealloc { /* ... */ }
- (instancetype)init { /* ... */ }

#pragma mark - View Lifecycle

- (void)viewDidLoad { /* ... */ }
- (void)viewWillAppear:(BOOL)animated { /* ... */ }
- (void)didReceiveMemoryWarning { /* ... */ }

#pragma mark - Custom Accessors

- (void)setCustomProperty:(id)value { /* ... */ }
- (id)customProperty { /* ... */ }

#pragma mark - IBActions

- (IBAction)submitData:(id)sender { /* ... */ }

#pragma mark - Public

- (void)publicMethod { /* ... */ }

#pragma mark - Private

- (void)zoc_privateMethod { /* ... */ }

#pragma mark - UITableViewDataSource

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath { /* ... */ }

#pragma mark - ZOCSuperclass

// ... overridden methods from ZOCSuperclass

#pragma mark - NSObject

- (NSString *)description { /* ... */ }

```

The above marks will help to visually separate and organize the code. One of the pros is that you can cmd+click on the mark to jump to the symbol definition.
Be aware that even though the use of pragma mark is a sign of craftsmanship, it's not a good reason to make grow the number of methods in your class in a unbounded fashion: having too many of them should be a warning sign that your class has too many responsibilities and a good opportunity for refactoring.


### Notes about pragma

At http://raptureinvenice.com/pragmas-arent-just-for-marks there's a great discussion about pragmas, reported in part here.

While most iOS developers don't play around much with compiler options, some options are useful to control how strictly to check (or not check) your code for errors. Sometimes, though, you want to make an exception directly in your code using a pragma, which purpose is to temporarily disable a compiler behavior.

When you use ARC, the compiler inserts memory-management calls for you. There are cases, though, where it can get confused. One such case is when you use `NSSelectorFromString` to have a dynamically-named selector called. Since ARC can't know what the method will be and what kind of memory management to use, you'll be warned with `performSelector may cause a leak because its selector is unknown`.

If you know your code won't leak, you can suppress the warning for just this instance by wrapping it like this:

```obj-c
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"

[myObj performSelector:mySelector withObject:name];

#pragma clang diagnostic pop
```

Note how we disable the -Warc-performSelector-leaks check by pushing and popping the change around our code. This assures us we don't disable it globally, which would be a huge mistake.

The entire list of options that can be enable or disabled can be found at [The Clang User's Manual](http://clang.llvm.org/docs/UsersManual.html) to learn about all of them.

Suppressing warnings for unused variables

It's useful to be told that a variable you've defined is going unused. In most cases, you want to remove these references to improve performance (however slightly), but sometimes you want to keep them. Why? Perhaps they have a future usage or the functionality is only temporarily removed. Either way, a smarter way to suppress the warning without brutally commenting out the relevant lines, is to use the `#pragma unused()`:

```obj-c
- (void)giveMeFive
{
    NSString *foo;
    #pragma unused (foo)

    return 5;
}
```

Now you can keep your code in place without the compiler complaining about it. And yes, that pragma needs to go below the offending code.

## Explicit warnings and errors

The compiler is a robot: it will mark what's wrong with your code using a set of rules that've been defined by Clang. But, every so often you're smarter than it. Often, you might find some offending code that you know will lead to problems but, for whatever reason, can't fix yourself at the moment. You can explicitly signal errors like this:

```obj-c
- (NSInteger)divide:(NSInteger)dividend by:(NSInteger)divisor
{
    #error Whoa, buddy, you need to check for zero here!
    return (dividend / divisor);
}
```

You can signal warnings similarly:

```obj-c
- (float)divide:(float)dividend by:(float)divisor
{
    #warning Dude, don't compare floating point numbers like this!
    if (divisor != 0.0) {
        return (dividend / divisor);
    }
    else {
        return NAN;
    }
}
```

## Docstrings

All non-trivial methods, interfaces, categories, and protocol declarations should have accompanying comments describing their purpose and how they fit into the larger picture. For more examples, see the Google Style Guide around [File and Declaration Comments](http://google-styleguide.googlecode.com/svn/trunk/objcguide.xml#File_Comments).

To summarize: There are two types of docstrings, long-form and short-form.

A short-form docstring fits entirely on one line, including the comment slashes. It is used for simple functions, especially (though by no means exclusively) ones that are not part of a public API:

```
// Return a user-readable form of a Frobnozz, html-escaped.
```

Note that the text is specified as an action ("return") rather than a description ("returns").

If the description spills past one line, you should move to the long-form docstring: a summary line (one physical line) preceded by an opening block comment with two asterisks on a line of its own (/\*\*, terminated by a period, question mark, or exclamation point, followed by a blank line, followed by the rest of the doc string starting at the same cursor position as the first quote of the first line, ending with an end-block comment (\*/) on a line by itself.

```
/**
 This comment serves to demonstrate the format of a docstring.

 Note that the summary line is always at most one line long, and
 after the opening block comment, and each line of text is preceded
 by a single space.
*/
```

A function must have a docstring unless it meets all of the following criteria:

* not externally visible
* very short
* obvious

The docstring should describe the function's calling syntax and its semantics, not its implementation.

## Comments

When they are needed, comments should be used to explain why a particular piece of code does something. Any comments that are used must be kept up-to-date or deleted.

Block comments should generally be avoided, as code should be as self-documenting as possible, with only the need for intermittent, few-line explanations. *Exception: This does not apply to those comments used to generate documentation.*

### Header Documentation

The documentation of class should be done using the Doxygen/AppleDoc syntax only in the .h files when possible. Documentation should be provided for methods and properties.

**For example:**

```obj-c
/**
 *  Designated initializer.
 *
 *  @param  store  The store for CRUD operations.
 *  @param  searchService The search service used to query the store.
 *
 *  @return A ZOCCRUDOperationsStore object.
 */
- (instancetype)initWithOperationsStore:(id<ZOCGenericStoreProtocol>)store
                          searchService:(id<ZOCGenericSearchServiceProtocol>)searchService;
```