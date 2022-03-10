# sonescript

> Sound language compiling to WASM

_Sonescript_ is designed to be useful for writing sound formulas / audio processing code for various audio targets, such as [AudioWorkletProcessor](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorkletProcessor/process), [audio engines](https://github.com/audiojs/web-audio-api), individual audio nodes etc.

Initially inspired by [zzfx](https://github.com/KilledByAPixel/ZzFX), [bytebeat](https://github.com/greggman/html5bytebeat), [hxos](https://github.com/stagas/hxos), [web-audio-engine](https://ghub.io/web-audio-engine) and others, but soon it became clear that JS limitations are no-go for sound processing and it needs something more foundational with better low-level control, which WASM perfectly provides.

## Goals

* Familiar & intuitive syntax.
* Fluent & expressive.
* Radically minimal & performant.
* 0 runtime, types predictable in advance, static memory.
* Compiling to WASM.

<!--
## Use cases

- describing sounds with tiny footprint (copy-pasteable string);
- reproducing synthesizers;
- alternative to heavy soundfonts ([like this](https://magenta.github.io/magenta-js/music/index.html#soundfonts));
- variable sounds (by analogy with [variable fonts](https://en.wikipedia.org/wiki/Variable_font)) − in reality sounds _are_ variable;
- fun.
-->

<!-- * drop `source | filter() | reverb() | fx()` and get controls with defaults; changing controls changes values; -->

## Examples

### Gain

Gain processor, providing k-rate amplification of mono, stereo or generic input.

```fs
range = 0..1000

gain([left], volume in range) = [left * volume]
gain([left, right], volume in range) = [left * volume, right * volume]
gain([..channels], volume in range) = [..channels * volume]
```

Features:

* _function overload_ − function clause is matched automatically by call signature.
* _channeled_ input/output − `[left]` for mono, `[left, right]` for stereo, `[..channels]` for any number of input channels;
* _a-rate_/_k-rate param type_ − `[arg]` indicates a-rate (_accurate_) param, direct `arg` param is k-rate (_controlling_), per-block.
* _range_ − `from..to`, `from..<to`, `from>..to` define language-level range primitive, useful in arguments validation, destructuring, arrays constructor etc.
* _validation_ − `a in range` asserts and clamps argument to provided range, to avoid blowing up state.
* _destructuring_ − collects channels or group as `[a,..bc] = [a,b,c]`.

### Biquad Filter

Biquad filter processor for single channel input.

```fs
import sin, cos, pi from "math"

pi2 = pi*2
sampleRate = 44100

lp([x0], freq = 100 in 1..10000, Q = 1.0 in 0.001..3.0) = (
  ...x1, x2, y1, y2 = 0;    // internal state

  w = pi2 * freq / sampleRate;
  sin_w, cos_w = sin(w), cos(w);
  a = sin_w / (2.0 * Q);

  b0, b1, b2 = (1.0 - cos_w) / 2.0, 1.0 - cos_w, b0;
  a0, a1, a2 = 1.0 + a, -2.0 * cos_w, 1.0 - a;

  b0, b1, b2, a1, a2 *= 1.0 / a0;

  y0 = b0*x0 + b1*x1 + b2*x2 - a1*y1 - a2*y2

  x1, x2 = x0, x1
  y1, y2 = y0, y1

  y0
)
```

Features:

* _import_ − by default, all top-level functions and variables are exported. Unused functions are tree-shaken from compiled code. Built-in libs are: `math`, `std`. Additional libs: `latr`, `musi` and [others]().
* _scope_ − block scope is defined by nesting `()` (unlike `{}` in JS) − variables defined in block act within its scope.
* _grouping_ − comma operator allows bulk operations on many variables, such as `a,b,c = d,e,f` → `a=d, b=e, c=f` or `a,b,c + d,e,f` → `a+d, b+e, c+f` etc.
* _state_ − internal function state is persisted between fn calls via ellipsis operator `...state=init`. State is identified by function callsite for current module instance. That is like language-level react hooks.

### [ZZFX Coin](https://codepen.io/KilledByAPixel/full/BaowKzv)

> `zzfx(...[,,1675,,.06,.24,1,1.82,,,837,.06])`:

```fs
import pow, sign, round, abs, max, pi, inf, sin from "math"

pi2 = pi*2
sampleRate = 44100

// waveshape generators
oscillator = [
  phase -> [1 - 4 * abs( round(phase/pi2) - phase/pi2 )],
  phase -> [sin(phase)]
]

// adsr weighting
adsr(x, a, d, s, sv, r=1) = (
  ...i=0, t=i++/sampleRate

  a = max(a, 0.0001)                 // prevent click
  total = a + d + s + r

  t >= total ? 0 : x * (
    t < a ? t/a :                    // attack
    t < a + d ?                      // decay
    1-((t-a)/d)*(1-sv) :             // decay falloff
    t < a  + d + s ?                 // sustain
    sv :                             // sustain volume
    (total - t)/r * sv
  )
)
adsr(x, a, d, s, r) = adsr(x, a, d, s, 1, r)  // no sustain volume case
adsr(a, d, s, r) = x -> adsr(x, a, d, s, r)   // pipe case

// curve effect
curve(x, amt=1.82 in 0..10) = pow(sign(x) * abs(x), amt)
curve(amt) = x -> curve(x, amt)

// coin = triangle with pitch jump
coin(freq=1675, jump=freq/2, delay=0.06, shape=0) = (
  ...i=0, phase=0

  t = i++/sampleRate
  phase += (freq + t > delay ? jump : 0) * pi2 / sampleRate

  oscillator[shape](phase) | adsr(0, 0, .06, .24) | curve(1.82)
)
```

This features:

* _pipes_ − `|` operator for function in rhs calls that function with argument from lhs, eg. `a | b` === `b(a)`;
* _lambda functions_ − useful for organizing pipe transforms;
* _arrays_ − linear collection of same-type elements with fixed size. Useful for organizing enums, dicts, buffers etc. Arrays support alias name for items: `a = [first: 1, second: 2]` → `a[0] === a.first === 1`

## [Freeverb](https://github.com/opendsp/freeverb/blob/master/index.js)

```fs
import comb from "./combfilter.son"
import allpass from "./allpass.son"
import floor from "math"

sampleRate = 44100

a1,a2,a3,a4 = 1116,1188,1277,1356
b1,b2,b3,b4 = 1422,1491,1557,1617
p1,p2,p3,p4 = 225,556,441,341

stretch(n) = floor(n * sampleRate / 44100)
sum(a, b) = a + b

reverb((..input), room=0.5, damp=0.5) = (
  ...combs_a = a0,a1,a2,a3 | stretch
  ...combs_b = b0,b1,b2,b3 | stretch
  ...aps = p0,p1,p2,p3 | stretch

  ..combs_a | a -> comb(a, input, room, damp) >- sum + ..combs_b | a -> comb(a, input, room, damp) >- sum
  ^, ..aps >- (input, coef) -> p + allpass(p, coef, room, damp)
)
```

This features:

* _multiarg pipes_ − pipe transforms can be applied to multiple input arguments (similar to jQuery style);
* _fold operator_ − `a,b,c >- fn` acts as `reduce((a,b,c), fn)`, provides native way to efficiently apply reducer to a group or an array;
* _topic reference_ −  `^` refers to result of last expression, so that expressions can be joined in flow fashion without intermediary variables. (that's similar to [Hack pipeline](https://docs.hhvm.com/hack/expressions-and-operators/pipe) or [JS pipeline](https://github.com/tc39/proposal-pipeline-operator), without special operator)

## [Floatbeat](https://dollchan.net/bytebeat/index.html#v3b64fVNRS+QwEP4rQ0FMtnVNS9fz9E64F8E38blwZGvWDbaptCP2kP3vziTpumVPH0qZyXzfzHxf8p7U3aNJrhK0rYHfgHAOZZkrlVVu0+saKbd5dTXazolRwnvlKuwNvvYORjiB/LpyO6pt7XhYqTNYZ1DP64WGBYgczuhAQgpiTXEtIwP29pteBZXqwTrB30jwc7i/i0jX2cF8g2WIGKlhriTRcPjSvcVMBn5NxvgCOc3TmqZ7/IdmmEnAMkX2UPB3oMHdE9WcKqVK+i5Prz+PKa98uOl60RgE6zP0+wUr+qVpZNsDUjKhtyLkKvS+LID0FYVSrJql8KdSMptKKlx9eTIbcllvdf8HxabpaJrIXEiycV7WGPeEW9Y4v5CBS07WBbUitvRqVbg7UDtQRRG3dqtZv3C7bsBbFUVcALvwH86MfSDws62fD7CTb0eIghE/mDAPyw9O9+aoa9h63zxXl2SW/GKOFNRyxbyF3N+FA8bPyzFb5misC9+J/XCC14nVKfgRQ7RY5ivKeKmmjOJMaBJSbEZJoiZZMuj2pTEPGunZhqeatOEN3zadxrXRmOw+AA==)

Transpiled floatbeat/bytebeat song:

```fs
import pi, asin, sin from "math"

sampleRate = 44100

fract(x) = x % 1;
mix(a, b, c) = (a * (1 - c)) + (b * c);
tri(x) = 2 * asin(sin(x)) / pi;
noise(x) = sin((x + 10) * sin((x + 10) ** (fract(x) + 10)));
melodytest(time) = (
	melodyString = "00040008";
	melody = 0;
	i = 0;
  i++ < 5 :>
    melody += tri(
      time * mix(
        200 + (i * 900),
        500 + (i * 900),
        melodyString[floor(time * 2) % melodyString.length] / 16
      )
    ) * (1 - fract(time * 4));
	melody;
)
hihat(time) = noise(time) * (1 - fract(time * 4)) ** 10;
kick(time) = sin((1 - fract(time * 2)) ** 17 * 100);
snare(time) = noise(floor((time) * 108000)) * (1 - fract(time + 0.5)) ** 12;
melody(time) = melodytest(time) * fract(time * 2) ** 6 * 1;

song() = (
  ...t=0, time = t++ / sampleRate;
  [(kick(time) + snare(time)*.15 + hihat(time)*.05 + melody(time)) / 4]
)
```

It features:

* _loop operator_ − `:>` or `<:` acts as _while_/_until_ loop, calling rhs until lhs condition holds true.
* _string literal `""`_ − acts as array with ASCII codes.


## Language Reference


<p align=center>🕉</p>
