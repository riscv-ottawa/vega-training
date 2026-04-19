# Development environment

## Overview

Embedded development has a reputation for being fiddly to set up.
You typically need a specific compiler (one that produces code for your microcontroller rather than your laptop), a program that talks to the debug probe on the board, a handful of supporting libraries, and sometimes a simulator.
Getting all of the above installed correctly can sometimes become a nightmare!
To avoid this, this training will use a *containerized* development environment.

> [!NOTE]
> A [container](https://www.docker.com/resources/what-container/) is a lightweight, isolated Linux environment that runs on top of your own operating system (your "host").
> Similar to virtual machines, you can think of it as an isolated box that you can use to run and install custom software inside without affecting your host system.
>
> We'll provide a custom pre-built container image for this training that contains everything you'll need.

The container image is described by a single `Containerfile` in the `vega-quickstart` repository. You will never need to read through or edit it by hand (but feel free to take a look to learn more).
The [Visual Studio Code Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers) extension handles everything: when you open the `vega-quickstart` folder in [Visual Studio Code](https://code.visualstudio.com/), it pulls the container (the first time only, which takes a few minutes), starts it, makes your project folder visible inside it, and attaches the editor to a shell running in it. From your point of view, you are just editing files and using a terminal as usual; under the hood those actions are happening inside the container.

### What is inside the container?

The container is built on Ubuntu 24.04 (a common Linux distribution). On top of that base, it includes three main pieces:

* [Custom RISC-V compiler suite](https://github.com/open-isa-org/open-isa.org/releases), often called a *toolchain*: the compiler, linker, and related tools that turn C source code into a binary the VEGAboard can execute. We use the prebuilt toolchain from OpenISA, configured for the `rv32i` instruction set (a minimal 32-bit RISC-V variant, which is what the VEGAboard's cores implement).
* [Renode](https://renode.io/), a simulator that can virtually emulate a VEGAboard. This lets you run and debug your firmware without any physical hardware attached, which is handy for getting started and for experimenting.
* [OpenOCD](https://openocd.org/), the program that communicates with the *debug probe* on the board. A debug probe is the small circuit, built into the VEGAboard, that lets your computer load firmware onto the chip and step through the running code.

Alongside the above, the image contains some additional utilities (`make`, `git`, `vim`, `minicom`, etc) and creates a regular, non-administrator user called `dev` that you will be logged in as when you open a terminal.

### How VS Code ties it together

The `devcontainer.json` file under the `.devcontainer` directory tells VS Code how to launch the container. Two details are worth knowing about:

* The container is started in privileged mode so that USB devices on your host (importantly, the debug probe on the VEGAboard) are visible inside it
* VS Code automatically installs a small set of extensions inside the container for you: C/C++ tooling, Makefile support, CMake highlighting, GitLens, a spell checker, and XML/YAML helpers. You do not need to install any of these yourself.
  - These extensions are also defined in `devcontainer.json`, feel free to add additional extensions that you typically use

## Host requirements

Your host machine only needs four things:

* A **container runtime** - this is the program that actually runs containers
  - [Docker Desktop](https://www.docker.com/products/docker-desktop/) is the easiest choice on macOS and Windows
  - On Linux you can use [Docker Engine](https://docs.docker.com/engine/) or [Podman](https://podman.io/)
* [Visual Studio Code](https://code.visualstudio.com/)
* The [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension for VS Code
* [Git](https://git-scm.com/) - the standard version control tool which you'll use to pull down the quickstart template (see the next section)

## Quickstart template

Once you've installed the above, clone the `vega-quickstart` repository to your machine:

```sh
git clone https://github.com/between-layers/vega-quickstart.git
```

Open the cloned repository folder in VS Code, and accept the prompt to "Reopen in Container". After a few minutes, VS Code should drop you into a terminal inside the container. From there you can edit code, run `make` to build firmware, run Renode to simulate the board, or (if your host is set up for it) connect to the real VEGAboard over USB. When you are done, closing VS Code shuts the container down; any code changes stay on your host.

The rest of this page covers the host-specific setup that the container cannot handle on its own. Most of it is about giving the container permission to see the VEGAboard's USB connection.

## Additional host-specific help

### Linux

Install Docker Engine (or Podman) and VS Code using your distribution's package manager or the upstream instructions. It may be worth adding your user to the `docker` group so that VS Code can talk to the container runtime without asking for a password every time (see the official Docker [post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/) for more info).

Finally, USB permissions will need one small tweak. If connectiong to the physical VEGAboard, you'll like be using the Segger J-Link debug probe provided in the box.
This device identifies itself to your computer with USB vendor ID `1366`. By default Linux only lets the root user open such devices, which is a problem because the container runs as the unprivileged `dev` user. The fix is a *[udev rule](https://wiki.archlinux.org/title/Udev#Introduction_to_udev_rules)*: a one-line configuration that tells the kernel to make the device readable and writable by everyone on the machine.

To create the udev rule, run the following commands:
```sh
sudo tee /etc/udev/rules.d/99-jlink.rules <<'EOF'
SUBSYSTEM=="usb", ATTR{idVendor}=="1366", MODE="0666"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
```

Unplug and replug the VEGAboard after applying the rule. You should then be able to flash and debug from inside the container.

> [!NOTE]
> USB device visibility inside containers can sometimes be finicky.
> If you run into issues connecting to your board:
> * Double check the board is plugged in properly
> * See if your host recognizes the device by running `lsusb` or `ls /dev/tty*`
> * If the device is there on the host, but not in the container - try restarting the container with `docker restart <container-name>`

> [!TIP]
> If you are using Podman instead of Docker, make sure its socket is enabled (`systemctl --user enable --now podman.socket`) and point the Dev Containers extension at it via the `dev.containers.dockerPath` setting.

### MacOS

Install [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/) and VS Code.
Grant it full disk access if it prompts you, otherwise it cannot share your project folder with the container.

There is one thing that the container cannot do on macOS: talk to the VEGAboard over USB. Docker Desktop's internal VM does not expose your Mac's USB ports, so the debug probe is invisible from inside the container. You can still do all editing, building, and simulation inside the container, exactly like on Linux and Windows. However, if you have a physical board and want to flash it, you will install the OpenISA SDK natively on macOS and run OpenOCD from there.

#### ARM-based

One additional exception for MacOS is if you're using a newer Apple Silicon (M1 and newer) machine. Both the container image and the OpenISA macOS bundle are Intel (x86_64). Luckily, macOS can run them transparently through Apple's Rosetta 2 translation layer.

Install it once with:
```sh
softwareupdate --install-rosetta --agree-to-license
```

Then open Docker Desktop's settings and enable "Use Rosetta for x86/amd64 emulation on Apple Silicon" under *General*.

> [!NOTE]
> Builds and simulations may be noticeably slower than on an Intel Mac or a native Linux machine due to emulation, but everything should still work.

#### Installing the OpenISA SDK (for flashing real hardware)

OpenISA publishes a macOS bundle that contains their prebuilt toolchain and a working OpenOCD. You only need this on macOS, and only for flashing.

1. From the [open-isa.org v1.0.0 release](https://github.com/open-isa-org/open-isa.org/releases/tag/1.0.0), download these two files into a working directory of your choice (for example `~/rv32m1/`):
   - `Toolchain_Mac.tar.gz`
   - `rv32m1_sdk_riscv_installer.sh`

2. Run the SDK installer from that same directory. It will unpack both the SDK and the toolchain/OpenOCD side by side:

   ```sh
   cd ~/rv32m1
   chmod +x rv32m1_sdk_riscv_installer.sh
   ./rv32m1_sdk_riscv_installer.sh
   ```

3. Add OpenOCD to your shell's `PATH` so you can invoke it from any terminal. Adjust the path if the installer puts it somewhere different on your system:

   ```sh
   echo 'export PATH="$HOME/rv32m1/Toolchain_Mac/riscv32-unknown-elf-gcc/openocd/bin:$PATH"' >> ~/.zshrc
   source ~/.zshrc
   openocd --version
   ```

When you are ready to flash, open a **native macOS terminal** (not the one inside the container) in the same project folder and run the project's flash target, or invoke `openocd` directly with the project's `.cfg`. The container and the host share the folder, so the firmware you just built inside the container is already visible from your Mac terminal.

### Windows

Install [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/) with the WSL2 backend (this is the default in recent versions). [WSL2](https://learn.microsoft.com/en-us/windows/wsl/about), short for Windows Subsystem for Linux 2, is a lightweight Linux VM that Windows ships with; Docker Desktop uses it to run containers. You will also want VS Code and the Dev Containers extension installed on the Windows side. The extension launches the container inside the WSL2 VM and then connects to it.

For USB access to the VEGAboard, install [`usbipd-win`](https://github.com/dorssel/usbipd-win), a small open-source tool that forwards USB devices from Windows into a WSL2 environment. After installing it, open an elevated PowerShell window and run:

```powershell
usbipd list
usbipd bind --busid <BUSID>
usbipd attach --wsl --busid <BUSID>
```

Replace `<BUSID>` with the identifier shown for the J-Link device in the output of `usbipd list` (look for vendor ID `1366`). You will need to re-run the `attach` command each time you unplug the board or reboot. Once attached, running `lsusb` inside the container should list the probe.
