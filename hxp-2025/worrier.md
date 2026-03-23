# worrier

_Cryptography, Zajebiste_

Credit: yyyyyyy

## Description
The good thing about making mistakes is that you can learn from your errors.

## Download
[worrier-1ed2fcc7e602200f.tar.xz](https://github.com/jasperjjhe/Capture-the-Flag/blob/main/hxp-2025/worrier-1ed2fcc7e602200f.tar.xz)

## Approach

This one took me a while. I went down several wrong paths finally solving.

The general idea here seems to be:
- A prime $p = 2^216 \cdot 3^137 - 1$
- A field $F = GF(p^2)$ built with the polynomial $x^2 + 1$ (so elements are $a + b\cdot i$)
- A base curve $E0: y^2 = x^3 + x$ over $F$
- $E0$ is set to have order $(p+1)^2$
- Two torsion bases are implicitly in play: the 2-power torsion ($2^216$) and the 3-power torsion ($3^137$)
- A secret isogeny $\Phi: E0 \rightarrow E1$ (unknown curve, unknown degree)
- We're given 256 pairs $(Xs[i], Ys[i])$ where $Ys[i] = \Phi(Xs[i])$
- We're given a challenge point $X$ on $E0$ and $Y = \Phi(X)$ on $E1$
- We need to recover some secret $\mu$ to decrypt the flag

This structure seems to be consistent with supersingular isogeny Diffie–Hellman key exchange (SIDH/SIKE). The isogeny $\Phi$ has degre $2^216$ and the torsion structure lets us do a pairing-based attack.

$\mu$ is a pair of integers, the secret's representation in the 2-torsion basis. Specifically $Y - \Phi(X)$ expressed in terms of $P1, Q1$ after scaling by $B3 = 3^137$.

After multiplying everything by $3^137$, we're working purely in the $2^216$-torsion and $\Phi$ is a linear map over $\frac{Z}{2^216}$ when restricted to $E0[2^216]$.

The attack is a Castryck-Decru / SIDH attack framework, where knowing hte acction of the secret isogeny on enough torsion points lets you reconstruct the isogeny entirely.

A standard approach:

1. Pick a basis $(A0, B0)$ of $E0[2^216]$
2. For each sample pair $(Xs[i], Ys[i])$, compute $U = 3^137 \cdot Xs[i]$ (projecting onto the 2-torsion) and $V = 3^137 \cdot Ys[i]$ (which equals $\Phi(U)$)
3. Express $U$ in the basis: $U = u1\cdot A0 + u2\cdot B0$
4. Express $V$ in the basis $(P1, Q1)$ on $E1$: $V = v1\cdot P1 + v2\cdot Q1$
5. The relationship $[v1, v2]^T = M \cdot [u1, u2]^T$ holds over $Z/2^216$ for some $2\cross2$ matrix $M$
6. Find $M$ from two such pairs and verify against the rest
7. Apply $M$ to get $\Phi(3^137 \cdot X)$ and subtract from $3^137 \cdot Y$ to get $\mu$

The coordinate extraction uses Weil pairings:

```python
g  = R1.weil_pairing(R2, n)   # generator in μ_{2^216}
ha = R2.weil_pairing(P,  n)   # = g^{-alpha}
hb = R1.weil_pairing(P,  n)   # = g^{beta}
alpha = (-ha.log(g)) % n
beta  = ( hb.log(g)) % n
```

If $P = \alpha \cdot R1