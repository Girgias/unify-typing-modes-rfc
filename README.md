# PHP RFC: Unify PHP's typing modes (aka remove strict_types declare)
  * Version: 0.2
  * Date: 2021-02-10
  * Author: George Peter Banyard, <girgias@php.net>
  * Status: Draft
  * First Published at: http://wiki.php.net/rfc/unify-typing-modes

## Introduction

PHP compared to other languages has two typing modes in which it can operate, one which is too lax and the other which is too strict. This RFC describes the reasons for the existence of these two modes, their shortcomings, and what needs to change before unifying both modes again.

## Historical context

The introduction of scalar type declarations into PHP was a contentious issue and was discussed in numerous RFCs.[1][2][3][4][5]

The RFC which ultimately landed this feature is the ["Scalar Type Declarations"][5] one made by Anthony Ferrara which iterated on Andrea Faulds's original RFC and which decided to add the `strict_types` declare.

However, the competing RFC from [Zend Technologies et al][4] proposed to tighten the type coercion rules for function arguments and return values instead. Its changes were the following:

 - No more conversion from boolean to another scalar type
 - Rejecting float values for boolean type declarations
 - Restrict float to integer conversion to floats where no significant digits are present after the decimal point
 - Redefining when a string is considered numeric, and reject appropriately according to the type declaration
 
However, as those were language semantic changes they would first have needed to be deprecated.

The strict typing mode in contrast provided the same benefits while being immediately enforceable, and with the added advantage of having non-nullable type declarations for internal functions.

## Rationale

In the years since the implementation of this RFC, various things have changed within the community, the engine, and the language semantics themselves.

One example is the rise of static analysers which were non-existent at the time, and an assumption made by the RFC [5] about static analysers was that it is impossible for a static analyser to check that a string is numeric as it depends on the value and not the type. While this is true, it is possible to use a static analyser to annotate that a string is numeric or use taint analysis for tracking string data which might or might not be numeric.

As such, revisiting this decision seems appropriate.

### Unintended consequences of strict_types mode

The blind use of the strict typing mode mandated by code styles as lead to some unintended consequences:

 - Use of explicit casts to conform to type requirements even though they are *less* type safe
 - The perceived need for "strict" type casts
 - Manual parsing or type juggling which the engine can already perform
 - The need for the `Stringable` interface

### Advantages of a unified typing mode

The advantages of a unified typing mode have remained the same as stated previously: [4]

 1. Smaller cognitive burden
 2. Too strict may lead to too lax
 3. Smooth integration with data sources

A new advantage which stems from the systematic use of the strict typing mode is:

 4. The need for "namespaced declares" is removed as it is only needed for `strict_types`

It bears to reanalyse these points after having seen how users interact with PHP's strict typing mode.

#### 1. Smaller cognitive burden

Due to the systematic use of the strict typing mode imposed by modern coding styles [6] many users do not understand what the scope of the declare is nor what it does.

Many assume it makes PHP stricter in regards to type juggling when in reality it only affects the passing of scalar inputs to functions called in userland, the scalar return value of custom userland functions, and setting a value to a typed scalar property. [11][12]

It does not prevent types being juggled with the use of operators, nor functions which are called by the engine, even if the function is defined in userland with strict typing enabled. Prime examples of this are engine handlers such as the error, exception, and shutdown handlers.

This means that at all times two sets of typing rules need to be remembered, one too strict and one too lax. The solution is to make the coercive typing mode (the default) more sensible and strict; not using declares to make certain aspects of PHP's type juggling system stricter.

#### 2. Too strict may lead to too lax

The perceived need for so called "strict" type casts is a clear symptom that explicit casts are used in places where they shouldn't be just to comply with the type declaration of a function's parameter. Most notably float to integer and string to integer or float conversions, conversions the engine can perform safely in coercive type mode.

Another issue introduced by the strict typing mode is the need for the `Stringable` interface, as objects which implement a `__toString()` method do not pass a `string` type declaration check even though they possess a string representation. This adds cognitive burden as one needs to either use the `string|Stringable` type declaration for parameters to accept such objects and manually cast them to string within the function, which somewhat defeats the point of the given type declaration, or force the user to type cast to string using `(string)`. Both solutions are suboptimal and depend on the type of code written (libraries for example will prefer the `string|Stringable` approach).

#### 3. Smooth integration with data sources

The previous section already highlights this issue, but it is important to remember that most sources with which PHP will exchange data use strings, e.g. HTTP query parameters or database result sets.

Automatic and sensible conversion in those situations is a desirable feature as it removes the need for manual checks which can be performed by the engine.

This can also be done to parse untrusted user input and fail gracefully by catching the thrown `TypeError` in the following way:

```php
try {
	function_taking_an_int(int $id);
} catch (\TypeError) {
	/* for example log and return a nice error to the user */
}
```


#### 4. No need for "namespaced declares"

This is only tangentially related as a proper definition of a package/module in PHP is still highly desirable, but the driving force for this has mostly been how to apply the `strict_types` declare without needing to repeat oneself, something which is a non-issue with a unified typing mode.

## Reason for keeping the `strict_types` declare

The main reason to keep the `strict_types` declare is to signal to static analysers and IDEs that one doesn't want to use PHP's type juggling.

However, due to it possessing well defined and constrained semantics, IDEs and static analysers must not use this declare statement to expand its scope, for example to show an error when using PHP's type juggling with operators. Being able to expand its semantics via a configuration option would remove the need for additional declare statements which cannot be used on previous versions of PHP. One such proposal is the withdrawn "Strict operator directive" RFC. [13]


## Prerequisites

According to the previous reasons mentioned in favour of the introduction of the `strict_types` declare, the following prerequisite changes need to be done before this proposal can reasonably be enacted:

 - [x] Consistent TypeErrors for internal functions [7]
 - [ ] Consistent TypeErrors when using a non numeric-string as an integer or float
	 - [x] Sensible semantics for what corresponds to a valid numeric string [8]
	 - [ ] Elevate remaining related warnings to TypeErrors 
 - [x] Internal functions must not use nullable types by default [9]
 - [x] No implicit float to integer type coercion if it has a fractional part [10]
 - [ ] No implicit boolean to scalar type coercion
 - [ ] No implicit float to boolean type coercion

## Proposal

Make the `strict_types` declare inoperative but keep the possibility to declare it without generating any warning in the next major version of PHP (tentatively PHP 9.0).


## Backward Incompatible Changes
 
None, as code running under the `strict_types` declare will *always* run under coercive typing rules.

## Proposed PHP Version
Next major version (tentatively PHP 9.0)


## Proposed Voting Choices 
As per the Voting RFC, a yes/no vote with a 2/3 majority is needed for this proposal to be accepted.

## Patches and Tests 
Links to any external patches and tests go here.

If there is no patch, make it clear who will create a patch, or whether a volunteer to help with implementation is needed.

Make it clear if the patch is intended to be the final patch, or is just a prototype.

For changes affecting the core language, you should also provide a patch for the language specification.

## Implementation 
After the project is implemented, this section should contain
  - the version(s) it was merged into
  - a link to the git commit(s)
  - a link to the PHP manual entry for the feature
  - a link to the language specification section (if any)

## Acknowledgements

## References 
[1]: <https://wiki.php.net/rfc/scalar_type_hinting_with_cast> Request for Comments: Scalar Type Hinting With Casts  
[2]: <https://wiki.php.net/rfc/scalar_type_hints_v_0_1> PHP RFC: Scalar Type Hints (Version 0.1)  
[3]: <https://wiki.php.net/rfc/basic_scalar_types> PHP RFC: Basic Scalar Types  
[4]: <https://wiki.php.net/rfc/coercive_sth> PHP RFC: Coercive Types for Function Arguments  
[5]: <https://wiki.php.net/rfc/scalar_type_hints_v5> PHP RFC: Scalar Type Declarations  
[6]: <https://www.php-fig.org/psr/psr-12/> PSR-12 Specification  
[7]: <https://wiki.php.net/rfc/consistent_type_errors> PHP RFC: Consistent type errors for internal functions  
[8]: <https://wiki.php.net/rfc/saner-numeric-strings> PHP RFC: Saner numeric strings  
[9]: <https://wiki.php.net/rfc/deprecate_null_to_scalar_internal_arg> PHP RFC: Deprecate passing null to non-nullable arguments of internal functions  
[10]: <https://wiki.php.net/rfc/implicit-float-int-deprecate> Deprecate implicit non-integer-compatible float to int conversions  
[11]: <https://www.php.net/manual/en/language.types.declarations.php#language.types.declarations.strict> PHP Documentation about the `strict_types` declare   
[12]: <https://wiki.php.net/rfc/typed_properties_v2#strict_and_coercive_typing_modes> "Strict and Coercive Typing Modes" section of the "Typed Properties 2.0" PHP RFC  
[13]: <https://wiki.php.net/rfc/strict_operators> PHP RFC: Strict operators directive  
