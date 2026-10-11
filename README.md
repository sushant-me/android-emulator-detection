# Detecting emulators with floating-point and audio-hardware probes

**A technical note on two cheap, local, network-free emulator-detection techniques found in a
production Android application, written up as techniques rather than as a target analysis.**

*Sushant Poudel*

---

## Why this note exists, and what it deliberately omits

While reversing an Android app I found two small native functions whose purpose was not what
their library names suggested. Both are emulator detectors, both run entirely on-device, and both
produce a value that is only meaningful in aggregate.

**I am writing up the techniques and not the target.** The operator's published policy explicitly
excludes "reversing the mobile application without demonstrating real consequences on the mobile
API level," which means there is no safe harbour for a target-specific writeup — and, more to the
point, the techniques are the interesting part. **No key material, no client identifiers, no
endpoints, and no hostnames appear in this note.** Where a detail is target-specific it is
described in the abstract.

---

## Technique 1 — fingerprint the FPU, not the device

The first function is a floating-point conformance probe. It is about 130 bytes, it takes a seed
and an iteration count, and it returns a double.

The loop computes, each iteration:

```
d = ( cos(d) * sin(d) + sqrt( |d| + i ) ) / 1.0000000000000011
```

**Two details make this work, and the second is the one people miss.**

### The fused multiply-add is the discriminating instruction

`sincos`, `sqrt` and `fdiv` are all **correctly rounded** under IEEE-754 — the standard requires it,
so any conformant implementation returns the same bits. **A conformance probe built only from those
would find nothing.**

The **fused** multiply-add (`fmadd` on AArch64, `vfmadd*` on x86) is different: **it computes
`a*b + c` with a single rounding at the end and does *not* round the intermediate product.**
Implementing that correctly requires carrying extra precision through the multiply. **An emulator
that approximates FMA as a separate multiply followed by an add rounds twice and diverges** — and
that divergence is exactly what the probe is measuring.

**This is why "just use `sin` and `cos` to fingerprint hardware" does not work, and why reaching for
the fused operation does.** It is also a nice example of a conformance test that is only a test
because of one instruction's defined deviation from the obvious implementation.

### The divisor is chosen to preserve the signal

**The constant `1.0000000000000011` is `1 + 2⁻⁵²` — one unit in the last place above 1.0** (the
double just past 1.0 is exactly `1 + 2⁻⁵²`; you can verify it with
`math.nextafter(1.0, 2)` in any language, or `0x3FF0000000000005` as raw bits).

Dividing by it moves the result by roughly one ULP. **So any divergence introduced by `fmadd` earlier
in the iteration is *carried forward* rather than rounded away, and the error accumulates across
iterations.**

**If the divisor were something like `1.0001` or `2.0`, the result would be rounded on every
iteration and the fingerprint would wash out.** **The constant is not a magic number — it is the
smallest perturbation that perturbs at all, chosen precisely because it does not erase what the
function measures.** That is the design insight worth taking away.

### How you would build this yourself

```c
double fpu_probe(double seed, int iterations) {
    for (int i = 0; i < iterations; i++) {
        double s, c;
        sincos(seed, &s, &c);
        seed = fma(c, s, sqrt(fabs(seed) + i)) / 0x1.0000000000001p0; /* 1 + 2^-52 */
    }
    return seed;
}
```

**Return the value; compare it against a baseline captured from known-good hardware.** One result
proves nothing. **A population of installs sharing one result is the signal** — that is what makes
it fraud instrumentation rather than a security control.

**A note on why this is not a good sole defence:** x86-64 has had FMA3 since 2013, ARM64 mandates it
in every implementation, so *real* hardware is consistent here and *emulated* hardware is where the
variation lives. **But it also means the probe will disagree with itself across two genuinely
different CPUs** — so treat the returned value as a fuzzy bucket, not an equality test.

---

## Technique 2 — read back the audio hardware profile

The second function is about 300 bytes. It opens a real `AAudio` output stream and then **reads back
four parameters that the hardware actually granted**, rather than the ones it asked for.

```
create builder -> set performance mode, sharing mode, sample rate
               -> open a REAL stream
               -> read back:  sharing mode
                              frames per burst
                              buffer capacity in frames
                              sample rate
               -> close
```

**The asymmetry is the design.** It *sets* three properties and *reads back* four. **The requested
values are a probe; the returned ones are the fingerprint.**

### Why frames-per-burst is the strongest signal

**Frames-per-burst is determined by the audio HAL and its driver, and is one of the values most
characteristic of a specific device class.** A real device reports a device-specific figure; **an
emulator with no audio device falls back to a generic default.** `buffer capacity` correlates with
it.

**`sharing mode` is the second-strongest, for a subtler reason:** real hardware frequently *refuses*
an exclusive sharing mode when another app holds the device. **So the value that comes back encodes
the hardware *and* the current device state** — which is hard to fake without a real audio stack.

### Why this is better than a build-property check

**Almost every emulator detector starts by reading `Build.FINGERPRINT`, `Build.MODEL`, or checking
for known emulator files.** Those are trivially spoofed with a Magisk module or an Xposed hook,
because they are *strings the emulator controls*.

**A hardware read-back is different: to fake it you have to emulate the audio HAL well enough that
the parameters it negotiates are plausible and *consistent with each other*.** That is a much higher
bar than editing `build.prop`, and it is why the technique is worth knowing.

### How you would build this yourself

```kotlin
val builder = AAudioStreamBuilder()
builder.setPerformanceMode(AAudioStream.PERFORMANCE_MODE_LOW_LATENCY)
builder.setSharingMode(AAudioStream.SHARING_MODE_EXCLUSIVE)
builder.setSampleRate(48000)
val stream = builder.openStream()

val profile = listOf(
    stream.getSharingMode(),                  // did exclusive get granted?
    stream.getFramesPerBurst(),               // hardware/driver specific
    stream.getBufferCapacityInFrames(),
    stream.getSampleRate(),
)
stream.close()
```

**Aggregate the tuple; do not threshold on any single field.** And handle the failure path: opening a
stream can fail legitimately (another app holds the device, audio focus rules), **so a failed open is
"unknown", not "emulator".** **Conflating the two is how these checks produce false positives against
real users.**

---

## What the pair tells you about the app that contained them

**Neither probe examines user behaviour. Both answer a question about the *device*, locally, with no
network round-trip.** They were two of three layers in the same app — the third was a
package-integrity check that also gated whether the native code would hand out usable key material
at all.

**The design assumption is explicit: the device is hostile, so fingerprint rather than trust, and
fail silently rather than loudly.** That is the right posture for an app whose business model makes
it a target for emulator farms, and it is a reasonable thing to build.

**It also means the useful framing is aggregate, not per-device.** One FPU result and one audio
profile say very little. A thousand installs agreeing on both is a farm. **Any detection built on
these should be scored centrally, not locally** — and any *evaluation* of them should ask what the
false-positive rate looks like on low-end and unusual hardware, which is where these probes are most
likely to misfire.

---

## Scope and honesty

**Everything above was established by static analysis — reading disassembly, symbol tables and
import lists — of an application I was given.**

**What I verified:** the instruction sequences, the constant `0x3FF0000000000005`, the `fmadd` and
`fdiv` usage, the `AAudio` import list and which of its calls are getters, and the function sizes.

**What I have *not* done, and do not claim:** I have not run either probe. **The claim that they
distinguish real hardware from emulated is an inference from *what the instructions compute*, and it
is well-founded because FMA rounding divergence is documented behaviour — but I have not measured a
real-vs-emulated difference on any device, so no measured numbers appear in this note.** Where I say
"an emulator diverges", read it as "the standard permits divergence here and this is the documented
mechanism", not as a benchmark I ran.

**An earlier draft of my own working notes called one of these functions a decoy and misread a branch
as two sequential passes.** Both were wrong and both were caught by disassembling properly. **I
mention it because a technique writeup that hides its author's wrong turns is less useful than one
that does not.**

---

## References

- IEEE 754-2019, for the fused multiply-add definition and correct-rounding requirements
- `math.nextafter` / `0x3FF0000000000005`, to confirm the divisor is `1 + 2⁻⁵²`
- Android NDK `AAudio` reference, for the read-back semantics of `getFramesPerBurst`,
  `getSharingMode`, `getBufferCapacityInFrames` and `getSampleRate`
