---
title: 01. Python plant and transaction
---
# Plant
To operate the Quanser Qube Servo 3, I will first introduce the Python-based Quanser code that utilizes the provided Python API. Afterward, I will present a simple test simulation to ensure proper hardware operation.
## Hardware
The code written can be found at the link below:
[QQS3C/interface/plant/py/hardware](https://github.com/RFA0608/QQS3C/tree/main/interface/plant/py/hardware)
### Import
``` Python
	# get quanser interface lib and tcp_protocol description (vscode[debuger] launched at root directory)
	import sys
	sys.path.append(r"C:\Quanser\0_libraries\python")
	from pal.products.qube import QubeServo2, QubeServo3
	from pal.utilities.math import SignalGenerator, ddt_filter
	from pal.utilities.scope import Scope
	
	sys.path.append(r"./py")
	import tcp_protocol_server as tcs
	
	# init tcp host and port
	HOST = '0.0.0.0'
	PORT = 9999
	
	# get other tools
	from threading import Thread
	import signal
	import time
	import math
	import numpy as np
```
The code above is the top part of the given code.
``` Python
	sys.path.append(r"C:\Quanser\0_libraries\python")
	from pal.products.qube import QubeServo2, QubeServo3
	from pal.utilities.math import SignalGenerator, ddt_filter
	from pal.utilities.scope import Scope
```
The code above is a library provided by Quanser for communicating with the hardware. This can be verified on Quanser's page: [Quanser_Academic_Resources](https://github.com/quanser/Quanser_Academic_Resources).

To import the saved library, the following line is included:
``` Python
	sys.path.append(r"C:\Quanser\0_libraries\python")
```
.

``` Python
	sys.path.append(r"./py")
	import tcp_protocol_server as tcs
	
	# init tcp host and port
	HOST = '0.0.0.0'
	PORT = 9999
```
The code above is the section for importing the file where the TCP communication functions provided by QQS3C are implemented. The important point to note here is that **you must run the file or the debugger (like vscode) from the QQS3C directory, which is the folder created after you `git clone` QQS3C**. This is necessary so that the path in the code below works correctly:
``` Python
	sys.path.append(r"./py")
```
This point is noted on the [QQS3C-github](https://github.com/RFA0608/QQS3C) page, where you can confirm it.

The rest of the code consists of libraries provided by pip in the Python environment being used, which can similarly be verified on the GitHub page mentioned above.
### Main
Next is the main code that actually operates
```Python
	# interface setting #
    # ------------------------------------------------ #
    # qube version, using hardware, pendulum
    qubeversion = 3
    hardware = 1
    pendulum = 1

    # frequency of system holder and sampler
    frequency = 40 # hz

    # for scope sampling rate
    countMax = frequency / 50
    count = 0

    # class initialization
    QubeClass = QubeServo3

    # swing-up standing gate
    stand_run = False
    er = 0.02

    # describe #
    # ------------------------------------------------ #
    # instance of hardware model 
    with QubeClass(hardware=hardware, pendulum=pendulum, frequency=frequency) as myQube:
        # instance of tcp layer
        with tcs.tcp_server(HOST, PORT) as tcsp:
            startTime = 0
            timeStamp = 0
            def elapsed_time():
                return time.time() - startTime
            startTime = time.time()

            while timeStamp < simulationTime and not KILL_THREAD:
                if not stand_run:
                    # read sensor information
                    myQube.read_outputs()

                    # calc output
                    theta = myQube.motorPosition * -1
                    alpha_f =  myQube.pendulumPosition
                    alpha = np.mod(alpha_f, 2*np.pi) - np.pi
                    alpha_deg = alpha * 180 / np.pi

                    if abs(alpha) < er and abs(theta) < er:
                        stand_run = True
                    
                    voltage = 0
                    # write commands
                    myQube.write_voltage(voltage)

                    print(f"control start: {stand_run}")
                else:
                    # running signal send for controller
                    tcsp.send("run")
    
                    # read sensor information
                    myQube.read_outputs()
    
                    # calc output
                    theta = myQube.motorPosition * -1
                    alpha_f =  myQube.pendulumPosition
                    alpha = np.mod(alpha_f, 2*np.pi) - np.pi
                    alpha_deg = alpha * 180 / np.pi
    
                    # send plant output
                    tcsp.send(-theta)
                    tcsp.send(-alpha)
    
                    # get control input
                    _, u = tcsp.recv()
    
                    # running range set
                    if abs(alpha_deg) < 15:
                        voltage = u
                    else:
                        voltage = 0
    
                    # write commands
                    myQube.write_voltage(voltage)

                # plot to scopes
                count += 1
                if count >= countMax:
                    scopePendulum.sample(timeStamp, [-alpha])
                    scopeBase.sample(timeStamp, [-theta])
                    scopeVoltage.sample(timeStamp,[voltage])
                    count = 0

                timeStamp = elapsed_time()

            tcsp.send("end")

```
First, I will introduce the top part.
```Python
	# interface setting #
    # ------------------------------------------------ #
    # qube version, using hardware, pendulum
    qubeversion = 3
    hardware = 1
    pendulum = 1

    # frequency of system holder and sampler
    frequency = 40 # hz

    # for scope sampling rate
    countMax = frequency / 50
    count = 0

    # class initialization
    QubeClass = QubeServo3

    # swing-up standing gate
    stand_run = False
    er = 0.02
```
This section has three categories.
```Python
	# qube version, using hardware, pendulum
    qubeversion = 3
    hardware = 1
    pendulum = 1
    
    # class initialization
    QubeClass = QubeServo3
```
This code section pertains to the options for instantiating the `QubeServo3` class, which is implemented in the library provided by Quanser.
```Python
	# frequency of system holder and sampler
    frequency = 40 # hz

    # for scope sampling rate
    countMax = frequency / 50
    count = 0
```
This part of the code sets the sampling rate for the Quanser plant.
```Python
	# swing-up standing gate
    stand_run = False
    er = 0.02
```
Finally, the section above exists to prevent an excessively large $u$ from being generated by $y$ being passed to the controller before the Quanser Qube Servo 3's pendulum is swung up. It is implemented so that $y$ is sent to the controller and $u$ is received **only when** the pendulum is positioned within an error margin of 0.02 from the equilibrium point we defined.

Afterward, the Quanser Qube Servo 3 model is instantiated using:
```Python
	# instance of hardware model 
    with QubeClass(hardware=hardware, pendulum=pendulum, frequency=frequency) as myQube:
```
.

And,
```Python
	# instance of tcp layer
        with tcs.tcp_server(HOST, PORT) as tcsp:
```
is used to run the server for communication with the controller located in another layer (WSL).

After skipping past some less important parts, you can see:
```Python
	# read sensor information
                    myQube.read_outputs()
```
This is the function that reads the outputs from the Quanser Qube Servo 3.

```Python
	# calc output
                    theta = myQube.motorPosition * -1
                    alpha_f =  myQube.pendulumPosition
                    alpha = np.mod(alpha_f, 2*np.pi) - np.pi
                    alpha_deg = alpha * 180 / np.pi

```
This is the part that converts the read data into radian (rad) or degree (deg) values. **Be aware that the equilibrium point is determined by the initial position of the Quanser Qube Servo 3's base and pendulum, so be careful to keep the pendulum stable and prevent it from swinging at the start.**

```Python
	if abs(alpha) < er and abs(theta) < er:
                        stand_run = True
                    
                    voltage = 0
                    # write commands
                    myQube.write_voltage(voltage)

                    print(f"control start: {stand_run}")
```
The section above is code that prevents operation when the angle is too far from the equilibrium point we set. When the function is stuck in this part, **'control start: False'** will be displayed in the terminal. Afterward, if you manually move the pendulum upright to a position near the equilibrium point, it will display **'control start: True'** and follow the loop shown in the code below.
```Python
	# running signal send for controller
                    tcsp.send("run")
    
                    # read sensor information
                    myQube.read_outputs()
    
                    # calc output
                    theta = myQube.motorPosition * -1
                    alpha_f =  myQube.pendulumPosition
                    alpha = np.mod(alpha_f, 2*np.pi) - np.pi
                    alpha_deg = alpha * 180 / np.pi
    
                    # send plant output
                    tcsp.send(-theta)
                    tcsp.send(-alpha)
    
                    # get control input
                    _, u = tcsp.recv()
    
                    # running range set
                    if abs(alpha_deg) < 15:
                        voltage = u
                    else:
                        voltage = 0
    
                    # write commands
                    myQube.write_voltage(voltage)
```
In the code above,
```Python
	tcsp.send("run")
```
sends a signal to the controller to operate, and when the simulation time is over,
```Python
	tcsp.send("end")
```
is sent to automatically turn off the controller's code.
```Python
	 # read sensor information
                    myQube.read_outputs()
    
                    # calc output
                    theta = myQube.motorPosition * -1
                    alpha_f =  myQube.pendulumPosition
                    alpha = np.mod(alpha_f, 2*np.pi) - np.pi
                    alpha_deg = alpha * 180 / np.pi
    
                    # send plant output
                    tcsp.send(-theta)
                    tcsp.send(-alpha)
```
The section above receives the output from the Quanser Qube Servo 3 model, processes it for use (by appropriately converting it to radians or applying coordinate transformation), and then sends the values to the controller.
```Python
	# get control input
                    _, u = tcsp.recv()
    
                    # running range set
                    if abs(alpha_deg) < 15:
                        voltage = u
                    else:
                        voltage = 0
    
                    # write commands
                    myQube.write_voltage(voltage)
```
The code above is implemented to receive the calculated $u$ from the controller and input it into the actual model.
## Simulation
### Model
Model is the file that implements the class for the operational model that will run in the simulation's plant.
```Python
	import numpy as np
	import control as ct
```
The required external libraries to operate this model are as listed above. This model class internally stores the discretized matrices of the equivalent ABCD state-space model using the 'zoh' (Zero-Order Hold) method.
```Python
	# system(state) matrix(state space linearlization - discrete model)
    A = np.zeros((4,4), dtype=float)
    B = np.zeros((4,1), dtype=float)
    C = np.zeros((2,4), dtype=float)
    D = np.zeros((2,1), dtype=float)

    # state and output
    xp = np.zeros((4,1), dtype=float)
    y = np.zeros((2,1), dtype=float)
```
The code above stores the values of the discretized ABCD matrices and additionally saves the plant's internal state values and outputs.
```Python
	def __init__(self, ts):
        # write your continous time linear model
        A = np.array([[0, 0, 1, 0],
                      [0, 0, 0, 1],
                      [0, 149.275096865093, -0.0104427822555581, 0],
                      [0, 261.609107366662, -0.0103213545549121, 0]], dtype=float)
        B = np.array([[0],
                      [0],
                      [49.7275345502766],
                      [49.1493074043432]], dtype=float)
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

        # ------------------------------------------------ #

        # set initial point of state
        x_init = np.array([[0],
                           [0],
                           [0],
                           [0]], dtype=float)
        
        # save initial point
        self.xp = x_init
```
This section is the constructor for the class. You can see that the continuous-time ABCD matrices are already hardcoded, and it simply discretizes and stores them according to the sampling time you provide.
```Python
	def set_init(self, x_init):
        self.xp = x_init

    def state_update(self, u):
        # update state on temp variable
        xp_next = self.A @ self.xp + self.B @ u
        
        # save state
        self.xp = xp_next

    def get_output(self):
        return self.C @ self.xp
```
The functions inside the class are as shown above, consisting of a function to set the initial values, a state update function, and an output function.
### Plant
[[01_Python plant and transaction#Simulation#Plant|Python plant and transaction>Simulation>Plant]] is the code implemented in the same sequence as deploying the simulation model (implemented in [[01_Python plant and transaction#Simulation#Model|Python plant and trasaction>Simulation>Model]]) to the actual Quanser hardware.
#### Import
```Python
	# get tcp_protocol description (vscode[debuger] launched at root directory)
	import sys
	sys.path.append(r"./py")
	import tcp_protocol_server as tcs
	
	# init tcp host and port
	HOST = '0.0.0.0'
	PORT = 9999
	
	# get model description
	import model
	
	# get other tools
	import numpy as np
	import matplotlib.pyplot as plt
	import time
```
Similar to the [[01_Python plant and transaction#Hardware|Python plant and transaction>Hardware]] section, you must also pay close attention to the **execution location** for this part. It is not significantly different from simply replacing the Quanser model with the simulation-driven model.

#### Main
```Python
	# set simulation(this section have to set same with controller)
    sampling_time = 0.02
    max_time = 10
    max_iter = int(max_time / sampling_time)

    # get model from model description file
    plant = model.rotpen(sampling_time)

    # set state initial value
    plant.set_init(np.array([[-0.3],
                             [-0.2],
                             [0],
                             [0]], dtype=float))

    # input/output initialization
    y = np.array([[0],
                  [0]], dtype=float)
    u = np.array([[0]], dtype=float)

    # state and outupt memory initialization for history plot
    time_stack = np.zeros((max_iter, 1))
    y_his = np.zeros((max_iter, 2))

    with tcs.tcp_server(HOST, PORT) as tcsp:
        for i in range(max_iter):
            # start time set for measurment
            start_clock = time.perf_counter_ns()

            # running signal send for controller
            tcsp.send("run")

            # get plant output, send data, and save data
            y = plant.get_output()
            tcsp.send(y[0, 0])
            tcsp.send(y[1, 0])
            time_stack[i, 0] = i * sampling_time
            y_his[i, 0] = y[0, 0]
            y_his[i, 1] = y[1, 0]

            # get control input from controller
            _, uk = tcsp.recv()
            u[0, 0] = uk

            # plant state update
            plant.state_update(u)

            # end time set for measurment and calculation duration(transform unit to ms)
            end_clock = time.perf_counter_ns()
            duration = (end_clock - start_clock) / 1000000000

            # sleep to satisfy sampling time(if you want to save your time, comment out the sentence below)
            # time.sleep(sampling_time - duration)
        
        # endding signal send for controller
        tcsp.send("end")
```
The code above is the code that will actually be executed.
```Python
	# set simulation(this section have to set same with controller)
    sampling_time = 0.02
    max_time = 10
    max_iter = int(max_time / sampling_time)

    # get model from model description file
    plant = model.rotpen(sampling_time)

    # set state initial value
    plant.set_init(np.array([[-0.3],
                             [-0.2],
                             [0],
                             [0]], dtype=float))
```
The section above sets the sampling time and the simulation time, instantiates the model class, and assigns its initial values.
```Python
	# state and outupt memory initialization for history plot
    time_stack = np.zeros((max_iter, 1))
    y_his = np.zeros((max_iter, 2))
```
The section above contains variables for storing the simulation results for viewing.
```Python
	with tcs.tcp_server(HOST, PORT) as tcsp:
        for i in range(max_iter):
            # start time set for measurment
            start_clock = time.perf_counter_ns()

            # running signal send for controller
            tcsp.send("run")

            # get plant output, send data, and save data
            y = plant.get_output()
            tcsp.send(y[0, 0])
            tcsp.send(y[1, 0])
            time_stack[i, 0] = i * sampling_time
            y_his[i, 0] = y[0, 0]
            y_his[i, 1] = y[1, 0]

            # get control input from controller
            _, uk = tcsp.recv()
            u[0, 0] = uk

            # plant state update
            plant.state_update(u)

            # end time set for measurment and calculation duration(transform unit to ms)
            end_clock = time.perf_counter_ns()
            duration = (end_clock - start_clock) / 1000000000

            # sleep to satisfy sampling time(if you want to save your time, comment out the sentence below)
            # time.sleep(sampling_time - duration)
        
        # endding signal send for controller
        tcsp.send("end")
```
This part is omitted as its code progression is the same as the Quanser hardware section.
```Python
	# draw plot of plant output y value
        fig, axes = plt.subplots(2, 1)
        axes[0].plot(time_stack, y_his[:,0])
        axes[0].set_title('position')
        axes[0].grid(True)
        axes[1].plot(time_stack, y_his[:,1])
        axes[1].set_title('angle')
        axes[1].grid(True)
        fig.suptitle('plant output')
        plt.tight_layout()
        plt.savefig('./interface/plant/py/simulation/result/plant output as sim.png')
```
The last part is the section for saving the simulation results.