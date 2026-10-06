---
title: 03. Controller model
---
# Notation
- $x$ and $\hat{x}$ are the **state variables** of the controller, and their dimension is $\mathbb{R}^{4}$.
- $A$, $B$, and $C$ are **matrices** with dimensions $\mathbb{R}^{4\times4}$, $\mathbb{R}^{4\times1}$, and $\mathbb{R}^{2\times4}$, respectively.
- $L$ and $K$ are the obtained **gains**, with dimensions $\mathbb{R}^{4\times2}$ and $\mathbb{R}^{1\times4}$, respectively.
- $R$ is the obtained **gain**, with dimension $\mathbb{R}^{4\times1}$.
- $[\cdot]_{q}$ denotes the **quantized value**.
- $[\cdot]^{i}$ denotes the **$i$-th row**.
- $[\cdot]^{1 \times j}$ denotes the **$j$-th column**.
# Observer
## Basic
The **continuous-time ABCD equivalent model** of the Quanser Qube Servo 3 is discretized at a **desired sampling time** using the **'zoh'** (Zero-Order Hold) method. Based on this discretized ABCD model, the following observer is designed:

$$
\hat{x}^{+} = A\hat{x} + Bu + L(y - C\hat{x})
$$

Here, the gain **$L$** is obtained using **pole placement** or **DLQR**.

Next, assuming the controller has access to the full state, the stabilizing control input is formulated as follows:

$$
u = -Kx
$$

Similar to the observer gain, **$K$** is obtained using **pole placement** or **DLQR**.

Since the **separation principle** applies to linear systems, the individually obtained $L$ and $K$ gains are combined as follows:

$$
\hat{x}^{+} = (A-BK-LC)\hat{x} + Ly
$$
$$
u=-K\hat{x}
$$

The resulting controller takes **$y$** as an input and generates the control input **$u$**.
## Quantization
If the above controller is directly quantized, the recursively computed component,

$$
A-BK-LC
$$

will **accumulate quantization parameters (errors)**. Therefore, when quantizing the observer directly, the state variables must be **de-quantized** after each computation and then fed back as the input for the next cycle.

Consequently, the controller can be represented as the following **static function**:

$$
F: (x_q,y_q) \mapsto (x_q,u_q) , \: x\in \mathbb{Z}^4, \: y\in\mathbb{Z}^{2}, \: u \in\mathbb{Z}
$$
$$
F(x_{q},y_{q}) := ([A-BK-LC]_{q} x_{q}+[L]_{q}y_{q}, \: [-k]_{q}x_{q})
$$

## Encryption
The static function representation $F$ of the controller presents a problem for encryption. The reason is that it **does not account for the time consumed by multiplication operations on encrypted data**.

In other words, operations on encrypted data have a **much higher computational cost** (latency) than operations on plaintext. Therefore, **scheduling** of the inputs, outputs, and state updates is required.

We thus conclude that the function $F$ as defined above is **infeasible to implement directly with encryption**. The details regarding this scheduling will be introduced in [[04_Transaction scheduling|Transaction scheduling]].
# Full-state feedback
## Basic
Although the **d/dt filter** implemented in the Quanser Qube Servo 3 model can be used, receiving **y** via the **WSL layer** implies an implementation where an **observer** is applied to obtain the **full state**, and a gain is then applied to that state.

In other words, we assume the **observer exists externally** to the controller, and its role is to provide the full state. The controller then generates the control input by multiplying the obtained state by a gain.

The structure is as follows:

$$
\hat{x}^{+} = (A-BK-LC)\hat{x} + Ly
$$
$$
u = -K\hat{x}
$$

Here, the gains **$L$** and **$K$** can be obtained using **pole placement** or **DLQR**.

In this formulation, the controller takes **$x^{+}$** (the next state) as an input to calculate the control input **$u$**. Refer to [[04_Transaction scheduling|Transaction scheduling]] for the reason why **$x^{+}$** is received instead of $x$ to generate the control input.
## Quantization
Since the observer is assumed to exist externally to the controller, the **state and gain are quantized separately**:

$$
 u_{q} = [-K]_{q} x_{q}
$$

Here, $u_{q}$, $[-K]_{q}$, and $x_{q}$ are the quantized values.
## Encryption
The quantized gain and state are **packed into a single polynomial**. A **component-wise multiplication** is performed, followed by **three rotations** using the **Galois keys**. After each rotation, an addition is performed. This process emulates an **inner product** operation to calculate the final control input.

This part is covered in more detail in the **code documentation** of the **Implementation** section.
# ARX
## Basic
The observer-based controller, given as:

$$
\hat{x}^{+} = (A-BK-LC)\hat{x} + Ly
$$
$$
u = -K\hat{x}
$$

is rearranged into the following representation:

$$
x^{+} = Fx +Gy
$$
$$
u=Hx
$$

At this point, if the controller pair **$(F, H)$ is observable**, a gain **$R$** can be found to make the matrix **$F-RH$ nilpotent**.

$$
x^{+} = (F-RH)x+Gy+Ru
$$

Since the matrix **$F-RH$** is now **nilpotent**, the state variable $x(t)$ at time $t$ has no influence on $x(t+4)$, such that **$(F-RH)^{4}x(t) = 0$**. Therefore, $x(t)$ can be implemented as a **static function** composed of the input $y$ and output $u$.

$$
x(t) = \sum_{i=0}^{3}\{(F-RH)^{i}Gy(t-i-1) + (F-RH)^{i}Ru(t-i-1)\}
$$
The recursive part can be eliminated by **storing past values of $y$ and $u$**.

We want the controller's next control input, which is $u(t) = Hx(t)$, so we can conclusively obtain the following:

$$
u(t) = \sum_{i=0}^{3}\{H(F-RH)^{i}Gy(t-i-1) + H(F-RH)^{i}Ru(t-i-1)\}
$$

## Quantization
The matrices $H(F-RH)G$ and $H(F-RH)R$ are quantized, along with the inputs $y(t-i-1)$ and $u(t-i-1)$. The following quantized $u_{q}(t)$ can be obtained:

$$
u_{q}(t) = \sum_{i=0}^{3}\{[H(F-RH)^{i}G]_{q}y_{q}(t-i-1) + [H(F-RH)^{i}R]_{q}u_{q}(t-i-1)\}
$$

## Encryption
The matrices $[H(F-RH)G]_{q}$ and $[H(F-RH)R]_{q}$ have dimensions of **$4 \times 2$** and **$4 \times 1$**, respectively. The rows of each matrix are **packed into a single polynomial**.

$$
\{[H(F-RH)G]_{q}^{i}, [H(F-RH)R]_{q}^{i}\}, \: i=1,..,4
$$

Next, the values of $y(t-i-1)_{q}$ and $u(t-i-1)_{q}$ are packed as follows:

$$
\{y(t-i-1)_{q}^{T},u(t-i-1)_{q}\}, \: i=1,...,4
$$

By proceeding this way, an operation is performed on each set of packed data. After summing them all, **two rotations** are performed to compute the control input via an inner product operation.

This part is covered in more detail in the **code documentation** of the **Implementation** section.
# Matrix conversion to integer
## Basic
Let's assume an observer-based controller design is given as:

$$
\hat{x}^{+} = (A-BK-LC)\hat{x} + Ly
$$
$$
u = -K\hat{x}
$$

Similar to an ARX transformation, this is simplified into the following controller representation:

$$
x^{+} = Fx +Gy
$$
$$
u=Hx
$$

If the transformed controller pair **$(F, H)$ is observable**, it can be re-expressed in the following form:

$$
x^{+} = (F-RH)x+Gy+Ru
$$
$$
u=Hx
$$

For this re-expressed system, a gain **$R$ exists** that makes the matrix $(F-RH)$ have **integer poles**, and this $R$ is found using **pole placement**.

After that, we find an **invertible matrix $T$** to transform the $(F-RH)$ matrix into an **integer matrix**, and perform a coordinate transformation as follows:

$$
z=Tx
$$
$$
z^{+}=T(F-RH)T^{-1}z + TGy +TRu
$$
$$
u = HT^{-1}z
$$

Since we set $R$ such that all poles of $(F-RH)$ are integers, the matrix $T$ (which makes $T(F-RH)T^{-1}$ entirely integer) can be found by transforming the system into a **canonical form**. We trivially accept that the denominator coefficients of the transfer function for a system with all integer poles are themselves integers.
## Quantization
Since we converted the recursive computation part ($T(F-RH)T^{-1}$) into an integer matrix in the previous step, the only matrix parts that need to be quantized are **$TG$**, **$TR$**, and **$HT^{-1}$**. For the signals, only **$y$** and **$u$** need to be quantized.

$$
z^{+} = [T(F-RH)T^{-1}]_{q}z + [TG]_{q}y_{q} + [TR]_{q}u_{q}
$$
$$
u_{q} = [HT^{-1}]_{q}z
$$

## Encryption
The rows of the matrices $[T(F-RH)T^{-1}]_q$, $[TG]_q$, and $[TR]_q$ (e.g., indexed by $j$) are each **packed into a polynomial** and encrypted. Each element of $z$ is copied 4 times and **packed into a polynomial** as $\{z(i), z(i), z(i), z(i)\}, \: i=1,...,4$ and then encrypted. $y$ and $u$ are structured similarly.

For the **recursive computation**, a **special encryption scheme** is used, which relies on **closed-loop stability** to ensure that the error remains appropriately **bounded**.

This part is covered in more detail in the **code documentation** of the **Implementation** section.
