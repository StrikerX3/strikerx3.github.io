---
layout: post
title:  "Simplify Ternary Operators"
date:   2019-04-15 19:00:26 -0300
categories: c, cpp, csharp, java
---
One of my pet peeves in C/C++/C#/Java programming is the use of the ternary operator `?:` with boolean values. It's almost always redundant, makes code harder to read and can be easily replaced by a simpler boolean expression that also has the advantage of being "branch-free" in some languages, that is, do not require the CPU to select a path of execution based on a condition, which can be fairly expensive. The best compilers will optimize them away in a manner similar to what I describe here, but they still don't improve code readability.

<!-- more -->

From this point onward I'll be using the C/C++/C#/Java code syntax. A boolean expression is any valid expression that results in a boolean value. This may be anything from the constants `true` and `false` all the way to complex expressions such as `x == 1 && (y > 3 || z <= 1.25) && !checkSomething(x, y, z)` (assuming `checkSomething(...)` returns or can be coerced to a boolean).

Say you have the condition `c` and the boolean expressions `p` and `q`. The following table contains all possible conversions from a ternary expression to an equivalent boolean expression:

| # | Ternary Expression  | if-then-else Expression   | Equivalent to           |
|:-:|:--------------------|:--------------------------|:------------------------|
| 1 | `c ? p : q`         | `if (c) p else q`         | `(c && p) || (!c && q)` |
| 2 | `c ? p : true`      | `if (c) p else true`      | `!c || p`               |
| 3 | `c ? p : false`     | `if (c) p else false`     | `c && p`                |
| 4 | `c ? true : p`      | `if (c) true else p`      | `c || p`                |
| 5 | `c ? false : p`     | `if (c) false else p`     | `!c && p`               |
| 6 | `c ? true : false`  | `if (c) true else false`  | `c`                     |
| 7 | `c ? false : true`  | `if (c) false else true`  | `!c`                    |
| 8 | `c ? true : true`   | `if (c) true else true`   | `true`                  |
| 9 | `c ? false : false` | `if (c) false else false` | `false`                 |

Note that #1 may or may not result in more readable code; it is probably better to use the ternary operator in that case. #6 through #9 are poor coding practices and should be replaced as soon as possible by their equivalent boolean expressions. #2 through #5 require attention and almost certainly will result in better, cleaner code.

Also note that `b = c ? p : q` is equivalent to `if (c) b=p else b=q`. In other words, if any boolean variables are assigned a value that depends on the condition of an if-then-else clause, then that value can be simplified according to the equivalence table above, as long as there are no unintended side-effects. Let's take a look at an example:

```java
boolean b = (i > 5) ? (j == 3) : false;
```

or, as an if-then-else clause:

```java
boolean b;
if (i > 5) {
    b = (j == 3);
} else {
    b = false;
}
```

This matches expression #3, with: `c = (i > 5)` and `p = (j == 3)`. The equivalent expression would be:

```java
boolean b = (i > 5) && (j == 3)
```

which is slightly shorter and certainly less confusing than the above ternary or if-then-else statements.

It is interesting to note that certain IDEs (such as IntelliJ IDEA) will suggest simplifications of such overcomplicated ternary expressions:

[![IntelliJ IDEA suggests a simplification of a complicated ternary expression](/assets/3.1-intellij-simplify-ternary.png)](/assets/3.1-intellij-simplify-ternary.png){:target="_blank"}  
**IntelliJ IDEA suggests a simplification of a complicated ternary expression**
{: style="text-align: center;"}


<script type="text/javascript">
amzn_assoc_placement = "adunit0";
amzn_assoc_search_bar = "false";
amzn_assoc_tracking_id = "strikerx3-20";
amzn_assoc_ad_mode = "manual";
amzn_assoc_ad_type = "smart";
amzn_assoc_marketplace = "amazon";
amzn_assoc_region = "US";
amzn_assoc_title = "Great Fundamental Programming Books";
amzn_assoc_linkid = "56f3b31d4652c935c9d79b3444e7a8ee";
amzn_assoc_asins = "0321751043,0262033844,032157351X,0596007124,020161622X,0132350882";
</script>
<script src="//z-na.amazon-adsystem.com/widgets/onejs?MarketPlace=US"></script>
