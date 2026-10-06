---
title: 01. System structure
---
# Design for transaction between Windows and WSL

## Conventional struct
### Figure. closed loop 
![[Structure_closed_loop.png]]
### Description
The system components provided by the library constitute an **autonomous system block**, configured as shown in "Figure. closed loop." Since the **Plant** is built using Python on Windows, it is necessary to pass the Plant's input/output (I/O) data to the coding language and environment that constitutes the **Controller**. Please refer to the diagram below.

---
### Figure. hierarchy
![[Structure_hierarchy_plant.png]]
- $\uparrow$ Plant side class

![[Structure_hierarchy_controller.png]]
- $\uparrow$ Controller side class
## Our Design
### Figure. layer for transaction
![[Structure_hierarchy_system.png]]
### Description
Implement a transmission server within the **Plant's** **Windows Python environment**. This server will be responsible for sending the Plant's output, **y**, to the controller and receiving the control input, **u**, from the controller. Concurrently, configure a transmission client within the **Controller's** **WSL environment**. This client will receive **y** from the Plant and send **u** back to the Plant at **each sampling time**. The server and client are to be implemented within the Plant code and Controller code, respectively. The Controller code should be **conceptually separated** (modularized) to accommodate the potential for **encrypting y and u**. This transmission server and client will be implemented using **TCP/IP**.