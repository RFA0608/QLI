---
title: 02. Installation guide
---
# Installation guide
This section covers the things that you need to install before you can use the library. The installation method is divided into two types, depending on how you use the library.

1. Using Windows enviroment only
2. Using both Windows and WSL

The reason for this division is that the library was originally intended to work in a combination of a Windows environment where Qanser's plant code works and a Linux environment where the cryptographic library is easy to apply. However, there was an interactive phenomenon in the Windows environment alone, so we additionally supported it on Windows only. 

However, since both Windows only and Windows and WSL environments use TCP, there are installation methods. (The reason for using TCP is to communicate data in different languages.) Below, I have written down a detailed explanation of it.
# Settings for operation
There exist two way to use this library. One is using both Windows and WSL environment, The other is using only Windows environment.

If you want to drive the code using Windows only, refer to 
1. [[03_Using Windows only|Using Windows only]].

On the other hand, if you want to use Windows and WSL environments, see this
1. [[04_Using both Windows and WSL|Using both Windows and WSL]].

You can use release file of [Automatic Setting on Windows](https://github.com/RFA0608/QQS3C/releases/tag/dependency_install). Just simply double click 'install_all.bat'. This automatically installs in all Windows environments. (**Please read github text**)
# Virtual ENV
All about this structure, need to launch venv enviroment. As you can see in [[03_Using Windows only|Using Windows only]] or [[04_Using both Windows and WSL|Using both Windows and WSL]] make venv with command `python -m venv venv`. After making, you can find folder venv and can see file "Activate.ps1" of "activate.bat". So if you set on venv, then always can run the code on actiavted enviroment.

There are three case here:
1. After setting, there is (venv) word on your command.
In this situation, venv is already on, so there is no need to worry about it.

2. Cannot find (venv) in cmd
You must run "Scripts" > "activate.bat" inside the folder named venv. (If you use Powershell, file name is "Activate.ps1") To activate this, simply type `./venv/Scripts/activate.bat` in a CMD window where the venv folder is visible (venv should be visible when you type `dir` in CMD or `ls` in Powershell).

3. Cannot run code on VScode
You can find python version on rigth bottom of VScode (If you do not check this, just one time run the code. After that appear version part). Click them and click "Browse..." part. Navigate to the folder containing "venv" using the Explorer browser above, and click 'Python.exe' in the "venv" > "Scripts" folder.
