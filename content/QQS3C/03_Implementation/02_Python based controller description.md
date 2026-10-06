---
title: 02. Python based controller description
---
# Controller
This page contains the code description for the controller that operates as a pair with the hardware and simulation implemented in [[01_Python plant and transaction|Python plant and trasaction]]. It is an explanation of the code for [QQS3C/interface/controller/py](https://github.com/RFA0608/QQS3C/tree/main/interface/controller/py), which is run in Python. Since most of the code is a variation of an observer-based design, it shares the observer-based structure, so please refer to it.
## Observer 
### Model
This is an explanation of the `obs` class within "model.py". First, let's check the internal variables of this class.
```Python
	# sampling peroid
    ts = 0.1

    # system(state) matrix(state space linearlization - discrete model)
    A = np.zeros((4,4), dtype=float)
    B = np.zeros((4,1), dtype=float)
    C = np.zeros((2,4), dtype=float)
    D = np.zeros((2,1), dtype=float)

    # output gain K and observer gain L
    K = np.zeros((1,4), dtype=float)
    L = np.zeros((4,2), dtype=float)

    # observer controller state space model
    F = np.zeros((4,4), dtype=float)
    G = np.zeros((4,2), dtype=float)
    H = np.zeros((1,4), dtype=float)
    J = np.zeros((1,2), dtype=float) 

    # state and output
    x = np.zeros((4,1), dtype=float)
    u = np.zeros((1,1), dtype=float)
```
It includes: the sampling time, the discrete-time ABCD matrices, variables to store the gains $K$ and $L$ used in the [[03_Controller model#Observer|Controller model>Observer]], the variables `F`, `G`, `H` (which will hold $A-BK-LC$, $L$, and $-K$) and `J` (which is not used), as well as variables to store the state values and the output.

Next, I will check the constructor.
```Python
	def __init__(self, ts):
        # write your continous time linear model
        A = np.array([[0, 0, 1, 0],
                      [0, 0, 0, 1],
                      [0, 149.275096865093, -0.0130994260613721, 0],
                      [0, 261.609107366662, -0.0129471071536817, 0]], dtype=float)
        B = np.array([[0],
                      [0],
                      [55.6948386963098],
                      [55.0472242928643]], dtype=float)
        C = np.array([[1, 0, 0, 0],
                      [0, 1, 0, 0]], dtype=float)
        D = np.array([[0],
                      [0]], dtype=float)
        sys_c = ct.ss(A, B, C, D)
        sys_d = sys_c.sample(ts, method='zoh')

        # save discretization value
        self.A = sys_d.A
        self.B = sys_d.B
        self.C = C
        self.D = D
        self.ts = ts

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

        # save gain K and L
        self.K = K
        self.L = L.T

        # make a controller state space matrix
        self.F = self.A - self.B @ self.K - self.L @ self.C
        self.G = self.L
        self.H = -self.K

        # ------------------------------------------------ #

        # set initial point of state
        x_init = np.array([[0],
                           [0],
                           [0],
                           [0]], dtype=float)
        
        # save initial point
        self.x = x_init
```
The constructor has the continuous-time ABCD equivalent model already hardcoded and takes ts (sampling time) as an argument to discretize it using 'zoh' (Zero-Order Hold). It stores this in the internal variables and uses DLQR (Discrete Linear Quadratic Regulator) to find the observer gain and the output gain. The DLQR parameters $Q$ and $R$ are hardcoded, but you must adjust them appropriately to find a controller that works well. Similarly, after obtaining these, the code is structured to store `F`, `G`, `H`, and the initial value of x as the state.

Next is the state update function.
```Python
	def state_update(self, y):
        # update state on temp variable
        x_next = self.F @ self.x + self.G @ y
        
        # save state
        self.x = x_next
```
It is structured to update the state variables of the controller.

Next is the output function.
```Python
	def get_output(self):
        self.u = self.H @ self.x

        return self.u
```
It is composed of code that computes and returns the actual control input.

---
This section is an explanation of the `obs_q` class. Following the same flow, we will check the internal variables first.
```Python
	# quantized level s for matrix and r for signal
    s = 1
    r = 1

    # gain of y and x 
    F = np.zeros((4,4), dtype=float)
    G = np.zeros((4,2), dtype=float)
    H = np.zeros((1,4), dtype=float)
    F_q = np.zeros((4,4), dtype=int)
    G_q = np.zeros((4,2), dtype=int)
    H_q = np.zeros((1,4), dtype=int)

    # state and input/output
    x_q = np.zeros((4,1), dtype=int)
    x = np.zeros((4,1), dtype=float)
    y_q = np.zeros((2,1), dtype=int)
    u_q = np.zeros((1,1), dtype=int)
    u = np.zeros((1,1), dtype=float)

```
The internal variables consist of the quantization level, variables to store `F`, `G`, and `H`, variables to store their quantized counterparts, and variables to store the state, input, output, and their respective quantized values.

In this case, the quantization level is a value greater than 1 and is calculated as follows.

$$
\text{value}_q := \lceil s \cdot \text{value} \rfloor 
$$

The formula above is the quantized value of $\text{value}$ when $s$ is considered the quantization level. The de-quantization is as follows.

$$
\text{value}_{dq} = \text{value}/s
$$

It is simply the same as dividing by the quantization level.

Returning to the code explanation, I will now look at this code's constructor.
```Python
	def __init__(self, F, G, H):
        self.F = F
        self.G = G
        self.H = H
```
The constructor receives `F`, `G`, and `H`, which indicates that `F`, `G`, and `H` must be calculated by the `obs` class and then passed to this class.

Next is setting the level.
```Python
	def set_level(self, r, s):
        self.r = r
        self.s = s
```
This is the function that sets the quantization level.

The following is the code for the quantization function.
```Python
	def quantize(self):
        self.F_q = (self.s * self.F).astype(int)
        self.G_q = (self.s * self.G).astype(int)
        self.H_q = (self.s * self.H).astype(int)
```
It consists of code that quantizes the `F`, `G`, and `H` received from `obs` and stores them.

The state update code that follows is as below.
```Python
	def state_update(self, x, y):
        # update state with quantization
        for i in range(2):
            self.y_q[i, 0] = int(self.r * y[i, 0])
        for i in range(4):
            self.x_q[i, 0] = int(self.r * x[i, 0])

        self.x_q = self.F_q @ self.x_q + self.G_q @ self.y_q
        self.x = self.x_q.astype(float) / self.r / self.s
        
        return self.x
```
According to [[03_Controller model#Observer#Quantization|Controller model>Observer>Quantization]], it directly shows that when simply quantizing the observer, which is a dynamic controller, it must be defined as a static function.

Next is the output function.
```Python
    
    def get_output(self):
        self.u_q = self.H_q @ self.x_q
        self.u[0, 0] = float(self.u_q[0, 0]) / self.r / self.s / self.s

        return self.u
```
The output function delivers the quantized value after de-quantization.
#### Original
This section includes the explanation for "ctrl_obs.py".
##### Import
```Python
	# get tcp_protocol description (vscode[debuger] launched at root directory)
	import sys
	sys.path.append(r"./py")
	import tcp_protocol_client as tcc
	
	# init tcp host and port
	HOST = 'localhost'
	PORT = 9999
	
	# get model description
	import model
	
	# get other tools
	import numpy as np
```
It should be noted that the debugger running this must be operated from the **root folder**. It consists of the section for importing the server and the section for importing the code file where the model is written.
##### Main
Let's look at the `def observer_based_controller():` function.

Looking at the main function, it consists of the sampling time and variables to store the operation signal received from the plant.
```Python
    # set simulation(this section have to set same with plant)
    sampling_time = 0.02
    run_signal = True
```
In this section, note that the sampling time **must match** that of the plant.

```Python
	# get model from model description file
	obs = model.obs(sampling_time)
		
	# input/output initialization
	y = np.array([[0],
		          [0]], dtype=float)
	u = np.array([[0]], dtype=float)
```
The code above instantiates an object and declares the variables to be used for input and output.

```Python
	with tcc.tcp_client(HOST, PORT) as tccp:
        while run_signal:
            # running signal send for controller
            _, signal = tccp.recv()

            if signal == "run":
                # get plant output
                _, y0 = tccp.recv()
                _, y1 = tccp.recv()
                y[0, 0] = y0
                y[1, 0] = y1

                # send control input data
                tccp.send(u[0, 0])

                # state update and generate output
                obs.state_update(y)
                u = obs.get_output()
            elif signal == "end":
                # end of loop signal get
                run_signal = False
                break
```
The code above creates a client and, similar to the flow in the plant, it first receives a signal to check whether it should operate. If it is the **run signal**, it operates in the sequence of receiving $y$, sending $u$, and updating the state values. If it receives the **end signal**, it exits the loop, and the controller program terminates.
#### Quantized
This section includes the explanation for "ctrl_obs_q.py".
##### Import
This part is omitted as it is the same as "ctrl_obs.py".
##### Main
Regarding `def obs_quantized():`, I will check only the parts that are different from "ctrl_obs.py".

```Python
	# get model from model description file
    obs = model.obs(sampling_time)
    obs_q = model.obs_q(obs.F, obs.G, obs.H)

    # set quantized level and quantize matrix
    obs_q.set_level(1000, 1000)
    obs_q.quantize()
```
After declaring the `obs` class, the `F`, `G`, and `H` from `obs` are passed as inputs when declaring the `obs_q` model. Afterward, the quantization level is set, and quantization is performed.

```Python
	# state update and generate output
    x = obs_q.state_update(x, y)
    u = obs_q.get_output()
```
Furthermore, unlike the (previous) state update function where the state variable was defined internally, resulting in no function output, here `x` exists as the output of the state update function. Referring to "model.py", this is the de-quantized value.
### Model Enc
"model_enc.py" contains the `crypto` class for generating the `crypto_context` for encryption, the `enc_for_obs` class which defines the matrices, signal encryption, etc., needed for the encrypted controller to run, and `obs_enc`, which represents the actual form of the controller. It is important to note that the `obs_enc` class acts as the encrypted controller, and its internal variables **must never contain the secret key**. Refer to this as demonstrating the conceptual separation of the encrypted controller.

First, I will explain the `crypto` class.
```Python
	# cryptocontext for encryption
    paramters = any
    crypto_context = openfhe.CryptoContext

    # key_pair for encrption
    key_pair = openfhe.KeyPair
```
The internal variables of this class include `crypto_context` for encryption parameters and operations, and `key_pair`, which stores the encryption keys.

Next, I will look at the constructor.
```Python
	def __init__(self):
        # parameter setting
        self.parameters = CCParamsBFVRNS()
        self.parameters.SetRingDim(4096)
        self.parameters.SetPlaintextModulus(4294008833)
        self.parameters.SetMultiplicativeDepth(2)
        self.parameters.SetSecurityLevel(SecurityLevel.HEStd_NotSet)
        
        # crypto context setting
        self.crypto_context = GenCryptoContext(self.parameters)
        self.crypto_context.Enable(PKESchemeFeature.PKE)
        self.crypto_context.Enable(PKESchemeFeature.KEYSWITCH)
        self.crypto_context.Enable(PKESchemeFeature.LEVELEDSHE)
        
        # key setting
        self.key_pair = self.crypto_context.KeyGen()
        self.crypto_context.EvalMultKeyGen(self.key_pair.secretKey)
        self.crypto_context.EvalRotateKeyGen(self.key_pair.secretKey, [1, 2, 3, 4])
```
The constructor begins with parameter settings at the top. The parameters include the Ring dimension, the plaintext modulus (a prime value used for packing), the Mult depth (which indicates how many multiplications are permitted), and a section that allows security levels lower than 128 bits.

Below this, the `crypto_context` is generated using these specified parameters, and the relinearization and rotation keys are created.

Next, code is implemented to extract only the `crypto_context` (which does not contain the secret keys) for later use.
```Python
	def get_crypto(self):
        return self.crypto_context
```

Next,
```Python
	def enc_vector(self, vec):
        # make plaintext with packing
        plaintext = self.crypto_context.MakePackedPlaintext(vec)
        
        # make ciphertext with encrypter
        ciphertext = self.crypto_context.Encrypt(self.key_pair.publicKey, plaintext)

        return ciphertext
    
    def dec_ciphertext(self, ciphertext):
        # make plaintext with decrypter
        plaintext = self.crypto_context.Decrypt(ciphertext, self.key_pair.secretKey)

        # make vecter(list) with unpacking
        vector = plaintext.GetPackedValue()

        return vector
```
It is set up to simplify receiving a 'vec' (vector), encrypting it, and receiving the ciphertext to decrypt it.

---

Next, I will look at the `enc_for_obs` class. Its purpose is to prepare in advance by encrypting the matrices. I will look at the internal variables first.
```Python
	crypto_class = crypto

    # quantization level
    r = 1000
    s = 1000

    # gain of x and y
    F_q = np.zeros((4,4), dtype=int)
    G_q = np.zeros((4,2), dtype=int)
    H_q = np.zeros((1,4), dtype=int)

    # encrypted gain
    F_enc = []
    G_enc = []
    H_enc = []

    # encrypted state
    x_enc = []
    x_dec = np.zeros((4,1), dtype=float)
    y_enc = []
    u_dec = np.zeros((1,1), dtype=float)
```
The internal variables include a variable to hold the `crypto` class for encryption/decryption, the quantized `F`, `G`, and `H` matrices, variables to store their encrypted values, and variables to store the encrypted/decrypted state values, the input $y$, and the output $u$.

Next, I will look at the constructor.
```Python
	def __init__(self, crypto_class, F_q, G_q, H_q):
        self.crypto_class = crypto_class
        self.F_q = F_q
        self.G_q = G_q
        self.H_q = H_q

        vec = [-1, -1, -1, -1]
        # encryption and packgin every column vec
        # encryption F_q
        for i in range(4):
            vec[0] = F_q[0, i]
            vec[1] = F_q[1, i]
            vec[2] = F_q[2, i]
            vec[3] = F_q[3, i]
            self.F_enc.append(self.crypto_class.enc_vector(vec))
        # encryption G_q
        for i in range(2):
            vec[0] = G_q[0, i]
            vec[1] = G_q[1, i]
            vec[2] = G_q[2, i]
            vec[3] = G_q[3, i]
            self.G_enc.append(self.crypto_class.enc_vector(vec))
        # encryption H_q
        vec[1] = 0
        vec[2] = 0
        vec[3] = 0
        for i in range(4):
            vec[0] = H_q[0, i]
            self.H_enc.append(self.crypto_class.enc_vector(vec))
        # encryption componentwise of x and y
        # encryption x_init
        for i in range(4):
            vec[0] = 0
            self.x_enc.append(self.crypto_class.enc_vector(vec))
        # encryption y_init
        for i in range(2):
            vec[0] = 0
            self.y_enc.append(self.crypto_class.enc_vector(vec))
```
It includes encrypting the quantized matrices received from the `obs_q` class, as well as encrypting the signals. The encryption method is as follows.

If $F=\pmatrix{a & b \\ c & d}$ and $x=\pmatrix{x_{1} \\ x_{2}}$ exist, they are split into vectors as shown below:

$$
F1 = \pmatrix{a \\ c}, \: F2 = \pmatrix{b \\ d}, \: X1 = \pmatrix{x1 \\ x1}, \: X2=\pmatrix{x2 \\ x2}
$$

These created vectors are each packed into polynomials, encrypted, and represented as follows:

$$
F1_{enc}, \: F2_{enc}, \: X1_{enc}, \: X2_{enc}
$$

Since the operations after encryption are performed element-wise, the following operation is processed:

$$
F1_{enc} \odot X1_{enc }\oplus F2_{enc} \odot X2_{enc}
$$

Here, $\odot$ and $\oplus$ represent ciphertext multiplication and addition, respectively. If the resulting ciphertext is decrypted without errors, the following result is obtained:

$$
\text{calc result} = \pmatrix{a \cdot x1 + b \cdot x2 \\ c \cdot x1 + d \cdot x2}
$$

This result is the same as the result of $Fx$, so encrypted matrix-vector computations are performed using this method.
#### Encrypted
This section includes the explanation for "ctrl_obs_enc.py".
##### Import
```Python
	# get model description
	import model
	import model_enc
```
When comparing this code to "ctrl_obs.py", the only difference is the inclusion of `model_enc`.
##### Main
Regarding `def obs_encrypted():`, I will check only the parts that are different from "ctrl_obs.py".

```Python
	# get crypto model from model_enc
    crypto_cl = model_enc.crypto()
    enc_4_obs = model_enc.enc_for_obs(crypto_cl, obs_q.F_q, obs_q.G_q, obs_q.H_q)
    enc_4_obs.set_level(obs_q.r, obs_q.s)
    obs_enc = model_enc.obs_enc(crypto_cl.crypto_context, enc_4_obs.F_enc, enc_4_obs.G_enc, enc_4_obs.H_enc)
```
It instantiates the objects defined in "model_enc.py" that were created for encryption.

```Python
	# state and plant output value encryption after packing
    x_enc, y_enc = enc_4_obs.enc_signal(x, y)
                
    ## controller description ##
    # ------------------------------------------------ #
    # get control input on ciphertext space
    enc_xn, enc_u = obs_enc.get_output(x_enc, y_enc)
    # ------------------------------------------------ #

    dec_x, dec_u = enc_4_obs.dec_signal(enc_xn, enc_u)

    # de-quantization
    for i in range(4):
        x[i, 0] = float(dec_x[i, 0]) / enc_4_obs.r / enc_4_obs.s 

    u[0, 0] = float(dec_u[0, 0]) / enc_4_obs.r / enc_4_obs.s 

```
If you check this section, it is after receiving $y$ from the plant and sending $u$. At this point, the state variables $x$ and $y$ are encrypted and stored as `x_enc` and `y_enc`. Next, the subsequent state value and output are obtained as `enc_xn` and `enc_u`. These are then de-quantized; $u$ is sent to the plant, and $x$ is re-encrypted.

_This method does not work correctly because the scheduling is not done properly._
## Full-state
### Model
This is an explanation of the `fs` class within "model.py". First, let's check the internal variables of this class.
```Python
	# gain of x
    H = np.zeros((1,4), dtype=float)

    # output
    u = np.zeros((1,1), dtype=float)
```
Since it is a simple method that takes the state value as input and multiplies it by $H$ to produce $u$, only the corresponding variables are declared.

Next is the constructor.
```Python
	def __init__(self, H):
        self.H = H
```
It is a simple code that receives `H` and stores it.

```Python
	def get_output(self, x):
        self.u = self.H @ x;

        return self.u
```
The output function is also simple.

--- 

Next is the explanation for the `fs_q` class.

```Python
	# quantized level s for matrix and r for signal
    s = 1
    r = 1

    # gain of x
    H = np.zeros((1,4), dtype=float)
    H_q = np.zeros((1,4), dtype=int)

    # input/output
    x_q = np.zeros((4,1), dtype=int)
    x = np.zeros((4,1), dtype=float)
    u_q = np.zeros((1,1), dtype=int)
    u = np.zeros((1,1), dtype=float)
```
Looking at the internal variables, there is one for storing the quantization level, a variable that receives, stores, and saves the quantized version of `H`, and variables that store the quantized and original values for each input and output.

Next is setting the level.
```Python
	def set_level(self, r, s):
        self.r = r
        self.s = s
```
This is the function that sets the quantization level.

The following is the code for the quantization function.
```Python
	def quantize(self):
        self.H_q = (self.s * self.H).astype(int)
```
It quantizes `H`.

```Python
	def get_output(self, x):
        for i in range(4):
            self.x_q[i, 0] = int(self.r * x[i, 0])

        self.u_q = self.H_q @ self.x_q
        
        self.u[0,0] = float(self.u_q[0, 0]) / self.r / self.s

        return self.u
```
Finally, this is the output code. It quantizes the input `x`, generates `u_q`, and then de-quantizes it before sending it out.
#### Original
The "ctrl_fs.py" part is similar to "ctrl_obs.py", so it will be explained briefly.

```Python
	# get model from model description file
    obs = model.obs(sampling_time)
    fs = model.fs(obs.H)
```
The `obs` object and the `fs` object are created respectively. The reason `obs` must be configured is as follows:

- The Plant sends out two outputs, but due to its dynamic structure, it is a model with four state variables. Therefore, the derivative for each output must exist, but since this cannot be obtained, the state variables from the observer are treated as if they were the model's state variables.
    

After this, the rest is almost identical to "ctrl_obs.py".
```Python
	# state update and generate output
    obs.state_update(y)
    u = fs.get_output(obs.x)
```
However, it is specified that the state variable update and the output are handled through separate objects, as shown above.
#### Quantized
The "ctrl_fs_q.py" part is similar to "ctrl_obs_q.py", so it will be explained briefly.
```Python
	# get model from model description file
    obs = model.obs(sampling_time)
    fs = model.fs(obs.H)
    fs_q = model.fs_q(fs.H)

    # set quantized level and quantize matrix
    fs_q.set_level(1000, 1000)
    fs_q.quantize()
```
The `obs` class is used for updating the Plant's state variables, while the rest is constructed using the `fs` and `fs_q` classes. The relationship between `fs` and `fs_q` is the same as the relationship between `obs` and `obs_q`.

```Python
	# state update and generate output
    obs.state_update(y)
    u = fs_q.get_output(obs.x)
```
Just as in the non-quantized case, it creates the state update and the output.
### Model Enc
Since the `crypto` class in "model_enc.py" is used in all Python encryption, its explanation is replaced by the description in the section above. This part also includes the same implementation as `enc_for_obs` and `obs_enc`. Therefore, the code explanation is omitted.

The different part here is the method of packing the variables. In the case of `obs_enc`, packing was done by replicating each element of the state variable, but here, the state variable is packed directly. Therefore, the encryption and decryption proceed in the following order.

To perform the multiplication of $H=\pmatrix{a & b & c & d}$ and $x=\pmatrix{x1 \\ x2 \\ x3 \\ x4}$ under encryption, they are encrypted in this form:

$$
H_{enc}, \: x_{enc}
$$

This is computed using encrypted multiplication:

$$
\text{result}_{enc} = H_{enc} \odot x_{enc}
$$

Decrypting this result directly yields the following:

$$
\text{result} = \pmatrix{a \cdot x1 & b \cdot x2 & c \cdot x3 & d \cdot x4}
$$

Since $u$ cannot be generated from this form, if we sum $\text{result}_{enc}$ while rotating it three times to the left, we can obtain the following result.
$$
\text{result}(0) = a \cdot x1 + b \cdot x2 + c \cdot x3 + d \cdot x4
$$
#### Encrypted
Since "ctrl_fs_enc.py" has the same structure as "ctrl_obs_enc.py", I will proceed simply.

```Python
	# get crypto model from model_enc
    crypto_cl = model_enc.crypto()
    enc_4_fs = model_enc.enc_for_fs(crypto_cl, fs_q.H_q)
    enc_4_fs.set_level(fs_q.r, fs_q.s)
    fs_enc = model_enc.fs_enc(crypto_cl.crypto_context, enc_4_fs.H_enc)
```
The classes related to cryptography are instantiated in "ctrl_fs_q.py".

```Python
	# state estimation on plant and encryption
    obs.state_update(y)
    x_enc = enc_4_fs.enc_signal(obs.x)
                
    ## controller description ##
    # ------------------------------------------------ #
    # get control input on ciphertext space
    enc_u = fs_enc.get_output(x_enc)
    # ------------------------------------------------ #

    int_u = enc_4_fs.dec_signal(enc_u)

    u[0, 0] = float(int_u) / enc_4_fs.r / enc_4_fs.s
```
The state is updated using the $y$ received from the Plant, and that state is encrypted and saved as `x_enc`. An operation is performed to take the inner product of this with `H_enc` to create `enc_u`, which is then decrypted and de-quantized to generate the control input.
## ARX
### Model
This is an explanation of the `arx` class within "model.py". First, let's check the internal variables of this class.
```Python
	# observer controller state space model
    F = np.zeros((4,4), dtype=float)
    G = np.zeros((4,2), dtype=float)
    H = np.zeros((1,4), dtype=float)
    J = np.zeros((1,1), dtype=float)

    # gain for coordination transform
    L = np.zeros((4,1), dtype=float)

    # sequence of y and u like y0, y1, ...,y3 and u0, ...,u3
    # every y is column vector but save is row vector
    Ys = np.zeros((4,2), dtype=float)
    Us = np.zeros((4,1), dtype=float)

    # gain of y sequence and u sequence
    HG = np.zeros((4,2), dtype=float)
    HL = np.zeros((4,1), dtype=float)

    # output
    u = np.zeros((1,1), dtype=float)
```
The variables brought from the observer design and the $R$ gain for converting to ARX are denoted as `L`. The output and input sequences, the matrices multiplied by the output and input sequences, and the output are defined. The meaning of each variable can be found in [[03_Controller model#ARX|Controller model>ARX]].

Next, we move on to the constructor.
```Python
	def __init__(self, F, G, H, J):
        # save original controller's state matrix
        self.F = F
        self.G = G
        self.H = H
        self.J = J
        
        O_matrix = ct.obsv(F, H)
        rank = matrix_rank(O_matrix)
        if rank != 4:
            is_obs = 0
            print(f"This controller is not observable")
        else:
            is_obs = 1
            
            # all desired_poles set to zero
            desired_poles = np.zeros(4)

            # calc L, which is gain that F-LH can be nilpotent
            L = ct.place_acker(self.F.T, self.H.T, desired_poles)

            # save gain L
            self.L[0, 0] = L[0]
            self.L[1, 0] = L[1]
            self.L[2, 0] = L[2]
            self.L[3, 0] = L[3]
            
            # validation of gain L
            eigenvalues = eigvals(self.F - self.L @self.H)
            print(f"eigenvalue of F - LH:\n {eigenvalues}")

            # calc H(F-LH)^3G ... HG and H(F-LH)^3L ... HL
            self.HG[0,:] = (self.H @ (self.F - self.L @ self.H) @ (self.F - self.L @ self.H) @ (self.F - self.L @ self.H) @ self.G)
            self.HG[1,:] = (self.H @ (self.F - self.L @ self.H) @ (self.F - self.L @ self.H) @ self.G)
            self.HG[2,:] = (self.H @ (self.F - self.L @ self.H) @ self.G)    
            self.HG[3,:] = (self.H @ self.G)   

            self.HL[0,:] = (self.H @ (self.F - self.L @ self.H) @ (self.F - self.L @ self.H) @ (self.F - self.L @ self.H) @ self.L)
            self.HL[1,:] = (self.H @ (self.F - self.L @ self.H) @ (self.F - self.L @ self.H) @ self.L)
            self.HL[2,:] = (self.H @ (self.F - self.L @ self.H) @ self.L)    
            self.HL[3,:] = (self.H @ self.L)

```

The key point to check in this section is that `L` (which corresponds to $R$ in the code) is obtained using Ackermann's pole placement. You can also see that `HG` and `HL` are implemented by stacking $H(F-RH)^{i}G$ and $H(F-RH)^{i}R$, respectively.

Next, as is characteristic of an ARX model, is the section that updates using previous inputs and outputs.
```Python
	def mem_update(self, Yn, Un):
        # i = 0 is oldest value
        for i in range(3):
            self.Ys[i,:] = self.Ys[(i+1),:]
            self.Us[i,:] = self.Us[(i+1),:]

        # new value input in i = 3
        self.Ys[3,:] = Yn.T
        self.Us[3,:] = Un
```
It consists of code that replaces the input and output sequences with new ones.

Finally, here is the output function.
```Python
	def get_output(self):
        self.u[0, 0] = 0

        for i in range(4):
            self.u = self.u + self.HG[i,:] @ self.Ys[i,:].T + self.HL[i,:] @ self.Us[i,:]
        
        return self.u
```

---

Next is the explanation for the `arx_q` class. First are the internal variables.
```Python
	# quantized level s for matrix and r for signal
    s = 1
    r = 1

    # sequence of y and u like y0, y1, ...,y3 and u0, ...,u3
    # every y is column vector but save is row vector
    Ys_q = np.zeros((4,2), dtype=int)
    Us_q = np.zeros((4,1), dtype=int)

    # gain of y sequence and u sequence
    HG = np.zeros((4,2), dtype=float)
    HL = np.zeros((4,1), dtype=float)
    HG_q = np.zeros((4,2), dtype=int)
    HL_q = np.zeros((4,1), dtype=int)

    # output
    u_q = np.zeros((1,1), dtype=int)
    u = np.zeros((1,1), dtype=float)
```
Similar to `obs_q`, variables are declared for the quantization level quantization for the input/output sequences quantization for the matrices multiplied with the sequences and the output variable.

Next, I will look at the constructor.
```Python
	def __init__(self, HG, HL):
        self.HG = HG
        self.HL = HL
```
It receives the matrices multiplied by the sequences, exactly as implemented in the `arx` class.

As for the rest,
```Python
	 def quantize(self):
        self.HG_q = (self.s * self.HG).astype(int)
        self.HL_q = (self.s * self.HL).astype(int)

    def mem_update(self, Yn, Un):
        # i = 0 is oldest value
        for i in range(3):
            self.Ys_q[i,:] = self.Ys_q[(i+1),:]
            self.Us_q[i,:] = self.Us_q[(i+1),:]

        # new value input in i = 3
        self.Ys_q[3,:] = (self.r * Yn.T).astype(int)
        self.Us_q[3,:] = (self.r * Un).astype(int)

    def get_output(self):
        self.u_q[0,0] = 0

        for i in range(4):
            self.u_q = self.u_q + self.HG_q[i,:] @ self.Ys_q[i,:].T + self.HL_q[i,:] @ self.Us_q[i,:]
        
        self.u[0,0] = float(self.u_q[0, 0]) / self.r / self.s

        return self.u
```

The explanation for the rest is omitted.
#### Original
The "ctrl_arx.py" part is similar to "ctrl_obs.py", so it will be explained briefly.
```Python
	# get model from model description file
    obs = model.obs(sampling_time)
    arx = model.arx(obs.F, obs.G, obs.H, obs.J)
```
It utilizes the matrices implemented in `obs` to structure the ARX form.

```Python
	# arx memory update and generate output
    arx.mem_update(y, u)
    u = arx.get_output()
```
The state update and output calculation have been changed to memory update and output calculation.
#### Quantized
The "ctrl_arx_q.py" part is similar to "ctrl_obs_q.py", so it will be explained briefly.

```Python
	# get model from model description file
    obs = model.obs(sampling_time)
    arx = model.arx(obs.F, obs.G, obs.H, obs.J)
    arx_q = model.arx_q(arx.HG, arx.HL)

    # set quantized level and quantize matrix
    arx_q.set_level(1000, 1000)
    arx_q.quantize()

    # print matrix of HG_q and HL_q
    print(arx_q.HG_q)
    print(arx_q.HL_q)
```
First, the `obs` class is declared, and then the `arx` and `arx_q` classes are created sequentially. Afterward, the quantization level is set, and quantization is performed. **Additionally, the quantized `HG` and `HL` values obtained in this section are printed. Note this, as these values will be used directly in the C++ implementation.**  The link is modified to C++ version of `arx_enc`: [[03_C++ based controller description|C++ based controller decription]].

Next, the state update and output functions are as follows.
```Python
	# arx memory update and generate output
    arx_q.mem_update(y, u)
    u = arx_q.get_output()
```
### Model Enc
Since the `crypto` class in "model_enc.py" is used in all Python encryption, its explanation is replaced by the description in the section above. This part also includes the same implementation as `enc_for_obs` and `obs_enc`. Therefore, the code explanation is omitted.

An additional part here is the method for variable packing. The following is the complete expression for the output time, showing how the actual ARX calculation is performed:

$$
H(F-RH)^{3}G \cdot \pmatrix{y0(t-3) & y1(t-3)}^{T} + H(F-RH)^{3}R \cdot u(t-3)
$$

$$
H(F-RH)^{2}G \cdot \pmatrix{y0(t-2) & y1(t-2)}^{T} + H(F-RH)^{2}R \cdot u(t-2)
$$
$$
H(F-RH)G \cdot \pmatrix{y0(t-1) & y1(t-1)}^{T} + H(F-RH)R \cdot u(t-1)
$$
$$
HG \cdot \pmatrix{y0(t) & y1(t)}^{T} + HR \cdot u(t)
$$

The new output is created by summing all of these. Therefore, encrypting and computing this uses the following method:

1. Pack and encrypt $\{y0(t-k), y1(t-k), u(t-k)\}$ for $k=0,...,3$ respectively.
    
2. Pack and encrypt $\{H(F-RH)^{k}G, H(F-RH)^{k}R\}$ for $k=0,...,3$ respectively.
    

If we call the ciphertext from step 1 $S_{enc}^{k}$ and the ciphertext from step 2 $M_{enc}^{k}$, the encrypted operation is computed as follows:

$$
\text{result}_{enc} = \sum_{i=0}^{3} M_{enc}^{i} \odot S_{enc}^{k}
$$

Afterward, if $\text{result}_{enc}$ is rotated and summed twice, $u$ will appear in the 0th element of its decrypted value, $\text{result}$.
#### Encrypted
Since "ctrl_arx_enc.py" has the same structure as "ctrl_obs_enc.py", I will proceed simply.
```Python
	# get crypto model from model_enc
    crypto_cl = model_enc.crypto()
    enc_4_arx = model_enc.enc_for_arx(crypto_cl, arx_q.HG_q, arx_q.HL_q)
    enc_4_arx.set_level(arx_q.r, arx_q.s)
    arx_enc = model_enc.arx_enc(crypto_cl.crypto_context, enc_4_arx.PQ_enc, enc_4_arx.Z_enc)
```
The classes related to cryptography are instantiated in "ctrl_arx_q.py".

```Python
	# y and u value encryption after packing
    signal = enc_4_arx.enc_signal(y, u)
                
    ## controller description ##
    # ------------------------------------------------ #
    # ctrl mem update on encrypted space after encryption input/output value
    arx_enc.mem_update(signal)
                
    # get control input on ciphertext space
    enc_u = arx_enc.get_output()
    # ------------------------------------------------ #

    int_u = enc_4_arx.dec_signal(enc_u)

    u[0, 0] = float(int_u) / enc_4_arx.r / enc_4_arx.s
```
The $y$ received from the Plant and the re-encrypted $u$ are encrypted to create `signal`. This is updated in memory, and the output is calculated and stored as `enc_u`. Decrypting and de-quantizing `enc_u` yields the control input.
## Intmat
### Model
This is an explanation of the `intmat` class within "model.py". First, let's check the internal variables of this class.
```Python
	# sampling peroid
    ts = 0.1

    # observer controller state space model
    F = np.zeros((4,4), dtype=float)
    G = np.zeros((4,2), dtype=float)
    H = np.zeros((1,4), dtype=float)
    J = np.zeros((2,1), dtype=float) 

    # gain of which convert F-RH's pole to integer 
    R = np.zeros((4,1), dtype=float)

    # converted to int model
    F_cv = np.zeros((4,4), dtype=float)
    G_cv = np.zeros((4,2), dtype=float)
    H_cv = np.zeros((1,4), dtype=float)
    R_cv = np.zeros((4,1), dtype=float) 

    # state and output
    x = np.zeros((4,1), dtype=float)
    u = np.zeros((1,1), dtype=float)
```
The sampling time, `F`, `G`, `H`, `J` created from the observer, `R` (which stores the $R$ that makes the state matrix $F-RH$ into integers), the integer-converted `F_cv`, and the correspondingly coordinate-transformed `G_cv`, `H_cv`, `R_cv`, along with the state variable and output, are declared as variables.

Next, I will check the constructor.
```Python
	def __init__(self, F, G, H, J, ts):
        self.F = F
        self.G = G
        self.H = H
        self.J = J
        self.ts = ts

        # find gain R which convert F-RH's pole to integer
        pole = np.array([[0, 1, 2, -1]])
        R = ct.place(self.F.T, self.H.T, pole).T
        
        # save gain R
        self.R = R
    
        # # This section doesn't support on python-control library
        # # So, you change on MATLAB and save manualy
        # # find canonical model transformed matrix T
        # sys = ct.ss((F-R@H), G, H, J)
        # csys, T = ct.canonical_form(sys, form='reachable')

        # # save converted form
        # self.F_cv = T@(F-R@H)/T
        # self.R_cv = T@R
        # self.G_cv = T@G
        # self.H_cv = H/T

        # save converted matrix manualy from MATLAB
        # you can found the file name of "transpose_matrix2int" on /interface/controller/py/tools folder
        # copy and paste observer state matrix F, G, H to there, get and write below invertible matrix T
        T = np.array([[1.08314828520927, -1.66955395965037, -0.00236729459240089, 0.00841047244267525],
                      [-12.3192475520305, 21.2280825206628, 1.11207418021025, -2.55670558562088],
                      [-6.55696463635024, 10.9797588171927, 0.590938554977826, -1.32609684315751],
                      [6.17193488024287, -10.6209349231961, -0.557028488424479, 1.27749537728892]])
        
        self.F_cv = (T@(F-R@H)@np.linalg.inv(T)).round()
        self.R_cv = T@R
        self.G_cv = T@G
        self.H_cv = H@np.linalg.inv(T)
```
The constructor receives and stores the observer's `F`, `G`, `H`, `J`, and the sampling time. It then finds and stores $R$ in the variable `R`, where $R$ makes the poles of $F-RH$ integers. After that, $F-RH$ needs to be converted to a canonical form to be transformed into integers. Since Python does not support canonical form transformation for MIMO systems, this is done separately in Matlab. Refer to the Matlab file at [QQS3C/interface/controller/tools](https://github.com/RFA0608/QQS3C/tree/main/interface/controller/tools). You put the existing `F`, `G`, `H` into the Matlab file, input the **same poles**, and then write the resulting invertible matrix T directly into the code above. Then, the transformed matrices are stored.

Next is the state update function.
```Python
	def state_update(self, y, u):
        # update state on temp variable
        x_next = self.F_cv @ self.x + self.G_cv @ y + self.R_cv @ u

        # save state
        self.x = x_next
```
Similar to the observer-based design, it receives $y$ and $u$ to update the state.

Next is the output function.
```Python
	def get_output(self):
        self.u = self.H_cv @ self.x
```

---

This section is an explanation of the `intmat_q` class. Following the same flow, we will check the internal variables first.
```Python
	# quantized level s for matrix and r for signal
    s = 1
    r = 1

    # gain of y and x 
    F = np.zeros((4,4), dtype=float)
    G = np.zeros((4,2), dtype=float)
    H = np.zeros((1,4), dtype=float)
    R = np.zeros((4,1), dtype=float) 
    F_q = np.zeros((4,4), dtype=int)
    G_q = np.zeros((4,2), dtype=int)
    H_q = np.zeros((1,4), dtype=int)
    R_q = np.zeros((4,1), dtype=int) 

    # state and input/output
    x_q = np.zeros((4,1), dtype=int)
    y_q = np.zeros((2,1), dtype=int)
    u_q = np.zeros((1,1), dtype=int)
    u = np.zeros((1,1), dtype=float)
```

Variables are declared for the quantization level each of the matrices (like F) converted to integers, and the quantized values.

The rest of the code is omitted as it has the same structure as the quantization classes, such as `obs_q`.
#### Original
This part is similar to "ctrl_obs.py", so it will be explained briefly.

```Python
	# get model from model description file
    obs = model.obs(sampling_time)
    print(f"controller's matrix F: \n{obs.F}")
    print(f"controller's matrix G: \n{obs.G}")
    print(f"controller's matrix H: \n{obs.H}\n")
```
To input each of the observer's matrices into Matlab, the values are displayed in the CMD window.

```Python
	intmat = model.intmat(obs.F, obs.G, obs.H, obs.J, obs.ts)
    print(f"controller's converted matrix F: \n{intmat.F_cv}")
    print(f"controller's converted matrix R: \n{intmat.R_cv}")
    print(f"controller's converted matrix G: \n{intmat.G_cv}")
    print(f"controller's converted matrix H: \n{intmat.H_cv}\n")

```
The transformed model is also generated and displayed. **This is absolutely necessary for the implementation in Go.** [[04_Go based controller description|GO based controller description]].

```Python
	# state update and generate output
    intmat.state_update(y, u)
    u = intmat.get_output()
```
The state update and the output are as shown above.
#### Quantized
This part is similar to "ctrl_obs_q.py", so it will be explained briefly.
```Python
	intmat_q = model.intmat_q(intmat.F_cv, intmat.G_cv, intmat.H_cv, intmat.R_cv)

    # set quantized level and quantize matrix
    intmat_q.set_level(1000, 1000)
    intmat_q.quantize()

    # print matrix of F_q, G_q, H_q, and R_q
    print(f"print F_q, G_q, H_q, and R_q: ")
    print(intmat_q.F_q)
    print(intmat_q.G_q)
    print(intmat_q.H_q)
    print(intmat_q.R_q)
```
There is a part that displays each of the quantized matrices for use in encryption.

The rest all have the same form.

The corresponding encryption code for `intmat` or `intmat_q` only exists in Go.
