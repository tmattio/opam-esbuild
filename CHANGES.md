## 0.14.34 - 2022-04-07

* Fix a regression regarding `super` ([#2158](https://github.com/evanw/esbuild/issues/2158))

    This fixes a regression from the previous release regarding classes with a super class, a private member, and a static field in the scenario where the static field needs to be lowered but where private members are supported by the configured target environment. In this scenario, esbuild could incorrectly inject the instance field initializers that use `this` into the constructor before the call to `super()`, which is invalid. This problem has now been fixed (notice that `this` is now used after `super()` instead of before):

    ```js
    // Original code
    class Foo extends Object {
      static FOO;
      constructor() {
        super();
      }
      #foo;
    }

    // Old output (with --bundle)
    var _foo;
    var Foo = class extends Object {
      constructor() {
        __privateAdd(this, _foo, void 0);
        super();
      }
    };
    _foo = new WeakMap();
    __publicField(Foo, "FOO");

    // New output (with --bundle)
    var _foo;
    var Foo = class extends Object {
      constructor() {
        super();
        __privateAdd(this, _foo, void 0);
      }
    };
    _foo = new WeakMap();
    __publicField(Foo, "FOO");
    ```

    During parsing, esbuild scans the class and makes certain decisions about the class such as whether to lower all static fields, whether to lower each private member, or whether calls to `super()` need to be tracked and adjusted. Previously esbuild made two passes through the class members to compute this information. However, with the new `super()` call lowering logic added in the previous release, we now need three passes to capture the whole dependency chain for this case: 1) lowering static fields requires 2) lowering private members which requires 3) adjusting `super()` calls.

    The reason lowering static fields requires lowering private members is because lowering static fields moves their initializers outside of the class body, where they can't access private members anymore. Consider this code:

    ```js
    class Foo {
      get #foo() {}
      static bar = new Foo().#foo
    }
    ```

    We can't just lower static fields without also lowering private members, since that causes a syntax error:

    ```js
    class Foo {
      get #foo() {}
    }
    Foo.bar = new Foo().#foo;
    ```

    And the reason lowering private members requires adjusting `super()` calls is because the injected private member initializers use `this`, which is only accessible after `super()` calls in the constructor.

* Fix an issue with `--keep-names` not keeping some names ([#2149](https://github.com/evanw/esbuild/issues/2149))

    This release fixes a regression with `--keep-names` from version 0.14.26. PR [#2062](https://github.com/evanw/esbuild/pull/2062) attempted to remove superfluous calls to the `__name` helper function by omitting calls of the form `__name(foo, "foo")` where the name of the symbol in the first argument is equal to the string in the second argument. This was assuming that the initializer for the symbol would automatically be assigned the expected `.name` property by the JavaScript VM, which turned out to be an incorrect assumption. To fix the regression, this PR has been reverted.

    The assumption is true in many cases but isn't true when the initializer is moved into another automatically-generated variable, which can sometimes be necessary during the various syntax transformations that esbuild does. For example, consider the following code:

    ```js
    class Foo {
      static get #foo() { return Foo.name }
      static get foo() { return this.#foo }
    }
    let Bar = Foo
    Foo = { name: 'Bar' }
    console.log(Foo.name, Bar.name)
    ```

    This code should print `Bar Foo`. With `--keep-names --target=es6` that code is lowered by esbuild into the following code (omitting the helper function definitions for brevity):

    ```js
    var _foo, foo_get;
    const _Foo = class {
      static get foo() {
        return __privateGet(this, _foo, foo_get);
      }
    };
    let Foo = _Foo;
    __name(Foo, "Foo");
    _foo = new WeakSet();
    foo_get = /* @__PURE__ */ __name(function() {
      return _Foo.name;
    }, "#foo");
    __privateAdd(Foo, _foo);
    let Bar = Foo;
    Foo = { name: "Bar" };
    console.log(Foo.name, Bar.name);
    ```

    The injection of the automatically-generated `_Foo` variable is necessary to preserve the semantics of the captured `Foo` binding for methods defined within the class body, even when the definition needs to be moved outside of the class body during code transformation. Due to a JavaScript quirk, this binding is immutable and does not change even if `Foo` is later reassigned. The PR that was reverted was incorrectly removing the call to `__name(Foo, "Foo")`, which turned out to be necessary after all in this case.

* Print some large integers using hexadecimal when minifying ([#2162](https://github.com/evanw/esbuild/issues/2162))

    When `--minify` is active, esbuild will now use one fewer byte to represent certain large integers:

    ```js
    // Original code
    x = 123456787654321;

    // Old output (with --minify)
    x=123456787654321;

    // New output (with --minify)
    x=0x704885f926b1;
    ```

    This works because a hexadecimal representation can be shorter than a decimal representation starting at around 10<sup>12</sup> and above.

    _This optimization made me realize that there's probably an opportunity to optimize printed numbers for smaller gzipped size instead of or in addition to just optimizing for minimal uncompressed byte count. The gzip algorithm does better with repetitive sequences, so for example `0xFFFFFFFF` is probably a better representation than `4294967295` even though the byte counts are the same. As far as I know, no JavaScript minifier does this optimization yet. I don't know enough about how gzip works to know if this is a good idea or what the right metric for this might be._

* Add Linux ARM64 support for Deno ([#2156](https://github.com/evanw/esbuild/issues/2156))

    This release adds Linux ARM64 support to esbuild's [Deno](https://deno.land/) API implementation, which allows esbuild to be used with Deno on a Raspberry Pi.
