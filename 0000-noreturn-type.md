 * Name: `noreturn-type`
 * Date: 2021-03-14
 * Author: Matt Brown <php@muglug.com>
 * Proposed Version: PHP 8.1
 * Implementation: https://github.com/php/php-src/compare/master...muglug:support-noreturn

# Introduction

The "noreturn" type is designed to be used by functions that never return a value.

# Proposal

Introduce a `noreturn` type that can be used in functions that never return a value.

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

PHP code can call this function safe in the knowledge that no statements after it will be evaluated:

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

## Applicability

Like `void`, the `noreturn` type is only valid when used as a function return type. Using `noreturn` as an argument or property type produces a compile-time error:

```php
class A {
    public noreturn $x; // Fatal error
}
```

## Variance

In type-theory `noreturn` would be called a "bottom" type. That means it's effectively a subtype of every other type in PHP’s type system, including `void`.

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

## Prior art in other interpreted languages

- Hacklang has a [noreturn type](https://docs.hhvm.com/hack/built-in-types/noreturn).
- TypeScript has a [never type](https://www.typescriptlang.org/docs/handbook/basic-types.html#never) that's also an explicit bottom type.
- Python added a [NoReturn type](https://docs.python.org/3/library/typing.html#typing.NoReturn) to its typing library.

## Prior art in PHP static analysis tools

In the absence of an explicit return type some PHP static analysis tools have also adopted support for `noreturn` or similar:

- Psalm and PHPStan support the docblock return type `/** @return noreturn */`
- PHPStorm supports a custom PHP 8 attribute `#[JetBrains\PhpStorm\NoReturn]`tt

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

I believe it’s more useful as a type. Internally PHP has a much more straightforward interpretation of return types than attributes, and PHP can quickly check variance rules for `noreturn` types just as it does for `void`. It's also just _neater_.

## Naming

Naming is hard, but I believe `noreturn` is the best name for this type.

Two other alternatives, `never` and `nothing`, are much more likely to already be used as class names in existing PHP projects.

# Backwards Incompatible Changes

`noreturn` becomes a reserved word in PHP 8.1

# Proposed Voting Choices

Yes/no vote for adding `noreturn`
