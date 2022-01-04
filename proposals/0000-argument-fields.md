# Argument Fields

* Proposal: [HXP-0000](0000-argument-fields.md)
* Author: [wartman](https://github.com/wartman)

## Introduction

Introduce argument fields to remove some of the boilerplate commonly found in constructors. 

## Motivation

> Note: The name "argument fields" is based on "parameter properties" from [TypeScript](https://www.typescriptlang.org/docs/handbook/2/classes.html#parameter-properties), just using "argument" and "field" as those seem to be the Haxe terms. Another name might be better. 

A huge proportion of constructor arguments in Haxe just consist of setting a class field. For example, this is a pretty common pattern:

```haxe
class Foo {
  public final value:String;

  public function new(value) {
    this.value = value;
  }
}
```

This proposal takes inspiration from [TypeScript](https://www.typescriptlang.org/docs/handbook/2/classes.html#parameter-properties) (and many other languages), and would allow the user to declare class fields in the constructor:

```haxe
class Foo {
  public function new(public final value:String) {}
}
```

## Detailed design

Argument fields are just syntax sugar -- the following two examples are equivalent:

```haxe
class Foo {
  public function new(
    public final value:String,
    var bar:Int = 0
  ) {}
}

// ...is the same as:

class Foo {
  public final value:String;
  var bar:Int;
  
  public function new(value, bar = 0) {
    this.value = value;
    this.bar = bar;
  }
}
```

Any argument that is prefixed with `var` or `final` will be converted into a field, and they may also optionally be `public` or `private` (`static` is not allowed). Methods and properties can not be declared here -- only variable fields are allowed. Argument fields and normal arguments can be mixed together and used in any order.

```haxe
class FooBar {
  final bar:String;

  public function new(
    public final foo:String,
    bar,
    private var fooBar:String = 'foobar'
  ) {
    this.bar = bar;
  }
}
```

Metadata will be attached to the _field_ and not the argument, so the following:

```haxe
class Foo {
  public function new(
    @fooable
    public final value:String
  ) {}
}
```

...will desugar to:

```haxe
class Foo {
  @fooable
  public final value:String

  public function new(value) {
    this.value = value;
  }
}
```

Applying metadata to the argument will simply require not using an argument field.

```haxe
class Foo {
  public final value:String

  public function new(@fooable value) {
    this.value = value;
  }
}
```

Initializers (if present) are treated as if they're on the _argument_, not the field.

```haxe
class FooBar {
  public function new(var fooBar:String = 'foobar') {}
}

// becomes:

class FooBar {
  var fooBar:String; 

  public function new(foobar = 'foobar') {
    this.foobar = foobar;
  }
}
```

The same rules for declaring fields apply to argument fields, so re-declaring a field that exists isn't allowed -- including for super-classes.

```haxe
class Foo {
  public function new(public final foo:String) {}
}

// This is fine:
class FooBar extends Foo {
  public function (public final bar:String, foo) {
    super(foo);
  }
}

// ...but this will fail:
class FooBar extends Foo {
  public function (
    public final bar:String,
    public final foo:String
  ) {}
}
```

Finally, it's important to note that argument fields may ONLY be used on class constructors. Anywhere else (including abstract types) will result in compiler errors.

```haxe
class Foo {
  // This will fail:
  public function setFoo(final foo:String) {}
}
```

> Note: Currently, the error will be something like "unexpected final", but we might change it to something like "Argument fields are only allowed in class constructors".

## Impact on existing code

This is new syntax and should have no impact on existing code.

## Drawbacks

There could be some confusion about what the names in argument fields point to. See [Unresolved questions](#Unresolved-questions) for a discussion on this.

## Alternatives

Instead of declaring fields in constructors, we could just introduce a shorthand to set the field. For example:

```haxe
class Foo {
  public final foo:String;

  public function new(this.foo) {}
}

// desugars to:

class Foo {
  public final foo:String;

  public function new(foo) {
    this.foo = foo;
  }
}
```

## Unresolved questions

In the following example, what is `value`: the constructor argument or the class field?

```haxe
class Foo {
  public function new(public var value:String) {
    value = value + 'bar';
  }
}

var foo = new Foo('foo');
trace(foo.value); // Is this "foo" or "foobar"?
```

In the current proposal, `foo.value` would be `"foo"`, as the code is desugaring to:

```haxe
class Foo {
  public var value:String;

  public function new(value) {
    this.value = value;
    value = value + 'bar';
  }
}

var foo = new Foo('foo');
trace(foo.value); // => "foo"
```

I'm not sure if this behavior feels obvious and I could see people get frustrated by it, so something like mangling the argument name might be needed (although this adds its own complexity -- the signature of `Foo.new` changing to something like `(value_1:String)->Foo` would feel pretty hacky, so the solution won't be quite that straightforward).
