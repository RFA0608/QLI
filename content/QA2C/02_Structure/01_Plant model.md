---
title: 01. Plant model
---
# System model
## Figure. model

![[QA2C/Figure/Structure_plant_model.png]]
## Description
By referencing as shown in the figure above, the system can be linearized at the **equilibrium point** (where **pitch = 0** and **yaw = 0**) to obtain the following  continuous **ABCD equivalent model**.

$$
A = \pmatrix{0 & 0 & 1 & 0 \\ 0 & 0 & 0 & 1 \\ M_g/J_p & 0 & D_p/J_p & 0 \\ 0 & 0 & 0 & D_y/J_y}
$$
$$
B = \pmatrix{0 & 0 \\ 0 & 0 \\ K_{pp}/J_p & K_{py}/J_p \\ K_{yp}/J_y & K_{yy}/J_y }
$$
$$
C = \pmatrix{1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0} 
$$
$$
D = \pmatrix{0 & 0 \\ 0 & 0}
$$

where `Jp = 0.0211`, `Jy = 0.0221`, `Dp = 0.0053`, `Dy = 0.0062`, `Mg = 0.0153`, `Kpp =  0.0011`, `Kyy =  0.0047`, `Kpy =  0.0021`, `Kyp = -0.0027`.

In conclusion, the system is as follows:

$$
\dot{X} = AX+Bu
$$
$$ 
y=CX + Du
$$


[Dynamics](https://www.quanser.com/wp-content/uploads/2017/01/Quanser_AERO_Courseware_Sample_for_MATLAB_Users.pdf) (NOT Aero 2 just Aero for example)
