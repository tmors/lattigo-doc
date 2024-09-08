# lattigo-doc

## Overview

![lattigo-hierarchy](https://github.com/tuneinsight/lattigo/raw/main/lattigo-hierarchy.svg)

There are three implementations of RLWE-based homomorphic encryption schemes:

* bfv: A Full-RNS variant of the Brakerski-Fan-Vercauteren scale-invariant homomorphic encryption scheme. This scheme is instantiated via a wrapper of the `bgv` scheme. It provides modular arithmetic over the integers
* bgv: A Full-RNS generalization of the Brakerski-Fan-Vercauteren scale-invariant (BFV) and Brakerski-Gentry-Vaikuntanathan (BGV) homomorphic encryption schemes. It provides modular arithmetic over the integers
* ckks: A Full-RNS Homomorphic Encryption for Arithmetic for Approximate Numbers (HEAAN, a.k.a. CKKS) scheme. It provides fixed-point approximate arithmetic over the complex numbers (in its classic variant) and over the real numbers (in its conjugate-invariant variant)

## Supported Circuits

| Circuit                     | Support schema |
| :-------------------------- | -------------- |
| comparison: sign, max, step | ckks           |
| inverse                     | ckks           |
| mod1                        | ckks           |
| polynomial                  | ckks, bgv, bfv      |
| lintrans                    | ckks, bgv, bfv      |
| dft                         | ckks           |
| minimax                       | ckks           |
| bootstrapping               | ckks, bgv, bfv          |


## Supported Types | Ops
| Type            | Op                       | Supported |
| --------------- | ------------------------ | :-------: |   
| ring 2**k      | `+`      Addition           |    ✅     |
|                 | `-`  Subtraction   |    ✅     |
|                 | `*`   Multiplication     |    ✅     |
|                 | `-`     Neq    |    ✅     | 
|                 | `!=` Not Equal |    ✅     |
|                 | `<<` Shift Left                |    ✅     |
|                 | `>>` Shift Right                |    ✅     |
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

## Usage Guide
To learn how to use lattigo, you can follow the example [here](https://github.com/tuneinsight/lattigo/blob/main/examples/multiparty/int_psi/main.go).
S1.Create encryption parameters(e.g. bgv).
```go
// LogN = 13 & LogQP = 218
params, err := bgv.NewParametersFromLiteral(bgv.ParametersLiteral{
		LogN:             13,
		LogQ:             []int{54, 54, 54},
		LogP:             []int{55},
		PlaintextModulus: 65537,
	})
```

S2. Define all the cooperators, create secret shares via common rlwe functionalities, and initiate inputs.
```go
N := 3 // Default number of parties
P := make([]*party, N)
kgen := rlwe.NewKeyGenerator(params)

for i := range P {
	pi := &party{}
	pi.sk = kgen.GenSecretKeyNew()

	pi.input = make([]uint64, params.N())
	for j := range pi.input {
		pi.input[j] = uint64(i)
	}
	P[i] = pi
}
```
S3.Generate collective public encryption key, collective public evaluation key, including relinearization key, galois key etc.
```go
crs, err := sampling.NewKeyedPRNG([]byte{'l', 'a', 't', 't', 'i', 'g', 'o'})
pk := ckgphase(params, crs, P)
RelinearizationKey := rkgphase(params, crs, P)
galKeys := gkgphase(params, crs, P)
```
S4.Encrypt plaintexts under collective public key.
```go
encoder := bgv.NewEncoder(params)
encInputs := make([]*rlwe.Ciphertext, N)

for i := range encInputs {
	[i] = bgv.NewCiphertext(params, 1, params.MaxLevel())
}
encryptor := rlwe.NewEncryptor(params, pk)
pt := bgv.NewPlaintext(params, params.MaxLevel())

elapsedEncryptParty := runTimedParty(func() {
	for i, pi := range P {
		if err := encoder.Encode(pi.input, pt); err != nil{
			panic(err)
		}
		if err := encryptor.Encrypt(pt, encInputs[i]); err != nil {
			panic(err)
		}
	}
}, N)

```

S5.Collective key switching and local decryption.
```go
encOut := cksphase(params, P, result)
decryptor := rlwe.NewDecryptor(params, P[0].sk)
ptres := bgv.NewPlaintext(params, params.MaxLevel())
elapsedDecParty := runTimed(func() {
	decryptor.Decrypt(encOut, ptres)
})
```
