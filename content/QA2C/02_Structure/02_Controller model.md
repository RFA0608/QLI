---
title: 02. Controller model
---

# Notation
- $x$ and $\hat{x}$ are the **state variables** of the controller, and their dimension is $\mathbb{R}^{4}$.
- $A$, $B$, and $C$ are **matrices** with dimensions $\mathbb{R}^{4\times4}$, $\mathbb{R}^{4\times2}$, and $\mathbb{R}^{2\times4}$, respectively.
- $L$ and $K$ are the obtained **gains**, with dimensions $\mathbb{R}^{4\times2}$ and $\mathbb{R}^{2\times4}$, respectively.
- $R$ is the obtained **gain**, with dimension $\mathbb{R}^{4\times2}$.
- $[\cdot]_{q}$ denotes the **quantized value**.
- $[\cdot]^{i}$ denotes the **$i$-th row**.
- $[\cdot]^{1 \times j}$ denotes the **$j$-th column**.
# Full-state feedback
## Basic
The Aero 2 model can measure all state variables, so it can be controlled through full-state feedback.

The controller is as follows:

$$
u = -Kx + N_{bar} r.
$$

Here, the gains **$K$** can be obtained using **pole placement** or **DLQR**. In the above part, $N_{bar}$ is a parameter attached to the reference and is the value used to maintain the reference in steady-state. That can easily get as follows:

$$
N_{bar} = (C (I - (A - BK))^{-1}B)^{-1}.
$$

This is not a difference, but rather an expression of respect for

$$
x_{ss}^{+} = Ax_{ss}+Bu_{ss}
$$

$$
y_{ss}=Cx_{ss}
$$

and is expanded as follows:

$$
y_{ss} = r = C x_{ss} \qquad (1)
$$
$$
x_{ss}^+ = x_{ss} = Ax_{ss} -BKx_{ss} + BN_{bar}r \qquad (2)
$$

arrange $(2)$, we can get $(3)$

$$
x_{ss} = (I - (A-BK))^{-1}BN_{bar}r \qquad (3)
$$

put $(3)$ to $(1)$

$$
r = C(I-(A-BK))^{-1}BN_{bar}r \qquad (4)
$$

calculate $(4)$ for $N_{bar}$

$$
\therefore N_{bar} = (C(I-(A-BK))^{-1}B)^{-1}
$$

where $x_{ss}^{+} = x_{ss}$ and $y_{ss} = r$ under steady-state. 
# Observer
## Basic
The **continuous-time ABCD equivalent model** of the Quanser Aero 2 is discretized at a **desired sampling time** using the **'zoh'** (Zero-Order Hold) method. Based on this discretized ABCD model, the following observer is designed:

$$
\hat{x}^{+} = A\hat{x} + Bu + L(y - C\hat{x})
$$

Here, the gain **$L$** is obtained using **pole placement** or **DLQR**.

Next, assuming the controller has access to the full state, the stabilizing control input is formulated as follows:

$$
u = -Kx + N_{bar} r
$$

Similar to the observer gain, **$K$** is obtained using **pole placement** or **DLQR**. $N_{bar}$ and $r$ are value of related on reference. $N_{bar}$ is paramerter such that

$$
N_{bar} = (C (I - (A - BK))^{-1}B)^{-1}
$$

and satisfy so that the output follows the reference $r$.


Since the **separation principle** applies to linear systems, the individually obtained $L$ and $K$ gains are combined as follows:

$$
\hat{x}^{+} = (A-BK-LC)\hat{x} + Ly
$$
$$
u=-K\hat{x} + N_{bar} r
$$


The resulting controller takes **$y$** as an input and generates the control input **$u$**.
# Integral Control
## Problem of $N_{bar}$ prametic method
Calculating $N_bar$ using this parametric method and tracking the reference becomes **highly vulnerable to model uncertainty or external disturbances**, making it impossible to perform an asymptotic approximation of the output to the reference. To solve this problem, I introduce a method of adding an integral to create it.
## Basic
let show output equation: $y(t) = C x(t) = [\theta(t), \psi(t)]^\top$, $\quad C = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \end{bmatrix}$, we can determine errors integral state $x_I(t+1) = x_I(t) + (r - y(t))$ and apply them to **augmented state $X(t) = [x(t)^\top, x_I(t)^\top]^\top$**. Whole augmented system like below:

$$
X(t+1) = \begin{bmatrix} A & 0 \\ -C & I \end{bmatrix} X(t) + \begin{bmatrix} B \\ 0 \end{bmatrix} u(t) + \begin{bmatrix} 0 \\ I \end{bmatrix} r.
$$

For augmented System

$$
X(t+1) = A_{\mathrm{aug}} X(t) + B_{\mathrm{aug}} u(t) + B_r r
$$

apply augmented controller $u(t) = -K X(t) = -K_x x(t) - K_I x_I(t)$:

$$
X(t+1) = (A_{\mathrm{aug}} - B_{\mathrm{aug}} K) X(t) + B_r r.
$$

When we set $(A_{\mathrm{aug}} - B_{\mathrm{aug}}K)$ to **Schur Stable($|\lambda| < 1$)** for Gain $K$, Even if there is external disturbance, the steady-state error converges to zero ($\lim_{t \to \infty} y(t) = r$).
And also original system $(A, C)$ is observable, can make luenberger observer such as:

$$
\hat{X}(t+1) = A_{\mathrm{aug}} \hat{X}(t) + B_{\mathrm{aug}} u(t) + L \left( y(t) - C_{\mathrm{aug}} \hat{X}(t) \right) + B_r r
$$

where $C_{\mathrm{aug}} = \begin{bmatrix} C & 0 \end{bmatrix}$. The point to note here is that there is no observability for the augmented system, so the existing system must be observable and have the condition that the initial values of the incremented state variables can be set. Therefore, in the above form, $L$ is a state in which all values corresponding to the incremented state variables are 0.
## ARX(AutoRegressive & eXogenous inputs)
Put  $u(t) = -K \hat{X}(t)$ in to observer based controller:

$$
\hat{X}(t+1) = F \hat{X}(t) + G y(t) + B_r r 
$$
$$
u(t) = H \hat{X}(t)
$$

where $F = A_{\mathrm{aug}} - B_{\mathrm{aug}} K$, $\quad G = L$, $\quad H = -K$.
With same method, put arbitrary matrix $R$ like below:

$$
\hat{X}(t+1) = (F - RH) \hat{X}(t) + G y(t) + R u(t) + B_r r.
$$

**If $(F, H)$ is observable, then can make all eigenvalue of $F - RH$ to 0, Nilpotent Gain $R$ is always exist.**

$(F - RH)^6 = 0$ is satisfy because $F - RH \in \mathbb{R}^{6 \times 6}$ is Nilpotent. By using this to eliminate the state estimate $\hat{X}(t)$, we can represent $u(t)$ solely from the past history of the input/output:

$$
u(t) = \sum_{i=1}^{6} H (F - RH)^{i-1} \Big( G y(t-i) + R u(t-i) + B_r r \Big)
$$

Summarized in matrix form **ARX Representation**:

$$
u(t) = P Y_{\mathrm{seq}}(t) + Q U_{\mathrm{seq}}(t) + T r
$$

where $Y_{\mathrm{seq}}(t) = [y(t-1)^\top, \dots, y(t-6)^\top]^\top$, $U_{\mathrm{seq}}(t) = [u(t-1)^\top, \dots, u(t-6)^\top]^\top$.