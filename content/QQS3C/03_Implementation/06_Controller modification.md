---
title: 06. Controller modification
---
# Possibility of controller change
QQS3C was written with the goal of easy controller conversion from the code design stage. If you wish to use an encryption method other than the Homomorphic Encryption (RLWE family, RGSW) primarily used in this code, or if you want to use a controller type other than observer-based or its variations, you can easily make changes by modifying a few lines of code and inserting your own controller code. This page explains how you can apply your code to QQS3C.

# Code implementation concept
## Control code
The controller code for the QQS3C was written in the following order. For reference code, you can refer to [QQS3C/interface/controller/py/observer_form/ctrl_obs.py](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/py/observer_form/ctrl_obs.py).

1. Import external package
2. Initializer that creates sampling time variable and controller model objects
3. Variable initializer to store input and output
4. Declaration of (code blocks) for TCP/IP communication
5. Handshake with the plant regarding execution
6. The part that retrieves the plant's output ($y$).
7. The part that sends input (u) to the plant
8. The part that updates the controller's state variables

Refer to the following for the order of the code mentioned above.
![[Implementation_controller_code.png]]
## Model code
At this time, the observer model, what decribe on Control code ,defined in No.2 can be found at [QQS3C/interface/controller/py/observer_form/model.py](https://github.com/RFA0608/QQS3C/blob/main/interface/controller/py/observer_form/model.py). The structure of the code is as follows.

1. Variable declaration section
2. Continuous-time linearization model (based on pendulum pointing upward)
3. System variables for discrete-time transformation
4. Storage of discrete-time model state-matrix
5. Generation of state feedback gains for control input generation
6. Generation of bbserver gains for observers
7. Storage of generated gains
8. Configuration of controller state space model
9. Declaration of variables corresponding to state variables
10. Declaration of state variable update rules
11. Declaration of control input generation rules

Refer to the following for the order of the code mentioned above.

![[Implementation_model_code_sec1.png]]
![[Implementation_model_code_sec2.png]]
![[Implementation_model_code_sec3.png]]
# Editable part
Based on the two photos introduced in [[06_Controller modification#Code Implementation Concept|Controller modification>Code implementation concept]], the changes to the controller are as follows. If you do not wish to use this model and want to configure your own controller as is, design using No.2 of [[06_Controller modification#Model Code|Controller modification>Model code]], which corresponds to the continuous-time model, and replace the objects for the models used in No.2 and No.8 in [[06_Controller modification#Control Code|Controller modification>Control code]] with your own. In other words, when applying encrypted control, you simply need to remove the code related to the models used in No.2 and No.8 from the base code [[06_Controller modification#Control Code|Controller modification>Control code]] and insert your own code in that section. Additionally, you can refer to [QQS3C/interface/controller/cpp/arx_model](https://github.com/RFA0608/QQS3C/tree/main/interface/controller/cpp/arx_model) for code that implements encrypted control using these rules. (You can check the consistency of the structure.)

## Control code
If you want to create and implement the entire controller yourself, you can simply replace the following sections with your own code, in order, based on the [[06_Controller modification#Control Code|Controller modification>Control code]] guidelines:
1. No.2 (Model declaration)
2. No.8 (Model state update and control input generation)
## Model code
On the other hand, if you only want to tune specific values such as gains, you can just tune the following sections, in order, based on the [[06_Controller modification#Model Code|Controller modification>Model code]] guidelines:
1. No.5 (Input law gain)
2. No.6 (Observer gain)
3. No.9 (Initial value settings)