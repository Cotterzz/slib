# sl.cn - Complex Numbers

## A collection of utilities and functions for managing complex numbers in shaders.

In most cases the complex number is represented by a vec2 where x = real and y = imaginary.

---

### Conjugate: `sl.cn.conj(z: vec2) → vec2` 

```glsl
vec2 conj(vec2 z){  return vec2(z.x, -z.y);  }
```
---
### Multiply: `sl.cn.cmul(a: vec2, b: vec2) → vec2` 

```glsl
vec2 cmul(vec2 a, vec2 b){ return mat2(a, -a.y, a.x)*b; }
```
Performs complex multiplication using a 2×2 matrix trick.

This is equivalent to ( a.x\*b.x - a.y\*b.y ,  a.x\*b.y + a.y\*b.x )

Could be written more explicitly like this:

```glsl
vec2 cmul(vec2 a, vec2 b) { return vec2(a.x*b.x - a.y*b.y, a.x*b.y + a.y*b.x); }
```
There's a chance this might perform better as no matrix needs to be constructed,

But with today's GPUs and compilers this is unlikely.

---
### Inverse: `sl.cn.cinv(z: vec2) → vec2`
```glsl
vec2 cinv(vec2 z){ return vec2(z.x, -z.y)/dot(z, z); }
```
Returns 1/a = a*/|a|²

..or with conj():
```glsl
vec2 cinv(vec2 a){ return conj(a) / dot(a, a); }
```
and/or with division by zero protection:
```glsl
vec2 cinv(vec2 a){ return a==vec2(0)?0.0:vec2(z.x, -z.y)/dot(z, z); }
```
----
### Division: `sl.cn.cdiv(a: vec2, b: vec2) → vec2`
```glsl
vec2 cdiv(vec2 a, vec2 b){ return cmul(a, cinv(b)); }
```

----
### Logarithm: `sl.cn.clog(z: vec2) → vec2`
```glsl
vec2 clog(in vec2 z){ return vec2(log(length(z)), atan(z.y, z.x)); }
```
Returns ln(z) = ln|z| + i·arg(z)

----
### Exponential: `sl.cn.cexp(z: vec2) → vec2`
```glsl
vec2 cexp(vec2 z){ return exp(z.x)*vec2(cos(z.y), sin(z.y)); }
```
Computes e^z using Euler's formula: e^x · (cos(y) + i·sin(y))

----
### Square Root: `sl.cn.csqrt(z: vec2) → vec2`
```glsl
vec2 csqrt(vec2 z){ return cexp(clog(z)/2.); }
```
Uses the identity √z via z^(1/2) = e^(ln(z)/2)

This works but is relatively expensive (calls both clog and cexp).

A direct polar formula exists:
```glsl
vec2 csqrt(vec2 z) {
    float r = length(z);
    float a = atan(z.y, z.x);
    return sqrt(r) * vec2(cos(a/2.0), sin(a/2.0));
}
```
This avoids the exp and log calls, but it still calls atan, cos, and sin, which are expensive.

This algebraic solution is better:
```glsl
vec2 csqrt(vec2 z) {
    float r = length(z);
    float x = sqrt((r + z.x) * 0.5);
    float y = sqrt((r - z.x) * 0.5);
    return vec2(x, z.y >= 0.0 ? y : -y);
}
```
But it still has three sqrts.
There's also the risk of catastrophic cancellation — when z.x ≈ r (small imaginary part), r - z.x loses nearly all significant bits, and vice-versa for large negative real parts.
The standard trick (used by production csqrt implementations) is to compute only the well-conditioned component via sqrt, then recover the other from the identity 2xy = z.y:

```glsl
vec2 csqrt(vec2 z) {
    float r = length(z);
    // r + abs(z.x) is always well-conditioned (no cancellation)
    float w = sqrt(0.5 * (r + abs(z.x)));
    if (w == 0.0) return vec2(0.0);            // z == 0
    float halfInvW = 0.5 / w;
    if (z.x >= 0.0) {
        // Re is the stable component
        return vec2(w, z.y * halfInvW);
    } else {
        // Im is the stable component
        return vec2(abs(z.y) * halfInvW, z.y >= 0.0 ? w : -w);
    }
}
```
----
### Power: `sl.cn.cpow(a: vec2, b: vec2) → vec2`
```glsl
vec2 cpow(vec2 a, vec2 b){ return cexp(cmul(b, clog(a))); }
```
Computes a^b = e^(b · ln(a)). Handles complex exponents.

----
### Integer Power: `sl.cn.cpowi(z: vec2, n: int) → vec2`
(faster for small integer exponents)
```glsl
vec2 cpowi(vec2 z, int n) {
    vec2 result = vec2(1.0, 0.0);
    for(int i = 0; i < abs(n); i++) result = cmul(result, z);
    return n < 0 ? cinv(result) : result;
}
```
----
## Trivial

These functions simply pass the complex number to another function.

They are listed here for reference and not included in the sl library

```glsl
// Modulus (magnitude)
float cabs(vec2 z) { return length(z); }

// Argument (angle)
float carg(vec2 z) { return atan(z.y, z.x); }

// Squared modulus (avoids sqrt)
float cabs2(vec2 z) { return dot(z, z); }

vec2 cfract(vec2 z) { return fract(z); }
vec2 cfloor(vec2 z) { return floor(z); }

float creal(vec2 z) { return z.x; }
float cimag(vec2 z) { return z.y; }

// More..
// cadd, csub	Just use + - on vec2
// cscale(z, f)	Just use z * float
// cmix	mix() already works on vec2

```
----
## Trigonomoetric

### Sine
```glsl
vec2 csin(vec2 z) { return vec2(sin(z.x)*cosh(z.y), cos(z.x)*sinh(z.y)); }
```
----
### Cosine
```glsl
vec2 ccos(vec2 z) { return vec2(cos(z.x)*cosh(z.y), -sin(z.x)*sinh(z.y)); }
```
----
### Tangent
```glsl
vec2 ctan(vec2 z) { return cdiv(csin(z), ccos(z)); }
```
----
## Hyperbolic

### SineH
```glsl
vec2 csinh(vec2 z) { return vec2(sinh(z.x)*cos(z.y), cosh(z.x)*sin(z.y)); }
```
----
### CosineH
```glsl
vec2 ccosh(vec2 z) { return vec2(cosh(z.x)*cos(z.y), sinh(z.x)*sin(z.y)); }
```
----
### TangentH
```glsl
vec2 ctanh(vec2 z) { return cdiv(csinh(z), ccosh(z)); }
```
----
## Inverse

### Arc Sine
```glsl
// asin(z) = -i * ln(iz + sqrt(1 - z²))
vec2 casin(vec2 z) {
    vec2 iz = cmul(vec2(0, 1), z);
    vec2 sq = csqrt(vec2(1, 0) - cmul(z, z));
    return cmul(vec2(0, -1), clog(iz + sq));
}
```
----
### Arc Cosine
```glsl
// acos(z) = -i * ln(z + sqrt(z² - 1))
vec2 cacos(vec2 z) {
    vec2 sq = csqrt(cmul(z, z) - vec2(1, 0));
    return cmul(vec2(0, -1), clog(z + sq));
}
```
----
### Arc Tangent
```glsl
// atan(z) = (i/2) * ln((1-iz)/(1+iz))
vec2 catan(vec2 z) {
    vec2 iz = cmul(vec2(0, 1), z);
    return cmul(vec2(0, 0.5), clog(cdiv(vec2(1, 0) - iz, vec2(1, 0) + iz)));
}
```
----
## Inverse Hyberbolic

### Arc SineH
```glsl
// asinh(z) = ln(z + sqrt(z² + 1))
vec2 casinh(vec2 z) {
    vec2 sq = csqrt(cmul(z, z) + vec2(1, 0));
    return clog(z + sq);
}
```
----
### Arc CosineH
```glsl
// acosh(z) = ln(z + sqrt(z² - 1))
vec2 cacosh(vec2 z) {
    vec2 sq = csqrt(cmul(z, z) - vec2(1, 0));
    return clog(z + sq);
}
```
----
### Arc TangentH
```glsl
// atanh(z) = (1/2) * ln((1+z)/(1-z))
vec2 catanh(vec2 z) {
    return cmul(vec2(0.5, 0), clog(cdiv(vec2(1, 0) + z, vec2(1, 0) - z)));
}
```
----
## Misc

### From Polar
```glsl
// Construct from polar form: r * e^(iθ)
vec2 cpolar(float r, float theta) {
    return vec2(r * cos(theta), r * sin(theta));
}
```
----
### To Polar
```glsl
// Convert to polar (r, theta)
vec2 ctopolar(vec2 z) {
    return vec2(length(z), atan(z.y, z.x));
}
```
----
### Normalize
```glsl
// z / |z| - projects onto unit circle
vec2 cnormalize(vec2 z) {
    return z / length(z);
}
```
----
### Optimised square
```glsl
vec2 csquare(vec2 z) {
    return vec2(z.x*z.x - z.y*z.y, 2.0*z.x*z.y);
}
```
