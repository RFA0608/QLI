---
title: 04. Go based controller description
---
# Controller
## Intmat
This code must use `F_q`, `G_q`, `H_q` and `R_q`, which are output when running the Python-implemented `ctrl_intmat_q.py`. You can verify this output in the CMD, as referenced in [[02_Python based controller description#Intmat#Quantized|Python based controller description>Intmat>Quantized]]. The code is located at [QQS3C/interface/controller/go](https://github.com/RFA0608/QQS3C/tree/main/interface/controller/go).
### Model_enc
This section explains the various functions in "model_enc.go". First, it introduces the custom structs.
```Go
	type crypto struct {
		params    *rlwe.Parameters
		levelQ    int
		levelP    int
		ringQ     *ring.Ring
		monomials []ring.Poly
		tau       int
	
		kgen *rlwe.KeyGenerator
		sk   *rlwe.SecretKey
		rlk  *rlwe.RelinearizationKey
	
		encryptorRLWE *rlwe.Encryptor
		decryptorRLWE *rlwe.Decryptor
		evaluatorRLWE *rlwe.Evaluator
		encryptorRGSW *rgsw.Encryptor
		evaluatorRGSW *rgsw.Evaluator
	}

	type info_enc struct {
		r float64
		s float64
		L float64
	
		params        *rlwe.Parameters
		ringQ         *ring.Ring
		monomials     []ring.Poly
		tau           int
		evaluatorRLWE *rlwe.Evaluator
		evaluatorRGSW *rgsw.Evaluator
	
		ctF []*rgsw.Ciphertext
		ctG []*rgsw.Ciphertext
		ctH []*rgsw.Ciphertext
		ctR []*rgsw.Ciphertext
	}
```
In this code, the `crypto` struct is a variable used to group cryptography-related components. The `info_enc` struct is related to encryption but does not include encryption keys, so it is a struct that can be freely used in the controller.

Next is a function that returns parameters.
```Go
	func Get_params() *rlwe.Parameters {
		params, _ := rlwe.NewParametersFromLiteral(rlwe.ParametersLiteral{
			LogN:    12,
			LogQ:    []int{60},
			LogP:    []int{60},
			NTTFlag: true,
		})
	
		return &params
	}
```
This function returns parameter values suitable for the cryptographic structure based on the hard-coded parameters. It is necessary when creating the crypto object.

Next is the `Crypto` function, which corresponds to the crypto object.
```Go
	func Crypto(params *rlwe.Parameters) *crypto {
		// set parameters
		n := 4
		m := 1
		p := 2
		levelQ := params.QCount() - 1
		levelP := params.PCount() - 1
		ringQ := params.RingQ()
	
		// Compute tau
		// least power of two greater than n, p_, and m
		maxDim := math.Max(math.Max(float64(n), float64(m)), float64(p))
		tau := int(math.Pow(2, math.Ceil(math.Log2(maxDim))))
	
		// Generate DFS index for unpack
		dfsId := make([]int, tau)
		for i := 0; i < tau; i++ {
			dfsId[i] = i
		}
	
		tmp := make([]int, tau)
		for i := 1; i < tau; i *= 2 {
			id := 0
			currBlock := tau / i
			nextBlock := currBlock / 2
			for j := 0; j < i; j++ {
				for k := 0; k < nextBlock; k++ {
					tmp[id] = dfsId[j*currBlock+2*k]
					tmp[nextBlock+id] = dfsId[j*currBlock+2*k+1]
					id++
				}
				id += nextBlock
			}
	
			for j := 0; j < tau; j++ {
				dfsId[j] = tmp[j]
			}
		}
	
		// Generate monomials for unpack
		logn := int(math.Log2(float64(tau)))
		monomials := make([]ring.Poly, logn)
		for i := 0; i < logn; i++ {
			monomials[i] = ringQ.NewPoly()
			idx := params.N() - params.N()/(1<<(i+1))
			monomials[i].Coeffs[0][idx] = 1
			ringQ.MForm(monomials[i], monomials[i])
			ringQ.NTT(monomials[i], monomials[i])
		}
	
		// Generate Galois elements for unpack
		galEls := make([]uint64, int(math.Log2(float64(tau))))
		for i := 0; i < int(math.Log2(float64(tau))); i++ {
			galEls[i] = uint64(tau/int(math.Pow(2, float64(i))) + 1)
		}
	
		// Generate keys
		kgen := rlwe.NewKeyGenerator(params)
		sk := kgen.GenSecretKeyNew()
		rlk := kgen.GenRelinearizationKeyNew(sk)
		evkRGSW := rlwe.NewMemEvaluationKeySet(rlk)
		evkRLWE := rlwe.NewMemEvaluationKeySet(rlk, kgen.GenGaloisKeysNew(galEls, sk)...)
	
		// Define encryptor and evaluator
		encryptorRLWE := rlwe.NewEncryptor(params, sk)
		decryptorRLWE := rlwe.NewDecryptor(params, sk)
		evaluatorRLWE := rlwe.NewEvaluator(params, evkRLWE)
		encryptorRGSW := rgsw.NewEncryptor(params, sk)
		evaluatorRGSW := rgsw.NewEvaluator(params, evkRGSW)
	
		crypto_cl := crypto{
			params:        params,
			levelQ:        levelQ,
			levelP:        levelP,
			ringQ:         ringQ,
			monomials:     monomials,
			tau:           tau,
			kgen:          kgen,
			sk:            sk,
			rlk:           rlk,
			encryptorRLWE: encryptorRLWE,
			decryptorRLWE: decryptorRLWE,
			evaluatorRLWE: evaluatorRLWE,
			encryptorRGSW: encryptorRGSW,
			evaluatorRGSW: evaluatorRGSW,
		}
	
		return &crypto_cl
	}
```

This code receives parameters as input and returns a crypto object. Detailed explanations are omitted, and the related code library is attached: [Encrypted_Quanser - Sangwon Lee](https://github.com/lsw23101/Encrypted_Quanser/tree/main).

Next is the function that returns `info_enc`, which can be freely used by the controller.
```Go
	func Enc_for_intmat(crypto_cl *crypto) *info_enc {
		F_q := [][]float64{
			{0, 0, 0, 0},
			{1, 0, 0, -2},
			{0, 1, 0, 1},
			{0, 0, 1, 2},
		}

		G_q := [][]float64{
			{999, -1409},
			{0, -11372},
			{0, -5881},
			{0, 5664},
		}
	
		H_q := [][]float64{
			{64740, -4882, 141655, 132429},
		}
	
		R_q := [][]float64{
			{4},
			{-1571},
			{-793},
			{776},
		}
	
		ctF := RGSW.EncPack(F_q, crypto_cl.tau, crypto_cl.encryptorRGSW, crypto_cl.levelQ, crypto_cl.levelP, crypto_cl.ringQ, *crypto_cl.params)
		ctG := RGSW.EncPack(G_q, crypto_cl.tau, crypto_cl.encryptorRGSW, crypto_cl.levelQ, crypto_cl.levelP, crypto_cl.ringQ, *crypto_cl.params)
		ctH := RGSW.EncPack(H_q, crypto_cl.tau, crypto_cl.encryptorRGSW, crypto_cl.levelQ, crypto_cl.levelP, crypto_cl.ringQ, *crypto_cl.params)
		ctR := RGSW.EncPack(R_q, crypto_cl.tau, crypto_cl.encryptorRGSW, crypto_cl.levelQ, crypto_cl.levelP, crypto_cl.ringQ, *crypto_cl.params)
	
		enc4intmat := info_enc{
			r:             1000.0,
			s:             1000.0,
			L:             1000000.0,
			params:        crypto_cl.params,
			ringQ:         crypto_cl.ringQ,
			monomials:     crypto_cl.monomials,
			tau:           crypto_cl.tau,
			evaluatorRLWE: crypto_cl.evaluatorRLWE,
			evaluatorRGSW: crypto_cl.evaluatorRGSW,
			ctF:           ctF,
			ctG:           ctG,
			ctH:           ctH,
			ctR:           ctR,
		}
	
		return &enc4intmat
	}
```

In this code, `crypto` is received as input, and `info_enc` is returned. `info_enc` includes the following information:
1. quantization level
2. (R)GSW error handling parameters `L`
3. crypto parameter(Not include Key pair)
4. RGSW parameters
5. evaluator
6. encrypted matrix

Refer to [CDSL/ctrRGSW/packing](https://github.com/CDSL-EncryptedControl/CDSL/tree/main/ctrRGSW/packing) to see how this encryption is structured.

**`F_q`, `G_q`, `H_q`, `R_q` must be set to the values output from Python, as explained previously.**
### Encrypted
#### Import
```Go
	import (
		"fmt"
		"time"
	
		tccp "github.com/RFA0608/QQS3C/go"
	
		utils "github.com/CDSL-EncryptedControl/CDSL/utils"
		RLWE "github.com/CDSL-EncryptedControl/CDSL/utils/core/RLWE"
	)
	
	const host = "localhost"
	const port = "9999"
```
This imports the client code for communicating with the server and the library for properly using the implemented cryptography.
#### Main
This part is similar to [[02_Python based controller description#Observer#Encrypted|Python based controller description>Observer>Encrypted]]. However, because this code uses a special encryption form called RGSW, preliminary preparation is required.
```Go
	// set init encrypted state
	xBar := utils.RoundVec(utils.ScalVecMult((info_enc.r * info_enc.s), x_ini))
	xCtPack := RLWE.EncPack(xBar, info_enc.tau, info_enc.L, *crypto_cl.encryptorRLWE, info_enc.ringQ, *info_enc.params)
```
The encrypted state variable, which will be continuously computed in the encrypted controller, is declared in advance.

```Go
	// Quantize and encrypt
	yBar := utils.RoundVec(utils.ScalVecMult(info_enc.r, y))
	yCtPack := RLWE.EncPack(yBar, info_enc.tau, info_enc.L, *crypto_cl.encryptorRLWE, info_enc.ringQ, *info_enc.params)

	// Re-encrypt output
	uBar := utils.RoundVec(utils.ScalVecMult(info_enc.r, u))
	uReEnc := RLWE.Enc(uBar, info_enc.L, *crypto_cl.encryptorRLWE, info_enc.ringQ, *info_enc.params)
```
It performs encryption and re-encryption for the plant output and the controller input.

```Go
	// controller description //
	// ------------------------------------------------ //
	// state update on ciphertext
	xCtPack = Intmat_state_update_enc(xCtPack, yCtPack, uReEnc, info_enc)

	// get output on ciphertext
	uCtPack := Intmat_get_output_enc(xCtPack, info_enc)
	// ------------------------------------------------ //
```
In the controller section, the state variable update and the control input are generated in the encrypted domain.

```Go
	u = RLWE.DecUnpack(uCtPack, 1, info_enc.tau, *crypto_cl.decryptorRLWE, 1/(info_enc.r*info_enc.s*info_enc.s*info_enc.L), info_enc.ringQ, *info_enc.params)
```
The generated control input undergoes decryption and de-quantization outside the controller.

Other than this, the code structure is similar to [[02_Python based controller description#Intmat#Quantized|Python based controller description>Intmat>Quantized]]. you can see whole code this: [QQS3C/interface/controller/py/ctrl_intmat_q.py](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/py/integer_matrix/ctrl_intmat_q.py).