 * Name: `noreturn-type`
 * Date: 2021-03-14
 * Author: Matt Brown & Ondřej Mirtes
 * Proposed Version: PHP 8.1
 * Implementation: https://github.com/php/php-src/compare/master...muglug:support-noreturn

# Introduction

There has been a trend over the past few years that concepts initially just expressed in PHP docblocks eventually become native PHP types.

Past examples are: [scalar typehints](https://wiki.php.net/rfc/scalar_type_hints_v5), [return types](https://wiki.php.net/rfc/return_types), [union types](https://wiki.php.net/rfc/union_types_v2), [mixed types](https://wiki.php.net/rfc/mixed_type_v2), and [static types](https://wiki.php.net/rfc/static_return_type).

Our static analysis tools currently provide support for `/** @return noreturn */` to denote functions that always `throw` or `exit`. Users of our static analysis tools have found that syntax useful to describe the behaviour of their own code, but we think it’s even more useful as a native return type `: noreturn`, where PHP compile-time and runtime type-checks can guarantee its behaviour.

# Proposal

Introduce a `noreturn` type that can be used in functions that always `throw` or `exit`.

A redirect function is an obvious candidate:

```php
function redirect(string $uri) : noreturn {
    header('Location: ' . $uri);
    exit();
}

function redirectToLoginPage() : noreturn {
    redirect('/login');
}
```

PHP developers can call these functions safe in the knowledge that no statements after the function call will be evaluated:

```php
function sayHello(?User $user) {
    if (!$user) {
        redirectToLoginPage();
    }

    echo 'Hello ' . $user->getName();
}
```

If, at some later date, the redirect function is changed so that it does _sometimes_ return a value, a compile error is produced:

```php
function redirect(string $uri) : noreturn {
    if ($uri === '') {
        return; // Fatal error: A noreturn function must not return
    }
    header('Location: ' . $uri);
    exit();
}
```

If, instead, the above function is rewritten to have an _implicit_ return, a `TypeError` is emitted:

```php
function redirect(string $uri) : noreturn {
    if ($uri !== '') {
        header('Location: ' . $uri);
        exit();
    }
}

redirect(''); // Uncaught TypeError: redirect(): Nothing was expected to be returned
```

`noreturn` function cannot be used as a generator:

```php
function list(string $uri) : noreturn {
    yield 1; // Fatal error: A noreturn function must not be a Generator
    exit();
}
```

## Applicability

Like `void`, the `noreturn` type is only valid when used as a function return type. Using `noreturn` as an argument or property type produces a compile-time error:

```php
class A {
    public noreturn $x; // Fatal error
}
```

## Variance

In type theory `noreturn` would be called a "bottom" type. That means it's effectively a subtype of every other type in PHP’s type system, including `void`.

It obeys the rules you might expect of a universal subtype:

Return type covariance is allowed:

```php
abstract class Person
{
    abstract public function hasAgreedToTerms() : bool;
}

class Kid extends Person
{
    public function hasAgreedToTerms() : noreturn
    {
        throw new \Exception('Kids cannot legally agree to terms');
    }
}
```

Return type contravariance is prohibited:

```php
abstract class Redirector
{
    abstract public function execute() : noreturn;
}

class BadRedirector extends Redirector
{
    public function execute() : void {} // Fatal error
}
```

### Allowed return types when a function always throws

Since `noreturn` is a subtype of all other types, a function that _could_ be annotated with `noreturn` can still safely be annotated with another return type:

```php
function doFoo(): int
{
    throw new \Exception();
}
```

## Prior art in other interpreted languages

- Hacklang has a [noreturn type](https://docs.hhvm.com/hack/built-in-types/noreturn).
- TypeScript has a [never type](https://www.typescriptlang.org/docs/handbook/basic-types.html#never) that's also an explicit bottom type.
- Python added a [NoReturn type](https://docs.python.org/3/library/typing.html#typing.NoReturn) to its typing library.

## Prior art in PHP static analysis tools

In the absence of an explicit return type some PHP static analysis tools have also adopted support for `noreturn` or similar:

- Psalm and PHPStan support the docblock return type `/** @return noreturn */`
- PHPStorm supports a custom PHP 8 attribute `#[JetBrains\PhpStorm\NoReturn]`

## Comparison to void

Both `noreturn` and `void` are both only valid as return types, but there the similarity ends.

When you call a function that returns `void` you generally expect PHP to execute the next statement after that function call.

```php
function sayHello(string $name) : void {
    echo "Hello $name";
}

sayHello('World');
echo ", it’s nice to meet you";
```

But when you call a function that returns `noreturn` you explicitly do not expect PHP to execute whatever statement follows:

```php
function redirect(string $uri) : noreturn {
    header('Location: ' . $uri);
    exit();
}

redirect('/index.html');
echo "this will never be executed!";
```

## Attributes vs types

Some might feel that `noreturn` belongs as a function/method attribute, potentially a root-namespaced one:

```php
#[\NoReturn]
function redirectToLoginPage() : void {...}
```

```php
function redirectToLoginPage() : noreturn {...}
```

We believe it’s more useful as a type. Internally PHP has a much more straightforward interpretation of return types than attributes, and PHP can quickly check variance rules for `noreturn` types just as it does for `void`. It's also tidier.

## Naming

Naming is hard, but we believe `noreturn` is the best name for this type.

Arguments for `noreturn`:

* Very unlikely to be used as an existing class name.
* Describes the behaviour of the function.

Arguments for `never` are:

* It's a single word - `noreturn` does not have any visual separator between the two words and one cannot be sensibly added e.g. `no-return`.
* It's a full-fledged type, rather than a keyword used in a specific situation. A far-in-the-future generics proposal could use `never` as a placeholder inside [contravariant generic types](https://docs.hhvm.com/hack/built-in-types/nothing#usages).

# Backwards Incompatible Changes

`noreturn` becomes a reserved word in PHP 8.1

# Proposed Voting Choices

Yes/no vote for adding `noreturn`
