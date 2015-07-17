# Beautifying the code

### Spacing

* Indent using 4 spaces. Never indent with tabs. Be sure to set this preference in Xcode.
* Method braces and other braces (`if`/`else`/`switch`/`while` etc.) always open on the same line as the statement but close on a new line.

**Preferred:**
```objective-c
if (user.isHappy) {
    //Do something
}
else {
    //Do something else
}
```

**Not Preferred:**
```objective-c
if (user.isHappy)
{
  //Do something
} else {
  //Do something else
}
```

* There should be exactly one blank line between methods to aid in visual clarity and organization. Whitespace within methods should separate functionality, but often there should probably be new methods.
* Prefer using auto-synthesis. But if necessary, `@synthesize` and `@dynamic` should each be declared on new lines in the implementation.
* Colon-aligning method invocation should often be avoided. There are cases where a method signature may have more than 3 colons and colon-aligning makes the code more readable. Always colon align methods, even if they contain blocks.

**Preferred:**

```objective-c
[UIView animateWithDuration:1.0
                 animations:^{
                     // something
                 }
                 completion:^(BOOL finished) {
                     // something
                 }];
```


**Not Preferred:**

```objective-c
[UIView animateWithDuration:1.0 animations:^{
    // something 
} completion:^(BOOL finished) {
    // something
}];
```

If auto indentation falls into bad readability, declare blocks in variables before or reconsider your method signature.

### Line Breaks
Line breaks are an important topic since this style guide is focused for print and online readability.

For example:
```objective-c
self.productsRequest = [[SKProductsRequest alloc] initWithProductIdentifiers:productIdentifiers];
```

A long line of code like the one above should be carried on to the second line adhering to this style guide's Spacing section (two spaces).

```objective-c
self.productsRequest = [[SKProductsRequest alloc] 
  initWithProductIdentifiers:productIdentifiers];
```

### Brackets

Use [Egyptian brackets](https://en.wikipedia.org/wiki/Indent_style#K.26R_style) for:

* control structures (if-else, for, switch)

Non-Egyptian brackets are accepted for:

* class implementations (if any)
* method implementations
