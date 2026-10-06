---
title: 03. C++ based controller description
---
# Controller
## ARX
This code must use `HG` and `HL`, which are output when running the Python-implemented `ctrl_arx_q.py`. You can verify this output in the CMD, as referenced in [[02_Python based controller description#ARX#Quantized|Python based controller description>ARX>Quantized]]. The code is located at [QQS3C/interface/controller/cpp](https://github.com/RFA0608/QQS3C/tree/main/interface/controller/cpp).
### Model_enc
In this section, we will explain `crypto` from "model_enc.h". First, let's examine the internal variables.
```Cpp
	// cryptocontext for encryption
    EncryptionParameters parms;
    SEALContext context;

    // key_pair for encryption
	KeyGenerator keygen; 
	SecretKey secret_key;
	PublicKey public_key;
	RelinKeys relin_keys;
	GaloisKeys galois_keys;
	Encryptor encryptor;
	Decryptor decryptor;
	Evaluator evaluator;
	BatchEncoder batch_encoder;
	size_t slot_count;

	// parameters helper (parameters setting)
	EncryptionParameters create_paramters()
	{
		EncryptionParameters parms(scheme_type::bgv);
		size_t poly_modulus_degree = 8192;
		parms.set_poly_modulus_degree(poly_modulus_degree);
		parms.set_coeff_modulus(CoeffModulus::Create(poly_modulus_degree, {57, 57, 57}));
		parms.set_plain_modulus(PlainModulus::Batching(poly_modulus_degree, 38));
		
		return parms;
	};

	PublicKey create_public_key(KeyGenerator& keygen)
	{
		PublicKey publickey;
		keygen.create_public_key(publickey);
		
		return publickey;
	};
```
Here, variables are declared to store the SEALContext and Key set that can be created from Microsoft's SEAL library. Since these variables are implemented as classes and depend on the existence of constructors, parameter helpers like `EncryptionParameters create_parameters()` or `PublicKey create_public_key(KeyGenerator& keygen)` are used.

Next is the constructor.
```Cpp
	crypto(): 
		parms(create_paramters()),
		context(this->parms),
		keygen(this->context),
		secret_key(this->keygen.secret_key()),
		public_key(create_public_key(this->keygen)),
		decryptor(this->context, this->secret_key),
		encryptor(this->context, this->public_key),
		evaluator(this->context),
		batch_encoder(this->context)
	{
		this->keygen.create_relin_keys(this->relin_keys);
		
		vector<int> steps = {1, 2};
		this->keygen.create_galois_keys(steps, this->galois_keys);

		this->slot_count = this->batch_encoder.slot_count();
        };
```
The constructor is written according to dependencies, but we will omit those details. It is sufficient to understand that it creates the SEALContext, parameters, and key set and stores them as internal variables.

Next are the functions designed for output in this class.
```Cpp
	SEALContext get_crypto()
	{
		return this->context;
	};

	RelinKeys get_relinkey()
	{
		return this->relin_keys;
	};

	GaloisKeys get_galoiskeys()
	{
		return this->galois_keys;
	};


	size_t get_slotsize()
	{
		return this->slot_count;
	};
```
Since these parts must be used from outside this class, they are configured to provide access to the internal variables through return values.

Next are the functions that accept a `vec` (vector) for encryption and decryption.
```Cpp
	Ciphertext enc_vector(vector<int64_t> vec)
	{
		// make plaintext with packing
		Plaintext plain;
		this->batch_encoder.encode(vec, plain);

		// make ciphertext with encryptor
		Ciphertext cipher;
		this->encryptor.encrypt(plain, cipher);

		return cipher;
	};

	vector<int64_t> dec_ciphertext(Ciphertext cipher)
	{
		// make plaintext with decryptor
		Plaintext plain;
		this->decryptor.decrypt(cipher, plain);

		// make vector with unpacking
		vector<int64_t> vec;
		this->batch_encoder.decode(plain, vec);

		return vec;
	};
```

---

Next is the `enc_for_arx` class, which is no different from what is implemented in [[02_Python based controller description#ARX#Model Enc|Python based controller description>ARX>Model Enc]]. However, the difference is that the following hard-coded part has been added.
```Cpp
	// you can get HG_q and HL_q from controller/py/controller.py file
	int64_t HG_q[4][2] = {{-16517, 23273},
                          {86679, -124076},
                          {-134363, 212506},
                          {64740, -118706}};
    int64_t HL_q[4][1] = {{-78},
                          {231},
                          {-287},
                          {281}};
```
**The values in this section must be set to the values output from Python, as explained previously.**

---

After this, `arx_enc` also has the same configuration as [[02_Python based controller description#ARX#Model Enc|Python based controller description>ARX>Model Enc]], so it will be omitted.
### Encrypted
This part is also very similar to [[02_Python based controller description#ARX|Python based controller description>ARX]], except that it is implemented in C++.
#### Include
```Cpp
	// get tcp_protocol decription
	#include "../../../cpp/tcp_protocol_client.h"
	
	// get other tools
	#include <iostream>
	#include <string>
	#include <chrono>
	using namespace std;
	
	// get model description
	#include "model_enc.h"
	
	// init tcp host and port
	const string host = "127.0.0.1";
	const int port = 9999;
```
It consists of code that includes client headers for data communication code for basic (utility) functions and a header file for cryptography.
#### Main
```Cpp
	crypto crypto_cl = crypto();
    enc_for_arx enc_4_arx = enc_for_arx(crypto_cl);
    enc_4_arx.set_level(1000, 1000);
    arx_enc arx_enc_v = arx_enc(crypto_cl.get_crypto(), crypto_cl.get_relinkey(), crypto_cl.get_galoiskeys());
    arx_enc_v.set_pq(enc_4_arx.get_PQ_enc());
    arx_enc_v.set_io(enc_4_arx.get_Z_enc());
```
This is the class object creation code for cryptographic control. The subsequent content is omitted as it is very similar to [[02_Python based controller description#ARX|Python based controller description>ARX]].