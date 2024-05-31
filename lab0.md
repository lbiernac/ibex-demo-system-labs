# Lab 0: Installing development software 

In this lab, we will:

- Install prerequisites for setting up your development environment on Windows, including the following:
* Visual Studio Code
* Docker
* Windows Subsystem for Linux (WSL) with Ubuntu 20.04

## Install Docker Desktop
Install Docker Desktop on Windows by following this tutorial: [https://docs.docker.com/desktop/install/windows-install/](https://docs.docker.com/desktop/install/windows-install/)

1. Open Window's _Task Manager_. Navigate to _Performance/CPU_ and ensure that "Virtualization" is listed as enabled. If virtualization is disabled, you will need to enable it in your BIOS settings.
2. Download and run the installer `Docker Desktop Installer.exe` from [Docker](https://docs.docker.com/desktop/install/windows-install/).
3. During the installation, under _Configuration_, ensure that the option "Install required Windows components for WSL 2" is selected. 
4. Complete the installation and open _Docker Desktop_. 
5. Select "Yes" to agree to the Subscription Service Agreement. Docker Desktop is free for small businesses (fewer than 250 employees AND less than $10 million in annual revenue), personal use, education, and non-commercial open-source projects.



## Configure Windows Subsystem for Linux (WSL) to use Ubuntu 20.04 (Required for Ibex)
Next, we will configure WSL 2, which allows us to build and run Linux-based containers on our Windows machine. For reference, the official documentation for WSL is found here: [https://learn.microsoft.com/en-us/windows/wsl/install](https://learn.microsoft.com/en-us/windows/wsl/install). Complete the steps below.

The Ibex docker container requires Ubuntu 20.04, which is an older version of Linux than the WSL default. Thus, the following instructions specify this exact version of Ubuntu during configuration. 

1. From the start menu, open _Turn Windows features on or off_ and make sure the options "Windows Subsystem for Linux" and "Virtual Machine Platform" are enabled. Then select "OK".
2. Open _Windows PowerShell_ as an administrator and run the command `wsl -l -v` to see the existing versions of WSL that are running. You can also run `wsl -l -o` to see all the available Linux distributions. 
3. Run `wsl --install -d Ubuntu-20.04` to install Ubuntu version 20.04. 
4. Once the installation is complete, the _Ubuntu_ terminal window will pop up. Follow the prompts to enter a username and password. 
5. Once this is completed, you have successfully created an Ubuntu environment on top of your Windows machine. Next, we will need to integrate the WSL Ubuntu operating system with Docker.
6. Open _Docker Desktop_. Under _Settings/Resources/WSL Integration_ enable "Ubuntu-20.24" under the heading "Enable integration with additional distros". 
7. Return to the Ubuntu terminal and type the command `docker` to ensure that the integration is complete. This command will print all available docker commands.
8. Make a new folder (called a _"directory"_) for these labs using the following command: `mkdir ibex-labs`. 
9. ***Finding your Linux files in File Explorer:*** You may want to navigate to your Linux files from File Explorer, rather than through the Ubuntu terminal. These files are not located in your Window's home directory (e.g., `C:\Users\USERNAME\ibex-labs`). Instead, they will be located at a path corresponding to the Linux environment's file system. This path will look something like: `\\wsl.localhost\Ubuntu-20.04\home\USERNAME\ibex-labs`.



## Install Visual Studio Code (VS Code)
We will use Visual Studio Code, also commonly referred to as VS Code, as an IDE for these labs. In future steps, we will configure VS Code to show both the Ubuntu terminal and docker container directly in the application window.  

1. Download and run the VS Code installer: [https://code.visualstudio.com/download](https://code.visualstudio.com/download). 
2. Complete the installation and open _Visual Studio Code_. Navigate to the folder corresponding to your Ubuntu environment.
3. Alternatively, from the Ubuntu terminal, you can open the project directory in VS Code using the command: `code ibex-labs/`. This command will boot a new version of VS Code with the corresponding folder. 
4. Open a terminal window inside of VS Code. The terminal may default to a "powershell" terminal, indicated by the text in the upper right-hand corner. Change the terminal to a Linux terminal by selecting the dropdown next to the +-sign and set the terminal to "Ubuntu-20.24 (WSL)". This gives you the same Ubuntu terminal you had previously.
5. This terminal is analogous to the separate Ubuntu terminal we were using earlier---it is just now integrated into the VS Code IDE. 
6. Install the "Docker" extension for VS Code by navigating to _Extensions_ (Ctrl+Shift+X) and searching for the extension titled "Docker" and produced by Microsoft. After installing this extension, the Docker icon will appear as an option on the left-hand sidebar. 



## Install Git 
We must also install Git on our new Linux environment in order to pull from the lowRISC Ibex repositories. If you are new to Git, the following resources may be helpful: 
- [https://happygitwithr.com/big-picture](https://happygitwithr.com/big-picture)
- [https://learngitbranching.js.org/](https://learngitbranching.js.org/)
- [https://www.codecademy.com/learn/learn-git/modules/learn-git-git-workflow-u/cheatsheet](https://www.codecademy.com/learn/learn-git/modules/learn-git-git-workflow-u/cheatsheet)

1. The final dependency for proceeding with the Ibex labs is to install git. From the Linux terminal, enter and run the following command: `sudo apt-get install git`. 
2. Now we are ready to set up the Ibex environment and build the docker container!



