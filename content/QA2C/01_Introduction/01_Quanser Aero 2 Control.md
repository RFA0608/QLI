---
title: 01. Quanser Aero 2 Control
---

# QA2C - Quanser Aero 2 Control
## Intro
This page introduces the implementation corresponding to [QA2C-github](https://github.com/CDSL-EncryptedControl/QA2C). QA2C is a code set for controlling Quanser's Aero 2 model. There is a Python library for the Aero 2 model provided by Quanser by default, but it is guaranteed to work only on Windows, and the libraries openFHE, Microsoft SEAL which are used in Encrypted control, which is the ultimate goal of the library, use the same language but do not work on Windows, or to solve the situation where the language used is different, we implemented communication with codes in other languages ​​and other OS systems using TCP/IP of data. This page includes an introduction to basic control design techniques and scheduling for encrypted control, along with communication over TCP/IP.
## Contents
The content provided on this page is as follows:

1. Installation dependency (substituted by QQS3C instruction)
2. Scheduling communication between plants and controllers for encrypted control (substituted by QQS3C instruction)
3. Control-theoretical design techniques for encrypted control

**(Most concepts and structures are shared with [[01_Quanser Qube Servo 3 Control|QQS3C]])**

A part to see how to install or operate the code before using it is included in the introduction in the side bar.

> [!INFO] Dependency installation
>  * [[02_Installation guide|Installation guide]]

Scheduling communication between plants and controllers for encrypted control is explained in "System structure" through "Transaction scheduling" in the "Structure" section of the sidebar.

>[!INFO] Scheduling communication between plants and controllers for encrypted control
> * [[01_System structure|System structure]]
> * [[01_Plant model|Plant model]]
> * [[02_Controller model|Controller model]]
> * [[04_Transaction scheduling|Transaction scheduling]]

Control-theoretical design techniques for encrypted control is explained in "Encryption method" and "Encrypted closed loop" in "Structure" of the sidebar of that page.

> [!INFO] Control-theoretical design techniques for encrypted control
> * [[05_Encryption method|Encryption method]]
> * [[06_Encrypted closed loop|Encrypted closed loop]]