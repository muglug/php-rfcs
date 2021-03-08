 * Name: `noreturn-type`
 * Date: 2021-03-14
 * Author: Matt Brown <php@muglug.com>
 * Proposed Version: PHP 8.1
 * RFC PR: [php/php-rfcs#0000](https://github.com/php/php-rfcs/pull/0000)

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
```

PHP code can call this function safe in the knowledge that no statements after it will be evaluated:

```php
function sayHello(?User $user) {
    if (!$user) {
        redirect('/login');
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

In a type-theory, `noreturn` would be called a "bottom" type. That means it's effectively a subtype of every other type in PHP’s type system, including `void`.

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

## Prior art in interpreted languages

- Hacklang has a [noreturn type](https://docs.hhvm.com/hack/built-in-types/noreturn). Slightly confusingly Hacklang also has an explicit bottom type called `nothing` that can be used anywhere `noreturn` can, but also in some other places too, like generics.
- TypeScript has a [never type](https://www.typescriptlang.org/docs/handbook/basic-types.html#never) that's also an explicit bottom type.
- Python added a [NoReturn type](https://docs.python.org/3/library/typing.html#typing.NoReturn) to its typing library.

# Backwards Incompatible Changes

`noreturn` becomes a reserved word.

# Proposed Voting Choices

Simple yes/no vote.
