# Development environment

## Overview

Coming soon...

* [Visual Studio Code](https://code.visualstudio.com/)
* [Visual Studio Code Dev Containers](https://code.visualstudio.com/docs/devcontainers/containers)

## Linux

udev rules:

```sh
sudo tee /etc/udev/rules.d/99-jlink.rules <<'EOF'
SUBSYSTEM=="usb", ATTR{idVendor}=="1366", MODE="0666"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger
```

## MacOS

## Windows
