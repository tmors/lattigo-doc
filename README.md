# lattigo-doc

## overview

![lattigo-hierarchy](https://github.com/tuneinsight/lattigo/raw/main/lattigo-hierarchy.svg)

There are three implementations of RLWE-based homomorphic encryption schemes:

* bfv: A Full-RNS variant of the Brakerski-Fan-Vercauteren scale-invariant homomorphic encryption scheme. This scheme is instantiated via a wrapper of the `bgv` scheme. It provides modular arithmetic over the integers
* bgv: A Full-RNS generalization of the Brakerski-Fan-Vercauteren scale-invariant (BFV) and Brakerski-Gentry-Vaikuntanathan (BGV) homomorphic encryption schemes. It provides modular arithmetic over the integers
* ckks: A Full-RNS Homomorphic Encryption for Arithmetic for Approximate Numbers (HEAAN, a.k.a. CKKS) scheme. It provides fixed-point approximate arithmetic over the complex numbers (in its classic variant) and over the real numbers (in its conjugate-invariant variant)

## supported circuits

| Circuit                     | Support schema |
| :-------------------------- | -------------- |
| comparison: sign, max, step | ckks           |
| inverse                     | ckks           |
| mod1                        | ckks           |
| polynomial                  | ckks、bgv      |
| lintrans                    | ckks、bgv      |
| dft                         | ckks           |
| bootstrapping               | Ckks           |

## supported types(CKKS)
todo: fixcode
#### Addition

```go
# ciphertext + ciphertext
pt1 := ckks.NewPlaintext(params, params.MaxLevel())
enc := rlwe.NewEncryptor(params, pk)
ct1, err := enc.EncryptNew(pt1)
if err != nil {
  panic(err)
}
pt2 := ckks.NewPlaintext(params, params.MaxLevel())
ct2, err := enc.EncryptNew(pt2)
if err != nil {
  panic(err)
}
ct3, err := eval.AddNew(ct1, ct2)
```

#### Multiplication

```go
# ciphertext * ciphertext
res, err := eval.MulRelinNew(ct1, ct2)
```

#### ROTATION & CONJUGATION

```go
rot := 5
galEls := []uint64{
  // The galois element for the cyclic rotations by 5 positions to the left.
  params.GaloisElement(rot),
  // The galois element for the complex conjugatation.
  params.GaloisElementForComplexConjugation(),
}

// We then generate the `rlwe.GaloisKey`s element that corresponds to these galois elements.
// And we update the evaluator's `rlwe.EvaluationKeySet` with the new keys.
eval = eval.WithKey(rlwe.NewMemEvaluationKeySet(rlk, kgen.GenGaloisKeysNew(galEls, sk)...))
# ROTATION
ct3, err = eval.RotateNew(ct1, rot)
# CONJUGATION
ct3, err = eval.ConjugateNew(ct1)
```

#### POLYNOMIAL EVALUATION

```go
SiLU := func(x complex128) (y complex128) {
  return x / (cmplx.Exp(-x) + 1)
}
interval := bignum.Interval{
  Nodes: 63,
  A:     *bignum.NewFloat(-8, prec),
  B:     *bignum.NewFloat(8, prec),
}
poly := bignum.ChebyshevApproximation(SiLU, interval)
tmp := bignum.NewComplex().SetPrec(prec)
for i := 0; i < Slots; i++ {
  want[i] = poly.Evaluate(tmp.SetComplex128(values1[i])).Complex128()
}
scalarmul, scalaradd := poly.ChangeOfBasis()
res, err = eval.MulNew(ct1, scalarmul)
if err != nil {
  panic(err)
}
if err = eval.Add(res, scalaradd, res); err != nil {
  panic(err)
}
if err = eval.Rescale(res, res); err != nil {
  panic(err)
}
polyEval := polynomial.NewEvaluator(params, eval)

if res, err = polyEval.Evaluate(res, poly, params.DefaultScale()); err != nil {
  panic(err)
}
```

#### LINEAR TRANSFORMATIONS

```go
batch := 37
n := 127

eval = eval.WithKey(rlwe.NewMemEvaluationKeySet(rlk, kgen.GenGaloisKeysNew(params.GaloisElementsForInnerSum(batch, n), sk)...))
if err := eval.InnerSum(ct1, batch, n, res); err != nil {
  panic(err)
}
if err := eval.Replicate(ct1, batch, n, res); err != nil {
  panic(err)
}
```

