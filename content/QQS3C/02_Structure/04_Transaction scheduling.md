---
title: 04. Transaction scheduling
---
# Calculation on controller
## General Situation
A typical observer-based controller is structured as follows:

$$
\hat{x}(t+1) = (A-BK-LC)\hat{x}(t) + Ly(t)
$$
$$
u(t)=-K\hat{x}(t)
$$

Looking at the time indices, at time **$t$**, the control input $u(t) = -K\hat{x}(t)$ must be calculated and sent as the input to the plant. At the **same instant**, the measurement $y(t)$ must be received as the input to the controller to calculate the next state, $\hat{x}(t+1)$.

While this calculation is generally very fast, it becomes **significantly slower when encrypted**.

Consequently, the computation of $-Kx$ introduces a delay, causing the control input $u(t)$ to be applied later by an amount equal to the computation time. **This severely impacts the system's stability**, and this problem must be solved.
## Encrypted Control Scenario
To solve the computation time problem in encrypted control, we propose the following control input, output, and computation scheduling:

![[Structure_schedule.png]]
This scenario **synchronizes the timing** of receiving $y$ from the plant and sending $u$ to the plant. The computation for the **next** control input (for the next time step) is then performed using the received $y$ during the **remaining time of the sampling period**.

This method has the advantage of **pre-computing** the next $u$ to be sent, which allows us to overcome the computational delay (latency) that arises in the encrypted control scenario.

This is why the **static function $F$** for the observer in [[03_Controller model#Observer#Encryption|Controller model>Observer>Encryption]] is **infeasible** (does not work). Furthermore, the fact that the state update is performed **in advance**, as explained in the full-state feedback section [[03_Controller model#Full-state feedback#Basic|Controller model>Full-state feedback>Basic]], is for this **same reason**.