# KT-89074: multiplatform rework

This is a fork-only experiment, not an upstream review request.

## Contract and scope

`roundToInt` and `roundToLong` return the nearest integer, resolving exact
halfway cases toward positive infinity. They throw IllegalArgumentException
for NaN and saturate outside the destination range. `round` has a different
halfway rule (ties to even), so it is not a replacement.

JVM delegates rounding to Java; JS uses its own backend implementation.
Native and Wasm both evaluate `floor(x + 0.5)` in the input floating-point
type. The shared tests should run on all these backends. Only Native and
Wasm need this implementation change. Wasm-JS and Wasm-WASI share the
changed source but remain separate test targets. Native host/architecture
results must be reported separately.

## Why the old formula fails

* Double 0.49999999999999994 is below 0.5, but adding 0.5 rounds to 1.0.
  It therefore returns 1 instead of 0. Float has the analogous boundary.
* Float 8388609 is already integral. Its spacing is 1, so adding 0.5
  rounds to the even value 8388610. Double has the same problem at
  4503599627370497. Negative integral values can also change.

The common tests separate halfway cases from the integral-precision cases,
so one failure cannot hide the other category. Further cases cover signed
zero, minimum subnormals, and destination range boundaries. Expected
integers are explicit, not computed with the implementation under test.

## Proposed formula

After the existing exceptional-value/range checks, take `n = floor(x)`.
Return `n + 1` when `x - n >= 0.5`, and `n` otherwise, then convert to the
destination integer type.

* Integral x gives n = x and a zero difference, so no addition is done.
* For nonintegral x >= 1 or x <= -1, x and n have the same sign and their
  magnitudes differ by at most a factor of two. Their subtraction is exact
  (Sterbenz's lemma). The comparison selects the correct nearest integer.
* For 0 <= x < 1, n is zero and the comparison uses x directly.
* For -1 < x <= -0.5, subtraction from -1 is exact by the same lemma.
  For -0.5 < x < 0, its exact result is greater than 0.5; rounding cannot
  put it below 0.5. Thus it correctly selects zero. The subtraction is
  NOT always exact here: for a tiny negative subnormal it can round to 1.
* Incrementing n happens only for nonintegral inputs. Their neighboring
  integer results are representable in the input type.

The existing checks and final conversion together preserve saturation.
In particular, Long.MAX_VALUE converted to Double/Float becomes 2^63.
For input 2^63, `x > Long.MAX_VALUE` is false because the comparison converts
its integer operand. The final `toLong()` must saturate; the new boundary
tests cover this explicitly. The same issue occurs for Float/Int at 2^31.

## Limits of this argument

This explains the source algorithm, not every compiler lowering. Actual
runs are still needed on Native host architectures and the two Wasm
runtimes. Targeted MathTest runs are not the full Native stdlib suite.
No performance equivalence is claimed: the new path adds subtraction and
a comparison. If performance is a review concern it needs measurement,
not an assumption that numerical correctness implies equal speed.
