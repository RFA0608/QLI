---
title: 01. Quanser Qube Servo 3 Control
---
# QQS3C - Quanser Qube Servo 3 Control
  
## Intro
This page introduces the implementation corresponding to [QQS3C-github](https://github.com/RFA0608/QQS3C). QQS3C is a code set for controlling Quanser's Qube Servo 3 pendulum model. There is a Python library for the Qube Servo 3 model provided by Quanser by default, but it is guaranteed to work only on Windows, and the libraries openFHE, Microsoft SEAL, and lattigo, which are used in Encrypted control, which is the ultimate goal of the library, use the same language but do not work on Windows, or to solve the situation where the language used is different, we implemented communication with codes in other languages ​​and other OS systems using TCP/IP of data. This page includes an introduction to basic control design techniques and scheduling for encrypted control, along with communication over TCP/IP.

## Contents
The content provided on this page is as follows:

1. Installation dependency
2. Scheduling communication between plants and controllers for encrypted control
3. Control-theoretical design techniques for encrypted control
4. Description of the implemented code for each language

A part to see how to install or operate the code before using it is included in the introduction in the side bar.

> [!INFO] Dependency installation
>  * [[02_Installation guide|Installation guide]]

Scheduling communication between plants and controllers for encrypted control is explained in "System structure" through "Transaction scheduling" in the "Structure" section of the sidebar.

>[!INFO] Scheduling communication between plants and controllers for encrypted control
> * [[01_System structure|System structure]]
> * [[02_Plant model|Plant model]]
> * [[03_Controller model|Controller model]]
> * [[04_Transaction scheduling|Transaction scheduling]]

Control-theoretical design techniques for encrypted control is explained in "Encryption method" and "Encrypted closed loop" in "Structure" of the sidebar of that page.

> [!INFO] Control-theoretical design techniques for encrypted control
> * [[05_Encryption method|Encryption method]]
> * [[06_Encrypted closed loop|Encrypted closed loop]]

Description of the implemented code for each language is introduced on each corresponding language page under "Implementation" in the sidebar.

>[!INFO] Description of the implemented code for each language
> * [[01_Python plant and transaction|Python plant and transaction]]
> * [[02_Python based controller description|Python based controller description]]
> * [[03_C++ based controller description|C++ based controller description]]
> * [[04_Go based controller description|Go based controller description]]

Variables with dependencies between the code can be checked below.
- [[05_Code dependency|Code dependency]]

If you wan to modify controller code to your description, then follows below.
 - [[06_Controller modification|Controller modification]]