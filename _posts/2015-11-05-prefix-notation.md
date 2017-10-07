---
layout: post
title: A Concise Prefix Notation Calculator in Python
---

Suppose we want to write a calculator that can evaluate math expressions of the form

```scheme
(operator argument1 argument2)
```

where each argument is either a number or a math expression of the same form.

For example, `(+ 1 2)` evaluates to 3, as does `(- (+ 4 1) 2)`.
We say an expression like this is written in **prefix notation**.
If you've worked in Lisp before, you know what this is.

The typical process of parsing prefix notation involves breaking the input into tokens, assembling an abstract syntax tree (AST), then evaluating using recursion. This works very well, and can easily be extended to include more powerful Lisp-y features like environments and mutation.

Peter Norvig's classic ["How to Write a Lisp Interpreter in Python"](http://norvig.com/lispy.html) uses this method. The main parsing procedure converts an expression into a nested list:

```python
>>> parse('(+ 1 (- 2 3))')
['+', 1, ['-', 2, 3]]
```

This list is then fed to the `eval` procedure. It evaluates an expression by first evaluating the arguments recursively, then calling the operator on them:

```python
>>> eval(['+', 1, ['-', 2, 3]])
0
```

(The operator is also evaluated. In the example above, the addition symbol `'+'` is eventually converted into Python's `operator.add`.)

This process works well for evaluating prefix notation expressions in general. But suppose we aren't writing a full-blown Lisp. Can we be more simple?

## Regex to the rescue

[Regular expressions](https://en.wikipedia.org/wiki/Regular_expression) are great.
We can use them to search and match for patterns in strings.

How would we match a flat (non-nested) prefix notation expression like `(+ 1 2)` with regex?
Here's one possibility:

```
\([-+\/\*] \d+ \d+\)
```

This regex says that the operator must be either `+`, `-`, `*`, or `/`, and that each argument must be some sequence of digits (0&ndash;9). The entire expression must be wrapped in a set of parentheses.

How do we evaluate? We can modify the regex to capture the operator and arguments, and then we can rearrange them to form a typical (infix notation) Python expression, which we can pass to Python's `eval`.
So to evaluate `(+ 1 2)`, we would end up calling `eval('1 + 2')`.

Here's how that looks:

```python
import re

def calc_eval(exp):
    m = re.match(r'\(([-+\/\*]) (\d+) (\d+)\)', exp)
    if m:
        return eval('%s %s %s' % (m.group(2), m.group(1), m.group(3)))
    raise SyntaxError('Not well formed')
```

Let's see this in action:

```python
>>> calc_eval('(+ 1 2)')
3
>>> calc_eval('(- 5 12)')
-7
>>> calc_eval('/ 12 5')  # Forgot parens
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 5, in calc_eval
SyntaxError: Not well formed
```

Great! We can now evaluate any prefix notation expression that does not contain nested subexpressions.

## Recursive pattern-matching

But what about expressions with "depth" like `(+ (- 1 2) 3)`? We need to be able to find expressions *inside* of other expressions.

How would we extend the regex above to allow one layer of nesting, e.g. `(+ (+ 1 2) (+ 3 4))`? Perhaps we could replace each argument (`\d+`) with a copy of the entire regex, like this:

<!-- Hardcoded so we can color the two internal groups -->
<div class="highlighter-rouge">
<div class="highlight">
<pre class="highlight">
<code>\([-+\/\*] <span style="color:blue">\([-+\/\*] \d+ \d+\)</span> <span style="color:red">\([-+\/\*] \d+ \d+\)</span>\)
</code></pre></div></div>


But this gets messy fast. Besides, we still wouldn't be able to match *arbitrarily* nested expressions.

Suppose we could use `(?R)` in our regex to indicate a **recursive match**. A substring matches `(?R)` if and only if it also matches the regex as a whole.

For example, consider the following regex:

```
happy(?R)?
```

This regex says, "Find the string `happy`, possibly followed by a string that also matches the regex `happy(?R)?`." So it would match `happy`, but it would also match `happyhappy`, or `happyhappyhappy`, or `happyhappyhappyhappy`, and so on.

With `(?R)`, our regex is no longer used just once, from left to right&mdash;we can now match patterns *recursively*.

Now, returning to our original problem, let's use `(?R)` to match nested prefix notation expressions:

```
\([-+\/\*] (?R) (?R)\)|\d+
```

This says that a valid prefix notation expression is either a number (`\d+`) or a function application, in which case we have a math operator followed by two arguments, each of which is also a valid prefix notation expression.

Note that the `|\d+` is very important. It lets us accept integers as valid input expressions, and it serves as the "base case" in our recursion: without it, our calculator would only accept expressions that are endlessly nested.

Okay, great. Now it would be nice if this `(?R)` feature existed.
Here's the good news: it does!
It's called a [recursive pattern](http://www.rexegg.com/regex-recursion.html), and it works out-of-the-box with certain programming languages. The bad news is that Python isn't one of them. Instead of using `re`, we'll need the `regex` module ([doc](https://pypi.python.org/pypi/regex)), which we can install with `pip install regex`.

We're now ready to write our recursive prefix notation calculator.
We'll use the regular expression we just wrote, but we'll throw in some parens to capture the tokens we want.
Our code looks very similar to what we had before, except we now have to recursively evaluate function arguments:

```python
import regex

def calc_eval(exp):
    m = regex.match(r'\(([-+\/\*]) ((?R)) ((?R))\)|(\d+)', exp)
    if m.group(1) and m.group(2) and m.group(3):  # exp is a procedure call
        return eval('%s %s %s' % (calc_eval(m.group(2)),
                                  m.group(1),
                                  calc_eval(m.group(3))))
    if m.group(4):  # exp is a number
        return str(eval(m.group(4)))
    raise SyntaxError('Not well formed')
```

Let's see how it fares&hellip;
```python
>>> calc_eval('(+ 1 2)')  # Make sure this still works
3
>>> calc_eval('(+ (/ 100 (+ 50 50)) (- 5 3))')  # Nested expressions!
3
```

We've done it! We can now evaluate prefix notation expressions with arbitrarily nested subexpressions, and we're doing it cleanly and concisely.
Pretty cool, right?

## Our final code

If we care less about error-handling, we can get our
`calc_eval` procedure down to just five lines, while still being somewhat readable:

```python
def calc_eval(exp):
    m = regex.match(r'\(([-+\/\*]) ((?R)) ((?R))\)|(\d+)|[-+\/\*]', exp)
    if all(map(m.group, [1, 2, 3])):  # exp is a procedure call
        return eval(' '.join([str(calc_eval(m.group(i))) for i in [2, 1, 3]]))
    return eval(exp) if m.group(4) else exp  # exp is a number / an operator
```

The complete version of the code (which includes a REPL) can be found on [GitHub Gist](https://gist.github.com/guoguo12/dfa2ed858821aa98da57).
