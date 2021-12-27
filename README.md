# STM32 Base Project

This base project is designed to provide a basic VS code setup for working with STM32 microcontrollers in conjunction with the [CubeMX Initialization Code Generator](https://www.st.com/en/development-tools/stm32cubemx.html).
I'm personally a student currently developing on the architecture and I simply hate the Cube MX IDE. _(I'm sorry.)_

There might be lots of comments in this base-project, because it helps me in remembering what I made here.

It contains:

 - [launch.json](/.vscode/launch.json)
 - [tasks.json](/.vscode/tasks.json)
 - [c_cpp_properties.json](/.vscode/c_cpp_properties.json)

Additionally, there *(will be)* ~~are~~ the following guides:

- [Windows](/guides/Windows.md)
- [WSL2 (Debian)](/guides/WSL2.md)

Designed and tested on Windows 11 with WSL2 (Debian). 
Device under test was the STM32L432KC.

If you're using Linux, you should be able to follow along and ignore the Windows parts.
_(However, I didn't attempt to replicate in Linux, so there might be issues in the guide)_

Inspired by: https://github.com/RoboGnome/efr32_base

# TODO

- [ ] VS Code Setup (Installation + Plugins)
- [ ] Guidance for Windows
  - [ ] Make for Windows
  - [ ] gcc for ARM
  - [ ] CodeMX under Windows
  - [ ] Project configuration
- [ ] Guidance for WSL2 in Linux _(based on Debian Distribution)_
  - [ ] Make
  - [ ] gcc for ARM
  - [ ] CodeMX under WSL2
  - [ ] Project configuration

## Contents:

* [1. Dependencies](#1-dependencies)
* [1.1 Personal Recommandations](#11-personal-recommandations)
* [2. Getting Started](#2-getting-started)
  * [2.1 Build using the Command Line](#21-build-using-the-command-line)
  * [2.2 Build using VS Code](#22-build-using-vs-code)
  * [2.3 Debugging in WSL2 with USB Passthrough](#23-debugging-in-wsl2-with-usb-passthrough)
    * [2.3.1 Setup USB Passthrough for the STM32](#231-setup-usb-passthrough-for-the-stm32)
    * [2.3.2 Debug with VS Code](#232-debug-with-vs-code)
  * [2.4 Debugging in WSL2 with st-util running under Windows](#24-debugging-in-wsl2-with-jlinkgdbserver-running-under-windows)

## 1) Dependencies

 - [VSCode](https://code.visualstudio.com/)
 - [C/C++ Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools-extension-pack)
 - Additionally, for WSL2: 
   - [usbipd-win](https://github.com/dorssel/usbipd-win)
   - make => `sudo apt install build-essential`
   - arm-none-eabi-gcc => `sudo apt install gcc-arm-none-eabi`

### 1.1) Personal Recommendations

Here are some personal & unrelated recommendations for VS code:

 - [Project Manager](https://marketplace.visualstudio.com/items?itemName=alefragnani.project-manager)

---

## 2) Getting Started

Download the repository and open VS Code inside WSL2.

- [ ] This part is not finished yet.

### 2.1) Build Using the Command Line

	$ mkdir build
	$ make

### 2.2) Build Using VS Code

There are a number of the vs code tasks that are already predefined.

---

### 2.3) Debugging in WSL2 with USB Passthrough

The newest WSL2 kernel version includes USBIP based USB passthrough.
This means, we can debug directly out of WSL without any Windows tools.


#### 2.3.1) Setup USB Passthrough for the STM32

First follow the steps here to install the usb passthrough support:
https://devblogs.microsoft.com/commandline/connecting-usb-devices-to-wsl/

Make sure to install the usbipd-win server mentioned at the start of the link above.

Debugging using a ST-Link device requires the ST-Link Tools to be installed.

Download and install from https://www.st.com/en/development-tools/stsw-link004.html#get-software for windows.
For Linux: `sudo apt install stlink-tools`
*(Additionally you might* `install stlink-gui` *if you'd like to have a GUI too)*

Now you need to forward the USB connection from windows to WSL2.
Open a powershell or bash with administrative rights.
WSL2 can also call windows executables, so the only thing that counts is that you have opened the terminal with windows administrative rights.

Next plugin the STM32 board and list all USB devices:

	usbipd.exe wsl list

You should see something like this:

	BUSID  DEVICE                                                        STATE
	2-1    ST-Link Debug, USB Mass Storage Device, STMicroelectronic...  Not attached

Now attach it to WSL2 and check again:

	usbipd.exe wsl attach --busid 2-1
	usbipd.exe wsl list
	
	BUSID  DEVICE                                                        STATE
	2-1    ST-Link Debug, USB Mass Storage Device, STMicroelectronic...  Attached - Debian

Great! The STM32 is attached inside WSL2. However, only `root` user can actually interact with the device due to permissions.

So we need to configure `udev` rules for being able to use the device inside WSL2.
I provided an example in [udev/stlink.rules](/udev/stlink.rules) which you can copy into `/etc/udev/rules.d/` to get you started. **Read the comments and modify that file to make it work.**

Since `udev` doesn't normally run inside WSL2, we need to start it:

```bash
sudo service udev start
sudo udevadm control --reload
```

The above takes affect once you **re-attach the device** with `usbipd`.

```
usbipd.exe wsl detach --busid 2-1
usbipd.exe wsl attach --busid 2-1
```

Inside WSL2 you should see new devices under `/dev/` and you can check their permissions via `stat /dev/<device>`.
Mine look like this:

```bash
❯ stat /dev/stlinkv2-1_0
  File: /dev/stlinkv2-1_0 -> ttyACM0
  Size: 7               Blocks: 0          IO Block: 4096   symbolic link
Device: 5h/5d   Inode: 169         Links: 1
Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-12-27 20:49:13.220293833 +0100
Modify: 2021-12-27 20:41:38.010287625 +0100
Change: 2021-12-27 20:41:38.010287625 +0100
 Birth: -

❯ stat /dev/stlinkv2-1_1
  File: /dev/stlinkv2-1_1 -> bus/usb/001/003
  Size: 15              Blocks: 0          IO Block: 4096   symbolic link
Device: 5h/5d   Inode: 167         Links: 1
Access: (0777/lrwxrwxrwx)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-12-27 20:49:13.220293833 +0100
Modify: 2021-12-27 20:41:38.010287625 +0100
Change: 2021-12-27 20:41:38.010287625 +0100
 Birth: -

❯ stat /dev/ttyACM0
  File: /dev/ttyACM0
  Size: 0               Blocks: 0          IO Block: 4096   character special file
Device: 5h/5d   Inode: 165         Links: 1     Device type: a6,0
Access: (0666/crw-rw-rw-)  Uid: (    0/    root)   Gid: (   46/ plugdev)
Access: 2021-12-27 20:41:38.010287625 +0100
Modify: 2021-12-27 20:41:38.010287625 +0100
Change: 2021-12-27 20:41:38.010287625 +0100
 Birth: -
```

#### 2.3.2) Debug with VS Code

Now go to VS Code, where you build the debug version of the example firmware and just hit F5.
If you used the bootstrap script then it should just work.
Otherwise, go to the .vscode/launch.json file and adapt the paths to the tools to your specific setup.
If you are not using the script, you will probably need to set these 2 properties:

	"serverpath": "<path_to_local_jlink_binaries>"
	"armToolchainPath": "<path to your arm_none_eabi_gcc binary folder>"

The cortex-debug extension can show the actual register values of chip while debugging.
For that purpose, the launch file uses a svd file that is part of the platforms submodule and can
be found here:
	${workspaceFolder}/libs/platforms/chipRegisterDescriptions/EFR32MG12P332F1024GL125.svd

If you use another chip, you need to add your own SVD file.
Here is a blog post where you can find other SVD files:
	https://community.silabs.com/s/article/svd-file-for-efm32-device?language=en_US

With all of this set in place, you can use the visual debugger of vs code to debug you 
microcontroller!

---

### 2.4) Debugging in WSL2 with St-Util Running under Windows

In the [.vscode/launch.json](.vscode/launch.json) file there is a launch configuration called `WSL2-External` for a debug session using WSL 2 and an external gdb server.

To automate the process, the launch and task scripts use some environment variables.
Add these lines to your WSL `~/.bashrc` so that these variables are always set when you open up a wsl bash:


	export WSL_HOST_IP=$(cat /etc/resolv.conf | sed -rn 's|nameserver (.*)|\1|p')

You will need to restart *(not reload)* your VS Code session in order for those variables to take effect.

The launch configuration runs a task that starts the st-util on the windows side, 
and it connects to it using the `WSL_HOST_IP` environment variable.

Also check the `launch ST-Link` task in [.vscode/tasks.json](.vscode/tasks.json). 
The path to the `st-util.exe` needs to be contained in the PATH environment variable.

You probably need to adapt according to your installation.

After that just hit F5 and start a debugging session.
The cortex-debug extension can show the actual register values of chip while debugging.
For that purpose, the launch file uses a svd file which can be found under [platforms/STM32L4x2.svd](platforms/STM32L4x2.svd)

If you use another chip, you need to add your own SVD file.
Here is a repository where you can find other SVD files for multiple platforms: https://github.com/posborne/cmsis-svd/tree/master/data

Note that the debugging session using USB passthrough runs much smoother and the overall debugging experience is faster.

With all of this set in place, you can use the visual debugger of vs code to debug your 
microcontroller!
