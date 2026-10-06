---
title: 02. Plant model
---
# System model
## Figure. model

![[QQS3C/Figure/Structure_plant_model.png]]
## Description
By referencing the two links below, and as shown in the figure above, the system can be linearized at the **equilibrium point** (where **alpha = 0** and **theta = 0**) to obtain the following  continuous **ABCD equivalent model**.

$$
A = \pmatrix{0 & 0 & 1 & 0 \\ 0 & 0 & 0 & 1 \\ 0 & 149.275096865093 & -0.0130994260613721 & 0 \\ 0 & 261.609107366662 & -0.0129471071536817 & 0}
$$
$$
B = \pmatrix{0 \\ 0 \\ 55.6948386963098 \\ 55.0472242928643}
$$
$$
C = \pmatrix{1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0} 
$$
$$
D = \pmatrix{0 \\ 0}
$$

In conclusion, the system is as follows:

$$
\dot{X} = AX+Bu
$$
$$
y=CX + Du
$$


[Pendulum Equations](https://github.com/quanser/Quanser_Academic_Resources/blob/dev-windows/6_teaching/1_Controls/Qube_Servo_3/supplementary_material/Pendulum%20Equations.pdf)
[Quanser Qube Servo 3 Linear model](https://github.com/quanser/Quanser_Academic_Resources/blob/dev-windows/6_teaching/1_Controls/Qube_Servo_3/sp5_pendulum_modeling/2_state_space_modeling/hardware/matlab/lab_procedure_state_space_modeling.pdf)