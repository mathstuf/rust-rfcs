- Feature Name: `skippable_tests`
- Start Date: 2021-05-02
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Some test cases have indeterminite results in some cases. These cases should be
distinct from a passing or failing state of a test today. It is possible to
turn tests off at compile time by using features determined by a `build.rs`
script, however, this does not work well for cross compilation or when the
build and test environment are otherwise quite different.

# Motivation
[motivation]: #motivation

Currently, tests in Rust's `libtest` have only a handful of end results:

  - success
  - failure (panic)
  - should panic (which must panic to succeed)
  - ignored (masked by default)

While this works for the vast majority of cases, there are situations where
tests must change their behavior based on the available runtime. While many of
these cases can be detected at build time, this is just not the case in
general. Cases that I, personally, have come across in various projects (Rust
and non-Rust):

  - *Cross-compilation.*: In cross-compilation, build-time detection of the
    target's feature set is generally not possible in any kind of reliable way
    (including all of the following items).
  - *Complicated dependency specifications*: Tests which require specific
    things may be hard to test at configure or build time. Even if the list of
    things is possible to get, ensuring that when those things change that the
    build is redone to sync up with the prevailing environment is another
    problem altogether. This includes all of the following items, but is
    certainly not limited to them.
  - *Kernel version/feature detection*: While possible at configure time,
    adding a dependency that is meaningful to ensure that the test suite is
    recompiled when the running kernel version changes is an unnecessary
    burden.
  - *OpenGL support*: Detecting whether the runtime OpenGL is suitable for
    testing usually involves creating an OpenGL context and querying for its
    limitations. Again, adding a dependency for "OpenGL limitations" is a
    burden. Doing this during the configure or `build.rs` step involves
    unnecessary complications. Many build systems do not even support running
    such code at configure time without building and running code itself (cf.
    CMake, autotools, etc.). Even if the build system is in a language which
    can make OpenGL contexts, it unnecessarily restricts what machines can
    build the project and its test suite.
  - *Python module detection*: This can also change external to the build
    system. Versions changing, availability in the given environment, etc. can
    all change externally. I've found it far easier to instead detect the
    module at test time and decide what to do at that point.
  - *Complex test requirements*: Kind of a grab bag of cases here. Generally,
    some tests may depend on some state of the enclosing build as to whether
    they are suitable or not. Requiring configure or build time detection of
    such things is either difficult or error-prone (as the detection code must
    be kept in-sync with the test itself). While it doesn't happen in Rust that
    often, in C++, some projects have examples which require various bits of
    the project to be built in order to actually work. Keeping the build system
    in the loop of what states of the project's build is suitable for each test
    is not easy when it is far easier for the test to just have a mechanism
    that says "this test is not suitable, please skip me".

As an example within one of my own crates, [this
test][rust-keyutils-invalidate] detects whether the remaining tests in that
module make sense. The best it can do is inform that the rest of the tests in
that module are meaningless (or have them report "success" when they really
didn't do anything of the sort).

[rust-keyutils-invalidate]: https://github.com/mathstuf/rust-keyutils/blob/e4b35b614af249bf1fbec7a9d2c0a662009c2b01/src/tests/invalidate.rs#L32-L41

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There may be cases where the results of a test are not useful in the current
environment. Usually, this can be detected at build time and a `cfg` be enabled
to change test function attributes or hide entire modules. However, some tests
are sensitive to states of the environment that may change at any time over the
test executable's life. Some examples include:

  - Is the running user an administrator or not?
  - Is there enough memory available for this test to succeed?
  - Are there enough CPU cores available?
  - Can a suitable OpenGL context be made for this test?
  - Is an X server available?
  - Is a hardware token available?

In such cases, it is possible to add a `skip_if` attribute to tests to indicate
when a test should be considered to have an indeterminate result in the current
execution environment. The `predicate` item may be set to the path to a
function which takes no arguments and returns an `Option<String>`. When this
function returns `None`, the test will be executed while if it returns `Some`,
the string value will be used as the reason the test could not be executed.

As an example, a test which ensures that removing permissions from a file has
the desired effect in the rest of the code generally will not work as intended
when executing as root because removing read permissions from a file does not
actually change anything in this case:

```rust
fn is_root() -> Option<String> {
    if libc::getuid() == 0 {
        Some("untestable when running as root".into())
    } else {
        None
    }
}

#[test]
#[skip_if(predicate = is_root)]
fn test_revoke_permissions() {
    // remove permissions from a file and make sure it is not readable.
}
```

While it would be possible to detect root permissions at build time, this would
compile the test out completely even though it could be expected to work if a
non-root user ran the test.

The `skip_if` attribute supports a single argument, which must be supplied:

  - `predicate`: the path to a function with a `fn() -> Option<String>`
    signature.

If the `predicate` function panics, the test is considered to have failed
whether or not `should_panic` is provided on the test function or not.

Currently `skip_if` is only available for `#[test]` and `#[bench]` functions,
not within doctests.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of this is largely in `libtest`. The `TestDescAndFn` gains a
new field `skipfn` which, if provided, is run before the test. If it returns a
reason, a new test result is constructed and handled appropriately in the
output formatters. If the skipping function panics, the test is considered to
have failed regardless of `#[should_panic]` or not.

The implementation takes the same panic-wrapping approach as the test itself
does, but instead of handling panics and returning a result for `should_panic`
to decide, panics are just treated as a hard failure.

I have a work-in-progress implementation [on my fork][mathstuf-rust-fork-mvp]
which implements it for `#[test]` and `#[bench]` functions. Adding support for
doctests is left out for now. In addition, a few corner cases are left as
"TODO" items, largely around what if the `predicate` is not suitable, error
messages, and parsing the attribute into a callable..

[mathstuf-rust-fork-mvp]: https://github.com/mathstuf/rust/tree/skippable-test-functions

# Drawbacks
[drawbacks]: #drawbacks

This does make testing more complicated, but I don't know of another way of
handling such dynamic requirements reliably from `build.rs`.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

I think this design keeps itself out of the way for existing test suites which
don't need to deal with this. For tests suites that do, it is easy to add a
single attribute to the affected tests and implement a quick check. The cost of
not doing this is that these test suites continue to have to do largely
unsuitable or inaccurate checks in `build.rs` or report "fake" success reports
for tests that aren't actually doing what they should due to environmental
states.

Prior art usually also has mechanisms where skipping is supported from within
the test. Rust does not have a throwing mechanism (other than `panic!` which is
already covered via the `#[should_panic]` attribute) and adding a return path
seems excessive given that the "happy path" would then also need to "return"
some kind of indication that the test ran fine.

In addition, using an attribute to call another function makes it easier to
handle when a collection of tests all need to skip for the same reason. Using
an attribute allows them all to share the same code without adding anything
which might constitute "noise" to the test function body itself.

# Prior art
[prior-art]: #prior-art

Other test harnesses support the concept of "skipping" tests. This is used as a
state that is neither success nor failure and is usually used to indicate that
a test is not applicable in the current environment. It is used as a separate
state because without it, the test must either report success without actually
testing what it purports to test or report failure and prevent "all green"
states which is typically wanted within development workflows.

## CMake

[CMake][cmake], a very popular C++ build system (full disclosure, the RFC
author is a CMake developer), supports this in its CTest tool which is a test
harness runner (though it supports running commands rather than test functions
directly). In CTest, tests can have "test properties" named
[`SKIP_RETURN_CODE`][ctest-skip-return-code] and
[`SKIP_REGULAR_EXPRESSION`][ctest-skip-regexp] which will cause the test to be
skipped if it exits with the indicated return code or its output matches the
given regular expression. This state is not reported as success or failure but
as a third state of "skipped" (rendered on [CDash][cdash] in a "Not Run"
column).

```cmake
add_test(NAME skip COMMAND …)
set_test_properties(skip PROPERTIES
  SKIP_RETURN_CODE 125
  SKIP_REGULAR_EXPRESSION "SKIPME")
```

```c
#include <stdio.h>

int main(int argc, char* argv[]) {
    printf("SKIPME\n"); // Will cause the test to be skipped.
    return 125; // As will this; either is sufficient, but both are available.
}
```

Given the nature of CTest running arbitrary commands, there's no good way to
indicate *why* a test was skipped. It is generally left to the test to print
out its reasoning to stdout or stderr where the content will be preserved (by
CTest) and displayed (when uploaded to CDash).

[cmake]: https://cmake.org
[ctest-skip-return-code]: https://cmake.org/cmake/help/latest/prop_test/SKIP_RETURN_CODE.html
[ctest-skip-regexp]: https://cmake.org/cmake/help/latest/prop_test/SKIP_REGULAR_EXPRESSION.html
[cdash]: https://cdash.org

## Pytest

The [pytest][pytest] framework is available for Python. Here, there are [a few
ways][pytest-skip-test-functions] to indicate that a test should be skipped.

  - Static Declarative: The `pytest.mark.skip` decorator which (optionally)
    accepts a `reason` keyword argument. This argument contains a description
    of why the test cannot be run.
  - Dynamic Declarative: The `pytest.mark.skipif` takes a boolean as its only
    required argument. If this condition is `True`, the test is skipped (this
    additionally takes a `reason` argument).
  - Imperative: The `pytest.skip` function may be called (with a required
    `reason` positional argument). This is used for more conditions which
    require runtime information in order to detect their suitability. This is
    also usable at the module level as well with the `allow_module_level=True`
    argument.

```python
@pytest.mark.skip(reason="I wrote this on a Monday")
def test_garfield_static():
    pass

@pytest.mark.skipif(time.localtime().tm_wday == 0, reason="Mondays, am I right?")
def test_garfield_dynamic():
    pass

def test_garfield_imperative():
    if time.localtime().tm_wday == 0:
        pytest.skip(reason="Mondays, am I right?")
```

Both declarative methods may be used at the class (group of test) level as
well.

[pytest]: https://docs.pytest.org
[pytest-skip-test-functions]: https://docs.pytest.org/en/6.2.x/skipping.html

## RSpec

Ruby's [RSpec][rspec] testing library [supports skipping tests][rspec-skip] via
a similar mechanism to the dynamic declarative and imperative modes available
in `pytest`. They are reported as "Pending" in this case.

```ruby
RSpec.describe "an example" do
  example "is skipped dynamic declaratively", :skip => true do
  end

  example "is skipped dynamic declaratively with a reason", :skip => "blue moon" do
  end

  it "is skipped imperatively" do
    skip
  end
end
```

[rspec]: https://rspec.info/
[rspec-skip]: https://relishapp.com/rspec/rspec-core/docs/pending-and-skipped-examples/skip-examples

## XCTest

The [XCTest][xctest] test framework provided by Apple as part of its normal SDK
provides [`XCTSkip`][xctskip] which may be thrown from a test function to
indicate that it should be skipped.

[xctest]: https://developer.apple.com/documentation/xctest
[xctskip]: https://developer.apple.com/documentation/xctest/xctskip

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should the return value of a `skipfn` be `Option<String>` or some other
structured type? The downside of a more structured type is the requirement that
the `test` crate exposes symbols which need to be used (currently all of these
are for `#[bench]` tests and behind a feature gate).

This does interact with [eRFC 2318][erfc-2318] ([tracking
issue][erfc-2318-tracking-issue]) in that custom test frameworks would need to
support skipping logic.

Is `predicate` the best name for the argument name? I'm not sure it is, but
couldn't come up with something better right now.

[erfc-2318]: https://github.com/rust-lang/rfcs/pull/2318
[erfc-2318-tracking-issue]: https://github.com/rust-lang/rust/issues/50297

# Future possibilities
[future-possibilities]: #future-possibilities

In addition to `skip_if(predicate = …)`, `skip_if(cfg = …)` might be plausible
to let users know of a given test that is not available on their platform.
