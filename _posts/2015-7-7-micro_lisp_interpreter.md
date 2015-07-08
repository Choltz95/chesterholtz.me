---
title: Micro Lisp interpreter and garbage collection
author: Chester Holtz
layout: post
permalink: /blog/post/micro_lisp_interpreter
categories:
  - Uncategorized
---

This past semester I have been rereading Gerald Sussman's popular text: Structure and Interpretation of Computer Programs. I have worked on a lisp interpreter in JavaScript in the past inspired by the implementations of [Peter Norvig][5] and [Mary Rose Cook][6] - one that followed the 10 rules of [John McCarthy][1] detailed in his paper [A Micro-Manual for Lisp - Not the whole Truth][2]. This implementation is what will be discussed in this first blog post. In a future post, I may go into detail regarding my eventual reimplimintation of Micro Lisp and several garbage collection algorithms in C++.

My [Micro Lisp][3] is an interpreter that supports function invocation, lambdas, lets, ifs, numbers, strings, a few JavaScript library functions, and lists. I wrote it over a weekend in about 150 lines of JavaScript, and also included a number of simple and more complex test cases. The code for the project can be found on my [github][3], while one can test a deployed version [here][4]. It is recommended, however, to clone yourself a copy directly from the repository and open index.html in your browser locally.

There are two important parts to consider when writting an interpreter: `parsing` and `evaluation`. When we parse a Lisp expression, we take the code typed by the programmer and transform  it into a representation that we can traverse and evaluate. Evaluation refers to the procedure of processing this structure according to the symantic rules of Lisp and returning a result.

Traditionally, the parsing process is separated into two parts: the `tokenizer` and the `AST assembler`. 

The tokenizer demarcates a string of input characters into `tokens` before passing them on to be assembled into an AST. For Micro Lisp, tokens consist of parentheses, symbols, and numbers.

We tokenize by taking advantage of JavaScript's `replace()` and `split()` functions to take a character string input, add whitespace around each parentheses, and split the result by whitespace to get a JavaScript list of tokens. tokenize() is given below.

<pre class="prettyprint linenums">var tokenize = function(input) {
   return input.replace(/\(/g, ' ( ')
               .replace(/\)/g, ' ) ')
               .trim()
               .split(/\s+/);
 };
</pre>

An AST, or Abstract Syntax Tree is a representation of the structure of code written in a language. The tree is abstract since each node of this tree represents a construct in code, but some elements of the code may be ommitted, i.e. parentheses in our case. Since the inherint syntax structure of a lisp symbolic expression - 'S-Expression' - is representative of an AST, this task is quite simple. An S-Expression can simply be defined inductively as an atom, or an expression (x y) where x and y can be S-Expressions themselves.

Atoms are collections of letters, digits or other characters not otherwise defined in the micro-lisp language. furthermore, lists consist of a left parenthesis followed by a head - or `CAR` - and a tail - a `CDR`. Lists always end with a closing parenthesis.

Parsing a Lisp S-Expression is quite simple. First, parse() is called with a character string input representing the program. to be interpreted. We then tokenize this input to get a list of tokens and pass this list into read_from to assemble the AST. We shift through elements of the list one at a time. If the token at the 0th indice is a '(', we instantiate a list of S-Expressions and recursively add to that list until we encounter a matching ')'. The parse() logic is given below.

<pre class="prettyprint linenums">function parse(input) {
  return read_from(tokenize(input));
}

function read_from(tokens) {
    if (tokens.length == 0) {
        throw 'unexpected EOF';
    }

    var token = tokens.shift();
    if ('(' == token) {
        var L = [];
        while (tokens[0] != ')') {
            L.push(read_from(tokens));
        }
        tokens.shift();
        return L;
    } else {
        return atom(token);
      }
}
</pre>

Below is an example of input, and the resulting output of parsing the Lisp S-Expression (hello (hello world)).

<pre class="prettyprint linenums">>tokenize('(hello (hello world))')
["(", "hello", "(", "hello", "world", ")", ")"]
</pre>

<pre class="prettyprint linenums">>parse('(hello (hello world))')
["hello", ["hello", "world"]]
</pre>

Evaluation is the most complex part of the interpretation process. When we evaluate, we look at an expression and check its value according to the `env` environment - implemented as a JS Dictionary. An environment is simply a mapping from a variable name to its value. 

In code, we impliment a finite number of predefined functions in the global environment as

<pre class="prettyprint linenums">
var Operations = {
'+'       : function(a, b) { return a + b; },
'-'       : function(a, b) { return a - b;},
'*'       : function(a, b) { return a * b; },
'/'       : function(a, b) { return a / b; },
'<'       : function(a, b) { return a < b; },
'>'       : function(a, b) { return a > b; },
'<='      : function(a, b) { return a <= b; },
'>='      : function(a, b) { return a >= b; },
'='       : function(a, b) { return a == b; },
'or'      : function(a,b)  { return a||b;   },
'cons'    : function(a, b) { return [a].concat(b); },
'car'     : function(a)    { return (a.length !==0) ? a[0] : null; },
'cdr'     : function(a)    { return (a.length>1) ? a.slice(1) : null; },
'list'    : function()     { return Array.prototype.slice.call(arguments); },
};
</pre>

and impliment a local environment when the interpreter parses a `lambda` or `def` function.

When we evaluate an expression, we check the expression's function and arguments. If an expression is prefixed by a `'`, or the function being applied is the string QUOTE, we return the expression, or arguments literally and do not evaluate. The forms necessary for a lisp to be considered a Micro Lisp are given as the rules below where expressions are denoted e or a, functions as f and variables as v.

1. QUOTE - The value of (QUOTE A) is A
2. CAR - The value of (CAR e) is the first element of e where e is defined as a non-empty list. i.e. (CAR (QUOTE (A B))) returns A
3. CDR - The value of (CDR e) is the list of remaining elements of e when CAR of e is removed, where e is defined above. ie (CDR (QUOTE (A B))) returns B.
4. CONS - The value of (CONS e1 e2) is the list that results from prefixing e1 onto e2. Thus, (CONS (QUOTE A) (QUOTE B)) returns the list (A B).
5. EQUAL - The value of (EQUAL e1 e2) is true if e1 = e2 and false if otherwise. (EQUAL 1 2) returns false.
6. ATOM - The value of (ATOM e1) is true if e1 is an atom and false if otherwise. (ATOM)
7. COND - The value of (COND(e1 e1) ... (pn en)) is the value of ei, where pi is the the first ofthe p's whose value is not NIL.
8. DEFINE - The DEFINE function maps a variable to the given expression.
9. LAMBDA - Lambda is a construct in the Micro-Lisp language which allows for the definition
of anonymous functions.
10. Operators - We also provide definitions for traditional boolean ( =, >, ≥, <, ≤...) and
arithmetic operators (+, −, ∗, /...)

An example of the case 'DEFINE' is given below:

<pre class="prettyprint linenums">
case "DEFINE":
    [_, variable, exp] = x;
    env.set(variable, evaluate(exp, env));
</pre>

Here, we switch over the first token x\[0\], and assuming that the next token x\[1\] is an atom, we set a new indice in the local environment to the variable name 'variable' and its value to the argument expression (x\[2\])  evaluated with respect to the global enviornment.

With these rules in place, we have a robust and portable lisp that we can use to program anywhere in with a browser. It becomes quite easy to define our own, more complex functions. By querying the help function by typing "sample" into the interpreter prompt, a number of different examples are presented with the most advanced being application of the fibonacci function onto a range of numbers.

<pre class="prettyprint linenums">> (define range (lambda (a b) (cond (= a b) (quote ()) (cons a (range (+ a 1) b)))))
null
> (define map (lambda (f xs) (cond (= xs nil) nil (cons (f (car xs)) (map f (cdr xs))))))
null
> (map (lambda (x) (+ x 1)) (range 0 10))
(1 2 3 4 5 6 7 8 9 10 null)
> (map (lambda (x) (fac x)) (range 0 10))
(1 1 2 6 24 120 720 5040 40320 362880 null)
> (define fib (lambda (n) (cond (or (= n 0) (= n 1)) 1 (+ (fib (- n 1)) (fib (- n 2))))))
null
> (map (lambda (x) (fib x)) (range 0 10))
(1 1 2 3 5 8 13 21 34 55 null)</pre>

Originally I had intended to also discuss preforming garbage collection on list based system in this post and implementing a turtle graphics module utilizing HTML5 canvas, but I think I will visit those topics at a later time. 

[1]: https://en.wikipedia.org/wiki/John_McCarthy_(computer_scientist)
[2]: http://www.cse.sc.edu/~mgv/csce330f13/micromanualLISP.pdf
[3]: http://github.com/Choltz95/microlispjs
[4]: http://littlelispjs.divshot.io/
[5]: https://en.wikipedia.org/wiki/Peter_Norvig
[6]: https://github.com/maryrosecook
