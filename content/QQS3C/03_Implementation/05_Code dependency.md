---
title: 05. Code depenency
---
# Modifiable elements
For tests or demos in this code, the following can be modified.

## Sampling period
If you want to change the sampling time, proceed with the following steps.

### 1. Plant

- **Hardware Demo:** To change the hardware demo sampling time, find and modify `frequency = 40 # hz` in the Plant code at [QQS3C/interface/plant/py/hardware/plant.py](https://github.com/RFA0608/QQS3C/blob/main/interface/plant/py/hardware/plant.py).
    
- **Simulation:** To change the simulation sampling time, modify `sampling_time = 0.02` in [QQS3C/interface/plant/py/simulation/plant.py](https://github.com/RFA0608/QQS3C/blob/main/interface/plant/py/simulation/plant.py).
    

---

### 2. Controller

If you have changed the Plant's sampling time, you must also change the sampling time in the corresponding controller code at [QQS3C/interface/controller](https://github.com/RFA0608/QQS3C/tree/main/interface/controller) to match the Plant.

- **Python:** Modify `sampling_time = 0.02`. 
    
- **C++:** Modify `double samplint_time = 0.02;`. Then, also change `sampling_time = 0.02` in the [QQS3C/interface/controller/py/arx_model/ctrl_arx_q.py](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/py/arx_model/ctrl_arx_q.py) code and input the resulting modified matrices into [QQS3C/interface/controller/cpp/arx_model/model_enc.h](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/cpp/arx_model/model_enc.h).
    
- **Go:** Change `sampling_time = 0.02` in [QQS3C/interface/controller/py/integer_matrix/ctrl_intmat_q.py](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/py/integer_matrix/ctrl_intmat_q.py) and input the results into [QQS3C/interface/controller/go/integer_matrix/model_enc.go](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/go/integer_matrix/model_enc.go).

## DLQR
Find and modify the following in [QQS3C/interface/controller/py/observer_form/model.py](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/py/observer_form/model.py), every folder (like observer_form, state_filter, full_state_feedback, arx_model, integer_matrix) have model description name of "model.py".
```Python
	# for gain K dlqr parameters setting
	Q_k = np.array([[5, 0, 0, 0],
					[0, 1, 0, 0],
					[0, 0, 1, 0],
					[0, 0, 0, 1]], dtype=float)
	R_k = np.array([[1]], dtype=float)
	K, Sk, Ek = ct.dlqr(self.A, self.B, Q_k, R_k)
```

or
```Python
	 # for gain K dlqr parameters setting
	Q_k = np.array([[5000, 0, 0, 0],
					[0, 400, 0, 0],
					[0, 0, 1, 0],
					[0, 0, 0, 1]], dtype=float)
	R_k = np.array([[1]], dtype=float)
	K, Sk, Ek = ct.dlqr(self.A, self.B, Q_k, R_k)
	
	# for gain L dlqr parameters setting
	Q_l = np.array([[1, 0, 0, 0],
					[0, 1, 0, 0],
					[0, 0, 500, 0],
					[0, 0, 0, 100]], dtype=float)
	R_l = np.array([[1, 0],
					[0, 1]], dtype=float)
	L, Sl, El = ct.dlqr(self.A.T, self.C.T, Q_l, R_l)
```
.

This change will modify the gain for all code in "model.py" in [QQS3C/interface/controller/py](https://github.com/RFA0608/QQS3C/tree/main/interface/controller/py).

If desired, it is also acceptable to replace this entire section and use pole placement.

## Security
This code uses the **openFHE-python**, **Microsoft SEAL**, and **lattigo** cryptography libraries. Accordingly, explanations for each are provided.

- **openFHE-python** In "model_enc.py" in every folder in [QQS3C/interface/controller/py](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/py), security can be managed by appropriately setting `self.parameters.SetRingDim(4096)`, `self.parameters.SetPlaintextModulus(4294008833)`, and `self.parameters.SetMultiplicativeDepth(2)`. However, the provided code adds `self.parameters.SetSecurityLevel(SecurityLevel.HEStd_NotSet)` to ensure that the cryptographic operations finish within the sampling time. If you wish to increase security, you can comment out this line.
    
- **Microsoft SEAL** In [QQS3C/interface/controller/cpp/arx_model/model_enc.h](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/cpp/arx_model/model_enc.h), `size_t poly_modulus_degree = 8192;`, `parms.set_coeff_modulus(CoeffModulus::Create(poly_modulus_degree, {57, 57, 57}));`, and `parms.set_plain_modulus(PlainModulus::Batching(poly_modulus_degree, 38));` can be changed.
    
- **lattigo** In [QQS3C/interface/controller/go/integer_matrix/model_enc.go](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/go/integer_matrix/model_enc.go), the section below can be modified.
```Go
	params, _ := rlwe.NewParametersFromLiteral(rlwe.ParametersLiteral{
		LogN:    12,
		LogQ:    []int{60},
		LogP:    []int{60},
		NTTFlag: true,
	})
```

- **Caution:** This code has **hard-coded parameters** that are preset to work correctly. If you modify these arbitrarily, you must **verify** that the new parameter settings function properly.

## Other
You are free to change the entire library code or modify it to fit your own style. However, the following structure is recommended:

1. **Separate the classes:** The class containing the cryptography (components) should be structured separately from the class containing the cryptographic operations.
    
2. **Port changes:** If you want to change the TCP/IP communication port, only change it to a larger number.
    
3. **File separation:** Separate the file for the control loop and the file for the controller representation, as is done in this code.
    

Please note that these are recommendations, not strict requirements.