---
title: 04. Using both Windows and WSL
---
# Using both Windows and WSL
## Windows environment
1. You need to download the code via git clone on PowerShell page.
   ``` powershell
     git clone "https://github.com/RFA0608/QQS3C.git"
   ```
2. Execute the following task in Windows PowerShell.
   * Navigate to the downloaded directory.
     ``` powershell
       cd QQS3C
     ```
   * Activate Python's virtual environment.
   (**⚠️All pip installations below must be installed in the applicable venv environment, and all operations of python are valid only in the applicable venv.**)
     ``` powershell
       python -m venv venv
       .\venv\Scripts\Activate.ps1
     ```
     or
     ``` powershell
       py -m venv venv
       .\venv\Scripts\Activate.ps1
     ```
     (If the above doesn't work, try the one below. Maybe you can run that script on CMD just change word of 'Activate.ps1' to 'activate.bat')
     
     If the command doesn't work, try again by following these steps:
       1. Launch PowerShell as administrator.
       2. Set execution policy
          ``` powershell
            Set-ExecutionPolicy RemoteSigned
          ```
       3. Turn off the administrator PowerShell, open a standard (non-administrator) PowerShell, and try the command again.
   * Download all required packages using pip.
     ``` powershell
       pip install numpy matplotlib control openfhe PyQt6 pyqtgraph
     ```
3. Link interpreter
   * First, move directory to root folder.
     ``` powershell
       cd QQS3C
     ```
   * Find absolute directory address(path) and memorize this.
     ``` powershell
       pwd
     ```
   * Change the address(path) above to <YOUR_PATH> below.
     ``` powershell
       pip install -e "<YOUR_PATH>/communication/py"
     ```
5. You need to check the hyper-v ip for TCP/IP communication between the Windows and WSL.
   ``` powershell
     ipconfig
   ```
   Save IPv4 address(like 172.xxx.xxx.xxx) of vEthernet (WSL (Hyper-V...)).

**⚠️CRITICAL: Essential Enviroment Setup**:

**Installation Quanser SDK**
If you want to use Ouanser Interactive Labs(QLab) or Quanser Hardwares, follows below.
1. Enter the url [portal_quanser](https://portal.quanser.com/Downloads), find 'these instructions' in "For Python users" section, and find 'Get Started' in "Design Philosophy" section.
2. (**Only QLab/Not need hardware users**)Download and install Quanser Interactive Labs to click 'Windows' in "Attention" section.
3. Download and install SDK to click 'Download Quanser SDK for Windows' in "Attention Windows" section.
4. If you do not touch any option during installing, you can find 'quanser_api' word in "Program Files/Quanser/Quanser SDK/python" path. Just check this file. (If you use real hardware and pre-installed QUARC, then path is "Program Files/Quanser/QUARC/python")
5. Make sure venv is on, write (**VERY IMPORTANT: Anything on Quanser requires this, so it must be installed**)
	``` powershell
	 python -m pip install --upgrade --find-links "C:\Program Files\Quanser\Quanser SDK\python" "C:\Program Files\Quanser\Quanser SDK\python\quanser_api-2025.11.1-py2.py3-none-any.whl"
	```
	or using real hardware(with QUARC)
	``` powershell
	 python -m pip install --upgrade --find-links "C:\Program Files\Quanser\QUARC\python" "C:\Program Files\Quanser\QUARC\python\quanser_api-2025.11.1-py2.py3-none-any.whl"
	```
  on the terminal and connect the SDK (this path can find in step 4, Please change the month and day of the year to your own at the end of the command).

**Installation Quanser library**
All python implementation need to apply on venv environment. (you can check (venv) word on left of your CLI area)
1. Enter the url [github-quanser](https://github.com/quanser/Quanser_Academic_Resources), download library(whole things) and unzip proper path(like document).
2. For easy to use Quanser's python library, you can make file `pyproject.toml` on 0_libraries->python in Quanser Academic Resources, which is before we downloaded, for pip installation.
3. Fill a text on `pyproject.toml` like below (Just copy and paste)
```
  [build-system]
  requires = ["setuptools", "wheel"]
  build-backend = "setuptools.build_meta"
  
  [project]
  name = "quanser-pal"
  version = "0.1.0"
  description = "Quanser PAL lib"
  
  [tool.setuptools]
  packages = ["pal"]
```
4. Find absolute address(path) location of `pyproject.toml`
5. Be careful that your environment is on venv, put the command below (change <YOUR_PATH> to address(path) what is the path name of python folder that contain `pyproject.toml`)
  ```
    pip install -e <YOUR_PATH>
  ```
---
### Option
OPTION 1: If you want to use QUARC-C based plant code (more suitable real-time interaction than python), need more setting on Visual Studio(VS 2022)
1. Install Visual Studio and QLab(according to the above content).
2. Make a new project.
3. Put the file plant.cpp, which is located in "interface/plant/cpp/hardware", to source file section.
4. Put the file tcp_protocol_server_windows.h, which is located in "communication/cpp", to header file section.
5. Enter project configuration, that is located project->Properties, Change C++ Language Standard C++17 (maybe it was C++14)
6. Find the address(path) of "Quanser SDK/include" and paste on C/C++->Additional Include Directories section. (maybe there's a Quanser SDK in the Quanser folder in Program Files, or there's a QUARC in it)
7. Find the address(path) of "Quanser SDK/lib/win64" and paste on Linker->General->Additional Library Directories section.
8. Move to Linker->Input->Additional Dependencies section, put 'hil.lib', 'quanser_runtime.lib', 'quanser_common.lib' in their.
## WSL environment
Assuming you have Python and Go installed.
If not, you should refer to the above version and install it.

1. Microsoft SEAL installation
   * See the "SEAL installation method.txt" file on the main page.
  
2. Essential Python package installation
   * First, download the relevant code via git clone on WSL bash page.
     ``` bash
       git clone "https://github.com/RFA0608/QQS3C.git"
     ```
   * Navigate to the downloaded directory.
     ``` bash
       cd QQS3C
     ```
   * Activate Python's virtual environment.
     ``` bash
       python3 -m venv venv
       source ./venv/bin/activate
     ```
   * Download all required packages using pip.
     ``` bash
       pip install numpy matplotlib control openfhe
     ```
3. Link complier and interpreter of communication tools
   * First, move directory to root folder.
     ``` bash
       cd QQS3C
     ```
   * Find absolute directory address(path) and memorize this.
     ``` bash
       pwd
     ```
   * Change the address(path) above to <YOUR_PATH> below. (C++ Link)
     ``` bash
       export CPATH=$CPATH:<YOUR_PATH>/communication/cpp
     ```
   * And with same address(path), change to below.
     ``` bash
       pip install -e "<YOUR_PATH>/communication/py"
     ```
4. Lattigo installation
   * This is automatically handled by go mod tidy, so no preparation is required.
## Ready to operate
Execution methods vary depending on the environment.
### Windows environment
1. Go to the previously downloaded QQS3C folder location and run the debugger (vscode) to write below.
    ``` powershell
       code .
    ```
2. Here, it is divided depending on whether simulation is performed or actual hardware is operated.
   * Simulation
     1. Find plant description code set which are located in "interface/plant/py/simulation" folder on debugger (vscode).
     2. Select the controller file named "plant.py"
     3. Get ready to press F5 button.
   * Quanser Interactive Labs
     1. Lanch Quanser Interactive Labs before we installed, login, and select "Qube 3 -Pendulum".
     2. Find plant description code set which are located in "interface/plant/py/hardware" folder on debugger (vscode).
     3. Select the controller file named "plant.py" or "plant_with_swing_up.py".
     4. Find variable 'hardware' in "def control_loop()" and change value to 0.
     5. Get ready to press F5 button.
   * Hardware
     1. Find plant description code set which are located in "interface/plant/py/hardware" folder on debugger (vscode).
     2. Select the controller file named "plant.py" or "plant_with_swing_up.py".
     3. Get ready to press F5 button.
3. Additionaly, if you use only Windows environment.
   * Controller
     1. Choose controller code (Except for using OpenFHE-python).
     2. And ready to run.

This completes the plant's preparation for operation.

### WSL environment
1. Go to the previously downloaded QQS3C folder location and run the debugger (vscode) to write below.
   ``` bash
     code .
   ```
2. Here, each file provided in three languages (py, cpp, go) has a different execution method.
   * Python
     1. Find controller description code set which are located in "interface/controller/py" folder on debugger (vscode).
     2. Select the controller file you want to run.
     3. In that file, change 'localhost' in HOST variable to the vEthernet ip you saved earlier.
     4. Get ready to press F5 button.
   * C++
     1. Find controller description code set which are located in "interface/controller/cpp/arx_model" folder on debugger (vscode).
     2. Select the controller file you want to run.
     3. In that file, change 'localhost' in HOST variable to the vEthernet ip you saved earlier.
     4. In the bash window, move directory to "interface/controller/cpp/arx_model".
        ``` bash
          cd interface/controller/cpp/arx_model
        ```
     5. Create a new make file using cmake.
        ``` bash
          cmake .
        ```
     6. Create an executable binary file using the make file.
        ``` bash
          make
        ```
     7. If you see a file called "ctrl_arx_enc" then you are done and ready to write the following in the bash window and press enter
        ``` bash
          ./ctrl_arx_enc
        ```
   * Go
     1. Find controller description code set which are located in "interface/controller/go" folder on debugger (vscode).
     2. Select the controller file you want to run.
     3. In that file, change 'localhost' in HOST variable to the vEthernet ip you saved earlier.
     4. In the bash window, move directory to "interface/controller/go/integer_matrix".
        ``` bash
          cd interface/controller/go/integer_matrix
        ```
     5. Set GOPATH to the current directory.
        ``` bash
          pwd
        ```
        Copy the result and paste it into <YOUR_PATH> below.
        ``` bash
          export GOPATH=<YOUR_PATH>
        ```
     6. At that location, write something like the following and be ready to press enter.
        ``` bash
          go run .
        ```
This completes the controller's preparation for operation.
## Operation
Proceed in the following order.
1. Press F5, which is waiting for Windows environment.
2. Press F5 or enter to launch the controller that was waiting in WSL.

If you ran a simulation, a graph of the output will appear in the file "plant output as sim.png" in "interface/plant/py/simulation/result" folder.

If you ran a Quanser Interactive Labs, you can see movement on QLabs.

If you are running real hardware, there is two side of launch, first is manually raise the pendulum(use "plant.py" code), second automatically swing up the pendulum(use "plant_with_swing_up.py"), so that control start is True while looking at the output of the debugger (vscode) running in the Windows environment.
Note: For successful control initialization, both the pendulum and the base must be positioned near the equilibrium point.
