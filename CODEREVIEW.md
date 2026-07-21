# Code Review: codraw/fixer

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **L1** — `ClassNotation/ClassStaticCallFixer.php`: replaced the cursor-preserving
  `continue` with `break` in the `while` loop of `applyFix()`, removing the latent
  infinite loop when `getNextMeaningfulToken()` returns `null`.
- **L3** — `composer.json`: added `"exclude-from-classmap": ["/Tests/"]` to `autoload`
  so consumers running `composer dump-autoload -o` no longer classmap the
  un-namespaced, duplicated test fixtures (no more "does not comply with psr-4" /
  ambiguous-class warnings).
- **L4 (partial)** — `composer.json`: fixed description typo "Customer rules" →
  "Custom rules". `ClassNotation/ClassStaticCallFixer.php`: added the missing risky
  description (4th `FixerDefinition` constructor argument) used by php-cs-fixer's
  `describe` output. The remaining L4 items are left open: the `"php": ">=8.5"`
  constraint style is a deliberate repo convention, and having `RuleSet::adjust()` call
  `$config->setRiskyAllowed(true)` is a behavior change that needs a design decision.
- Not fixed (deliberately): H1, H2, M1, M2, M3 change the transforms the fixers apply
  and would alter formatting output in consuming applications; M4 (marking
  `ClassPrivateStaticCallFixer` risky) would make it stop running for consumers not
  using `--allow-risky`; L2 is a performance refactor; L5 needs a scope decision.
  All remain open findings.

### Validation pass (2026-07-20)

- `composer install --optimize-autoloader --no-interaction --prefer-dist --no-scripts`:
  clean; the `^8.5` constraint resolves without changes.
- PHPUnit 12.5.31: 7 tests, 7 assertions, all passing with the applied fixes — no test
  fallout, no code changes needed.
- PHPStan (level 5): 4 errors reported, all in `Tests/fixtures/**` fixture files
  (`staticClassAccess.private*` — fixtures deliberately contain `static::` access to
  private members, which is exactly what the fixers transform). Verified via `git stash`
  that all 4 occur without the applied fixes too: pre-existing, left as-is.
- markdownlint-cli2: auto-fix normalized blank-line spacing around headings/lists in
  this file only; all Markdown files now lint clean.
- No additional CODEREVIEW findings were fixed in this pass: the open items (H1, H2,
  M1–M3, M4, L2, L5, remaining L4) have no existing test coverage or require design
  decisions, so they were left untouched per validation scope.

## Overall Assessment

`codraw/fixer` is a very small package providing two custom php-cs-fixer rules
(`Draw/class_static_call`, `Draw/class_private_static_call`) plus a `RuleSet` helper that
registers and enables them. The code is short, readable, and gated correctly through
`isCandidate()`, and the fixture-based test harness is easy to extend. However, the
token-scanning heuristics are too naive for real-world PHP: the "enclosing class" and
"member visibility" lookups both search the raw token stream without scoping, which
produces wrong transforms in common code (`::class` literals, `final readonly` classes,
multiple classes per file, members without explicit visibility). Most significantly, the
two fixers prescribe *opposite* transforms for the same code — the package's own test
fixtures for the two fixers are exact mirror images — so when `RuleSet::adjust()` enables
both, the end result silently depends on fixer registration order. Coverage exists only
for the trivial happy paths.

## Findings

### High

#### H1. `isClassFinal()` resolves the wrong class: `::class` literals and anonymous classes defeat the backward search

`ClassNotation/ClassStaticCallFixer.php:98-113`

`isClassFinal()` walks backward from the `::` token to the *nearest preceding*
`T_CLASS` token and checks whether the token before it is `final`. But `T_CLASS` is also
produced by the `class` keyword in `Foo::class` literals and by anonymous classes
(`new class { ... }`). Example, entirely normal code:

```php
final class A
{
    public function m(): void
    {
        if ($x === Foo::class) {
            self::helper();   // nearest preceding T_CLASS is the `::class` token
        }
    }
}
```

The backward search finds the `class` of `Foo::class`, whose previous meaningful token is
`::`, not `final`, so the class is treated as non-final and `self::helper()` is rewritten
to `static::helper()` — the exact opposite of the fixer's own rule for final classes
(see fixture `Tests/fixtures/ClassStaticCallFixerTest/in|out/StaticPrivateFinalClass.php`,
which asserts final classes must use `self::`). The same misresolution happens after any
anonymous class in the method body. Relatedly, the outer loop at
`ClassNotation/ClassStaticCallFixer.php:59-71` also treats the `::class` token as a class
declaration: `getNextTokenOfKind($index, ['{', ';'])` will latch onto the `{` of a
following `if`/`match` block and re-scan it as a "class body". A guard that skips
`T_CLASS` tokens preceded by `T_DOUBLE_COLON` / `T_NEW` (as php-cs-fixer core fixers do)
is required in both places.

#### H2. The two fixers contradict each other on private static method calls; the winner is decided by registration order

`ClassNotation/ClassStaticCallFixer.php:88-89`, `ClassNotation/ClassPrivateStaticCallFixer.php:70-72`, `RuleSet.php:13-16,38-39`

`ClassStaticCallFixer` converts `self::foo()` → `static::foo()` for *any* method call in a
non-final class — including private methods (`isMethodCall()` never checks visibility).
`ClassPrivateStaticCallFixer` converts `static::foo()` → `self::foo()` precisely when
`foo` is private. The package's own fixtures prove the contradiction: for the identical
input class,
`Tests/fixtures/ClassStaticCallFixerTest/*/StaticPrivateMethod.php` asserts
`self::foo()` → `static::foo()`, while
`Tests/fixtures/ClassPrivateStaticCallFixerTest/*/StaticPrivateMethod.php` asserts
`static::foo()` → `self::foo()` — mirror images of each other. `RuleSet::adjust()`
enables both rules. Both fixers use `AbstractFixer`'s default priority (0), so
php-cs-fixer's stable sort applies them in registration order (`RuleSet.php:13-16`
registers `ClassStaticCallFixer` first); the private fixer then reverts the static
fixer's work, and the net outcome flips entirely if a consumer registers the fixers in
the other order. `ClassStaticCallFixer` should skip members that resolve as private (or
the fixers should declare explicit, coordinated priorities), and there should be a test
running both fixers together.

### Medium

#### M1. `final readonly class` is treated as non-final

`ClassNotation/ClassStaticCallFixer.php:106-112`

`isClassFinal()` only inspects the single meaningful token immediately before `class`.
For `final readonly class Foo` (the standard modifier order enforced by php-cs-fixer's
own `class_definition` rules on PHP 8.2+), that token is `readonly`, so the class is
classified as non-final and gets `static::` calls instead of `self::`. The check needs to
walk all class modifiers.

#### M2. `hasPrivateAccessor()` scans the whole file, not the enclosing class, and first match wins

`ClassNotation/ClassPrivateStaticCallFixer.php:126-162`

The visibility lookup iterates the entire token stream from index 0 with no class
scoping. Consequences: (a) in a file with two classes, a private `foo()` in class A makes
`static::foo()` inside class B convert to `self::foo()` even if B's `foo()` is public or
inherited; (b) for `T_VARIABLE` lookups, *any* variable with the same name matches —
a method parameter or local `$foo` occurring earlier in the file than the
`private static $foo` declaration is examined instead of the property, yielding the wrong
visibility. The scan must be bounded to the class body that contains the `static::`
reference and restricted to member declarations.

#### M3. Unbounded backward visibility search misclassifies members without an explicit modifier

`ClassNotation/ClassPrivateStaticCallFixer.php:147-157`

`getPrevTokenOfKind($index, [private, protected, public])` has no lower bound, so it can
land on the visibility keyword of a *previous* member. Example:

```php
class A
{
    private string $x;

    static function foo(): void {}  // implicitly public

    public function run(): void
    {
        static::foo();  // converted to self::foo() — foo() is judged "private"
    }
}
```

The nearest preceding visibility keyword before `function foo` is the `private` of `$x`,
so the implicitly-public `foo()` is treated as private and the call is rewritten to
`self::foo()`. If a subclass overrides `foo()`, this changes runtime behavior. The search
must stop at the start of the member declaration (e.g. at the previous `;`/`}`/`{`).

#### M4. `ClassPrivateStaticCallFixer` is not marked risky although it can change behavior

`ClassNotation/ClassPrivateStaticCallFixer.php:11` (no `isRisky()` override)

Rewriting `static::member()` to `self::member()` alters late-static-binding semantics
whenever a subclass (re)declares the member with non-private visibility: before the fix,
`static::foo()` dispatches to the subclass override; after, `self::foo()` always calls
the parent's private method. `ClassStaticCallFixer` correctly declares `isRisky(): true`
(`ClassNotation/ClassStaticCallFixer.php:18-21`), but this fixer silently runs without
`--allow-risky`, and the misclassification bugs in M2/M3 widen its blast radius.

### Low

#### **[FIXED]** L1. Latent infinite loop: `continue` without advancing the cursor

`ClassNotation/ClassStaticCallFixer.php:80-83`

Inside the `while` loop, `if (null === $nextIndex = ...) { continue; }` re-enters the
loop with an unchanged `$thisIndex`, which would spin forever. It is unreachable today
only because a token before `$classCloseIndex` always has a meaningful successor (the
closing brace), but the statement should be `break` to be safe.

#### L2. O(n²) scanning in `hasPrivateAccessor()`

`ClassNotation/ClassPrivateStaticCallFixer.php:129`

Every `static::` reference triggers a fresh full-file token scan from index 0. On large
files with many static references this multiplies quickly; visibility of members could be
collected once per class in a single pass.

#### **[FIXED]** L3. Tests and non-PSR-4 fixtures are exposed through the production autoloader

`composer.json:25-29`

The PSR-4 root mapping `"Draw\\Fixer\\": ""` includes `Tests/` and
`Tests/fixtures/**`. The fixture files are un-namespaced (`class StaticPrivateMethod`
in the global namespace) and duplicated between `in/` and `out/` directories, so any
consumer running `composer dump-autoload -o` gets "does not comply with psr-4" /
ambiguous-class warnings from this package. Move tests to `autoload-dev` and/or add the
fixtures to `exclude-from-classmap`.

#### **[FIXED]** (partially) L4. Packaging / API polish

`composer.json:2-3,17`, `RuleSet.php:32-44`, `ClassNotation/ClassStaticCallFixer.php:23-45`

- `"php": ">=8.5"` is unbounded upward (allows PHP 9) and pins to a bleeding-edge minimum;
  `^8.5` (or the intended supported range) would be safer.
- Description typo: "Customer rules" → "Custom rules".
- `RuleSet::adjust()` enables the risky `Draw/class_static_call` rule but does not call
  `$config->setRiskyAllowed(true)`, so a config that uses only `RuleSet::adjust()` fails
  at runtime until the caller remembers to allow risky rules — worth documenting or
  handling.
- The risky fixer's `FixerDefinition` omits a risky description (third constructor
  argument), which php-cs-fixer uses in `describe` output.

#### L5. Enums, traits, and interfaces are ignored

`ClassNotation/ClassStaticCallFixer.php:47-50`, `ClassNotation/ClassPrivateStaticCallFixer.php:42-45`

`isCandidate()` and the scan loop only handle `T_CLASS`. `static::`/`self::` usage inside
enums and traits (where these conventions apply just as much, and where `static::` in
traits is especially common) is never processed. If intentional, document the limitation.

## Strengths

- Clear, single-purpose package with minimal dependencies and small surface area.
- `ClassStaticCallFixer` correctly declares itself risky, and both fixers gate work with
  a cheap `isCandidate()` check before scanning.
- `RuleSet::addCustomFixers()` deduplicates against already-registered fixers, so calling
  it on a pre-configured `ConfigInterface` is safe and idempotent.
- The fixture-based test harness (`in/` → `out/` file pairs discovered by glob) makes
  adding regression cases trivial.
- Static analysis is configured at PHPStan level 5 with an *empty* baseline
  (`phpstan-baseline.neon`), and CI workflows are present.
- Token replacement is done in place without insertions, so cached token counts and block
  indices stay valid during the pass.

## Test Coverage

Coverage is happy-path only. `ClassPrivateStaticCallFixerTest` has three fixtures
(private const, private static property, private static method — all single-class,
explicit-visibility, convert-expected cases) and `ClassStaticCallFixerTest` has four
(the same three plus one final class). Notably missing:

- **Negative cases**: no fixture asserts that `static::` calls to *public/protected*
  members are left alone by `ClassPrivateStaticCallFixer`, or that non-call `self::`
  usages are untouched.
- **Both fixers together**: no test runs the two fixers in sequence, which is exactly the
  configuration `RuleSet::adjust()` produces and would have exposed finding H2.
- **`RuleSet` itself**: completely untested (registration dedup, rule enabling).
- **Real-world token patterns**: `::class` literals, anonymous classes,
  `final readonly class`, multiple classes per file, members without explicit visibility,
  traits/enums — every bug found above (H1, M1–M3) lacks a fixture.
