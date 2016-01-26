# Implementing a Digit Separator

The digit separator provides developers with a way to format numerical literals
in code. This provides greater readability to large numbers by breaking them
into sub-groups, where a character (the underscore, in this case) acts as a
delineator.

This article will explain how we can add this relatively simple feature into
PHP.

Digression:
> The official RFC for this can be found at: [Number Format Separator]
(https://wiki.php.net/rfc/number_format_separator)


## Digit Separator Syntax

The follow syntactic rules have been made:
 1. Disallow leading underscores
 2. Disallow trailing underscores
 3. Disallow adjacent underscores
 4. Enable underscores between digits only
 5. Enable for arbitrary grouping of digits

(These five rules will be referenced below.)

Thus, the following examples demonstrate invalid usages of the underscore:
```PHP
_100; // invalid as a numerical literal since it's already a valid constant name in PHP
100_; // Parse error: syntax error, unexpected '_' (T_STRING)...
1__000___000_00; // Parse error: syntax error, unexpected '__000___000_00' (T_STRING)...
100_.0; // Parse error:  syntax error, unexpected '_' (T_STRING) in...
100._01; // Parse error:  syntax error, unexpected '_01' (T_STRING) in...
0x_0123; // Parse error:  syntax error, unexpected 'x_0123' (T_STRING) in...
0b_0101; // Parse error:  syntax error, unexpected 'b_0101' (T_STRING) in...
1_e2; // Parse error: syntax error, unexpected '_e2' (T_STRING)...
1e_2; // Parse error: syntax error, unexpected 'e_2' (T_STRING)...
```

And the following examples demonstrate valid usages of the underscore:
```PHP
1_234_567;
123_4567;
1_00_00_00_000;
3.141_592;
0x11_22_33_44_55_66;
0x1122_3344_5566;
0b0010_1101;
0267_3432;
1_123.456_7e2;
```

## The Implementation

Implementing our digit separator will only involve updating the lexer. We will
start by changing the current lexing rules so that numerical literals with
underscores in them are recognised by the lexer. These underscores will then be
stripped so that the numerical literals can be parsed (converted from stringy
numbers to actual numbers) by PHP's internal functions/macros into suitable
numeric types.

### Update the lexing rules

If we execute the following code: 
```
1_000;
```

It will result with:
```
Parse error: syntax error, unexpected '_0' (T_STRING) in Command line code on line 1
```

We are given a nice parse error because the syntax is currently unknown to the
lexer. The lexer attempted to match against all of the numerical literal rules,
but failed to find a match because it does not recognise numbers with
underscores in them. So let's start by updating these rules.

In [Zend/zend_language_scanner.l](http://lxr.php.net/xref/PHP_7_0/Zend/zend_language_scanner.l)
(line ~1100), we have the following rules:
```
LNUM    [0-9]+
DNUM    ([0-9]*"."[0-9]+)|([0-9]+"."[0-9]*)
EXPONENT_DNUM   (({LNUM}|{DNUM})[eE][+-]?{LNUM})
HNUM    "0x"[0-9a-fA-F]+
BNUM    "0b"[01]+
```

The above shows the five basic ways to represent numerics in PHP. The regular
expressions define their syntax, and so these are what we're going to want to
change. Let's update these regular expressions with the five aforementioned
syntax rules in mind for using underscores in numerical literals:
```
LNUM	[0-9]+(_[0-9]+)*
DNUM	(([0-9]+(_[0-9]+)*)*"."([0-9]+(_[0-9]+)*)+)|(([0-9]+(_[0-9]+)*)+"."([0-9]+(_[0-9]+)*)*)
EXPONENT_DNUM   (({LNUM}|{DNUM})[eE][+-]?{LNUM})
HNUM	"0x"[0-9a-fA-F]+(_[0-9a-fA-F]+)*
BNUM	"0b"[01]+(_[01]+)*
```

Looking at the `LNUM` rule, we can see that the regular expression specifies
that there must be at least one leading digit (satisfying rule #1). It then
says that there may be zero or more instances of an underscore and at
least one following digit (satisfying rules #2, #3, and #5). Rule #4 has been
satisfied in the `DNUM`, `EXPONENT_DNUM`, `HNUM`, and `BNUM` rules by
disallowing the underscore to sit adjacently to the **.**, **e**, **0x**, and
**0b**.


Now, compile PHP and execute the following code again:
```PHP
1_000;
```

Result:
```
Parse error: Invalid numeric literal in Command line code on line 1
```

The result is still a parse error, but now it is no longer a syntax error. So
the lexer now sees that it is a numerical literal, but it thinks it's invalid
because the string to integer conversion functions/macros in PHP's internals
(like `ZEND_STRTOL` and `zend_strtod`) cannot parse the stringy numerics with
underscores in. We could update these functions/macros to cater for the new
underscores, but this would change the coercion rules in numerous other parts
of the language that use these.

The simpler solution (that would also maintain backwards compatibility) would
be to strip the underscores in the lexer. This will be the next step.

### Updating the lexing actions

Let's take a look at the current token definition for `BNUM`:
```C
<ST_IN_SCRIPTING>{BNUM} {
	char *bin = yytext + 2; /* Skip "0b" */
	int len = yyleng - 2;
	char *end;

	/* Skip any leading 0s */
	while (*bin == '0') {
		++bin;
		--len;
	}

	if (len < SIZEOF_ZEND_LONG * 8) {
		if (len == 0) {
			ZVAL_LONG(zendlval, 0);
		} else {
			errno = 0;
			ZVAL_LONG(zendlval, ZEND_STRTOL(bin, &end, 2));
			ZEND_ASSERT(!errno && end == yytext + yyleng);
		}
		RETURN_TOKEN(T_LNUMBER);
	} else {
		ZVAL_DOUBLE(zendlval, zend_bin_strtod(bin, (const char **)&end));
		/* errno isn't checked since we allow HUGE_VAL/INF overflow */
		ZEND_ASSERT(end == yytext + yyleng);
		RETURN_TOKEN(T_DNUMBER);
	}
}
```

The `ST_IN_SCRIPTING` mode is the state the lexer is currently in when it
matches the `BNUM` rule, and so it will only match binary numbers when in
normal scripting mode. The code between the curly braces is C code that will be
executed when a binary number is found in the source code.

The `yytext` macro is used to fetch the matched input data, and the `yyleng`
macro is used to hold the length of the matched text. The reason why
they're both macros (that are disguised as variables) rather than the actual
`yytext` and `yyleng` variables is because they have global state.

Digression:
> Variables with global state need the Thread Safe Resource Manager (TSRM) to
> handle the fetching of these global variables when PHP is built in Zend
> Thread Safety (ZTS) mode. For example, the `yytext` macro expands to
> `((char*)SCNG(yy_text))`, where the `SCNG` macro is expanded into
> `LANG_SCNG`, which encapsulates this fetch operation for us. As a consequence
> of thread safety in PHP, you'll see all global variables encapsulated in
> macros so that the TSRM is hidden away behind `#ifdefs` that are resolved at
> precompilation time.

Since the preceding **0b** is not needed when parsing binary literals, the
`bin` variable skips it by equalling to `yytext + 2`. This also decreases the
matched text length by two, hence why `len = yyleng - 2`. From there, we skip
all leading 0's because they are not important, and then the binary number is
either parsed as an integer if it is within the range of a long, or as a double
otherwise.

Let's take a look at the updated `BNUM` token definition to account for
underscores:
```C
<ST_IN_SCRIPTING>{BNUM} {
	/* The +/- 2 skips "0b" */
	int len = yyleng - 2, contains_underscores, i;
	char *end, *bin = yytext + 2;

	/* Skip any leading 0s */
	while (*bin == '0' || *bin == '_') {
		++bin;
		--len;
	}

	for (i = 0; i < len && bin[i] != '_'; ++i);

	contains_underscores = i != len;

	if (contains_underscores) {
		bin = estrndup(bin, len);
		STRIP_UNDERSCORES(bin, len);
	}

	if (len < SIZEOF_ZEND_LONG * 8) {
		if (len == 0) {
			ZVAL_LONG(zendlval, 0);
		} else {
			errno = 0;
			ZVAL_LONG(zendlval, ZEND_STRTOL(bin, &end, 2));
			ZEND_ASSERT(!errno && end == bin + len);
		}
		if (contains_underscores) {
			efree(bin);
		}
		RETURN_TOKEN(T_LNUMBER);
	} else {
		ZVAL_DOUBLE(zendlval, zend_bin_strtod(bin, (const char **)&end));
		/* errno isn't checked since we allow HUGE_VAL/INF overflow */
		ZEND_ASSERT(end == bin + len);
		if (contains_underscores) {
			efree(bin);
		}
		RETURN_TOKEN(T_DNUMBER);
	}
}
```

The new code differs in a few ways from the previous code. Firstly, `0`'s and
`_`'s are now stripped from the beginning of the matched text. This ensures
that binary numbers like the following are properly stripped:
```PHP
0b00_00_11_01;
```

Secondly, we are now checking if there are any underscores present in the
matched text. If there aren't any, then we continue working directly off of
`yytext`. If, however, there are underscores present, then we must perform a
copy of the matched string because `yytext` cannot be directly modified. This
is because files in PHP are loaded in via `mmap`, and so attempting to rewrite
`yytext` (that points to this) will cause a bus error because of insufficient
write permissions for that mmap'ed segment. The string copying is done with the
`estrndup` macro (notice the leading **e** here).

Digression:
> Whenever new memory needs to be allocated, the normal memory allocation
> functions are not used - instead, their counterpart macros (disguised as
> functions) are. These macros (including `emalloc`, `ecalloc`, `efree`, and
> so on) are a part of the Zend Memory Manager (ZendMM). ZendMM handles the
> cleanup of allocated memory when the Zend Engine bails out because of an
> error (like a fatal error) typically by calling `php_error_docref`. It also
> keeps track of the amount of memory allocated for the current request to
> ensure that the amount of allocated memory does not surpass the
> `memory_limit` ini setting.

A new `STRIP_UNDERSCORES` macro has also been added (at the top of the same
file):
```C
#define STRIP_UNDERSCORES(n, len) do { \
	int i, old_len = len; \
	char *new_n, *old_n; \
	for (i = 0, new_n = old_n = n; i < old_len; ++i, ++old_n) { \
		if (*old_n != '_') { \
			*new_n++ = *old_n; \
		} else { \
			--len; \
		} \
	} \
	if (old_len > len) { \
		*new_n = '\0'; \
	} \
} while (0)
```

This simply copies the matched input to the new string copy, ignoring
underscores as it encounters them.

The other small changes made in `BNUM`'s new definition are the replacement of
`yytext` with `bin` and `yyleng` with `len`, as well as the conditional
`efree`ing if the string was copied.

Now compile PHP and try using underscores in binary numbers:
```PHP
var_dump(0b00_00_11_01); // int(13)
```

It works!

Have a go at rewriting the other `LNUM`, `HNUM`, and `DNUM` definitions (they
are similiar to, if not simpler than `BNUM`). To see what they look like,
check out the [changed files of my PR](https://github.com/php/php-src/pull/1699/files).

## Conclusion

We've covered how only a few changes in the lexer rules and definitions are
needed to accept a new syntax in PHP's numerical literals. On the way, we've
also glossed over the TSRM and ZendMM. I hope this has provided a brief, yet
pragmatic overview into PHP's lexer, and I hope to delve further into this
topic in some articles in future.
