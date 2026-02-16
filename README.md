<p align="center">
<img src=".github/sdcard-logo.png" style="width:40%; max-width: 300px;" >
</p>

# G2-OS

A [Raspberry Pi OS](https://www.raspberrypi.org/software/) based distribution designed specifically for the **Ginger G2** 3D Printer. It includes all the necessary software and optimizations to get started with **Kalico Firmware** and **Mainsail** for 3D printing with **pellet extrusion** technology.

This project is a **fork of [MainsailOS](https://github.com/mainsail-crew/MainsailOS)**, tailored for the unique requirements of pellet-based 3D printing with the Ginger G2.

## Learn more about:

-   [Kalico (Firmware for Pellet 3D Printing)](https://github.com/gingeradditive/kalico)
-   [Moonraker (API Web Server for Kalico)](https://github.com/Arksine/moonraker)
-   [Mainsail (Web Interface for Kalico)](https://github.com/mainsail-crew/mainsail)

<!-- ## How to install G2-OS?

Detailed installation instructions can be found in our [documentation](https://docs-os.mainsail.xyz), with a section dedicated to the **Ginger G2**. We recommend using the [Raspberry Pi Imager](https://docs-os.mainsail.xyz/getting-started/raspberry-pi-os-based) for installation. -->
<!-- 
## Need help?

Join our community on [Discord](https://discord.gg/mainsail) for support. You can also check the FAQ below for common issues. -->

[![discord](https://img.shields.io/discord/758059413700345988?color=%235865F2&label=discord&logo=discord&logoColor=white&style=flat)](https://discord.gg/mainsail)

## What's included?

G2-OS comes with the following pre-installed and configured software:

-   [Kalico (Customized Firmware for Pellet 3D Printing)](https://github.com/gingeradditive/kalico)
-   [Moonraker (API for Kalico)](https://github.com/Arksine/moonraker)
-   [Mainsail (Kalico Web Interface)](https://github.com/mainsail-crew/mainsail)
-   [Crowsnest (Webcam Streaming)](https://github.com/mainsail-crew/crowsnest)
-   [Sonar (Keepalive Daemon)](https://github.com/mainsail-crew/sonar)
-   [Nginx (Web Server & Proxy)](https://nginx.org/en/)

## G2-OS also includes:

-   **Preconfigured Serial Connection** for the Ginger G2 using Hardware UART (PL011).
-   **Preinstalled Dependencies** for Kalico's Input Shaper. Simply build the [kalico_mcu](https://www.kalico.org/RPi_microcontroller.html) and install the service. See [Kalico documentation](https://www.kalico.org/Measuring_Resonances.html) for more info.
-   **Preinstalled Python3-serial package**, required for [CanBoot](https://github.com/Arksine/CanBoot).

## Support the Mainsail-Crew

MainsailOS is a passion project developed and maintained by the Mainsail Crew.
We dedicate a significant amount of our free time, almost daily, to keep the
project alive and moving forward.

Your support directly fuels our development efforts. Donations help us cover
essential costs for hardware, such as new SBCs and SD cards, which are crucial
for testing, developing new features, and expanding board compatibility.

**Q:** How do I report a bug?  
**A:** Please ensure it's not a configuration issue with:

-   Kalico
-   Moonraker
-   Crowsnest
-   Sonar

If the issue is specific to the **G2-OS** setup or the Ginger G2 printer, report it via the **G2-OS** GitHub Issues section. Provide detailed information to help us resolve the problem quickly.

**Q:** What is the philosophy behind G2-OS?  
**A:** We maintain a **KISS** principle—Keep It Simple and Straightforward. G2-OS is built on the same foundation as **MainsailOS**, but optimized for pellet printing with the Ginger G2. We aim to keep things as close to the Raspberry Pi OS and MainsailOS documentation as possible, providing extra documentation only where needed.

**Q:** How can I contribute?  
**A:** Contributions are always welcome! Please check out the [CONTRIBUTING.md](https://github.com/mainsail-crew/MainsailOS/blob/develop/CONTRIBUTING.md) for ways to support the project or submit code.

# Build your own / Development

To build your own version of **G2-OS**, simply fork this repository, enable workflows, and each push will trigger an automated image build.

For more information on local builds, please refer to [CustomPiOS](https://github.com/guysoft/CustomPiOS) and the guide ["Build a Distro From within Raspbian/Debian/Ubuntu/CustomPiOS Distros"](https://github.com/guysoft/CustomPiOS#build-a-distro-from-within-raspbian--debian--ubuntu--custompios-distros).
