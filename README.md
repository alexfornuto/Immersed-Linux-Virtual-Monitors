# [Immersed](https://immersed.com/) Linux Virtual Monitors Guide

**Unlock Immersed full productivity powers**

> [!CAUTION]  
>
> ## Disclaimer Notice - Use at Your Own Risk
>
> The instructions provided in this guide for setting up a virtual monitor on a Linux system is offered as-is, without any warranties or guarantees. Users should possess the necessary technical skills, back up their data, and understand that system configurations can vary. The guide may involve third-party software and potential security risks. Users are responsible for legal compliance and should use this guide at their own risk. The guide provider disclaims liability for any damages or issues resulting from its use. If you have doubts, consider seeking professional assistance or community advice.

## Summary

- [Immersed Linux Virtual Monitors Guide](#immersed-linux-virtual-monitors-guide)
  - [Summary](#summary)
  - [Intel driver](#intel-driver)
  - [Nvidia driver](#nvidia-driver)
  - [EVDI module](#evdi-module)
    - [Dependencies](#dependencies)
    - [Compiling and Installing the EVDI Module](#compiling-and-installing-the-evdimodule)
    - [Module Activation](#module-activation)
    - [Virtual monitors setup](#virtual-monitorssetup)
    - [Limitations](#limitations)
  - [Headless ghost adapter](#headless-ghostadapter)
  - [Scripts](#scripts)
  - [References](#references)
  - [Contributors](#contributors)

If you're reading this, you're likely in the same boat as me. You've discovered that Immersed can create virtual monitors for Windows and Mac, but on Linux, this feature is marked as "unsupported." This means you can't create virtual monitors directly through the Immersed agent. For now, the known workaround is to manually set up virtual monitors. While we eagerly await Wayland support for native virtual displays on the Immersed agent, we can only currently use it with X11 displays.

The workaround varies depending on your computer's video card. In this guide, we'll cover four methods to create a virtual display on Linux and suggest which method best suits each type of video card. Here's an overview of the methods:

- **Intel card**: [Intel driver](#intel-driver), [EVDI module](#evdi-module), [Headless ghost adapter](#headless-ghostadapter);
- **Nvidia card**: [Nvidia driver](#nvidia-driver), [EVDI module](#evdi-module), [Headless ghost adapter](#headless-ghostadapter);
- **AMD card**: [EVDI module](#evdi-module), [Headless ghost adapter](#headless-ghostadapter).

It's important to note that while there's a "Dummy driver" option, I couldn't use it.

> [!WARNING]
>
> ## Xorg modifications disclaimer
>
> Modifying the xorg.conf or related X11 settings can lead to video issues, and, in some specific cases (such as encrypted disks with PopOS), it can break the boot process. Before making any modifications, it's crucial to know how to reverse any changes that affect your video. We recommend the following options:
>
> - Always create a backup file before making modifications, making it easy to revert changes. For example: `sudo cp /etc/X11/xorg.conf /etc/X11/xorg.conf.bkp`.
> - If you can boot successfully but have no video, switch to a different TTY. Use the keyboard shortcut `Ctrl + Alt + [2–6]` to switch to a shell, login, and use a terminal text editor of your choice(nano, vi, vim, emacs) to edit /etc/X11/xorg.conf or restore the backup file: `sudo cp /etc/X11/xorg.conf.bkp /etc/X11/xorg.conf`.
> - Use recovery options to operate the system in failsafe video mode and modify or recover `/etc/X11/xorg.conf`.
> - If none of the above options work, the last resort is to boot from a Linux distro live CD to modify or recover `/etc/X11/xorg.conf`. If you're using PopOS with an encrypted disk, follow this [guide](https://support.system76.com/articles/pop-recovery/#encrypted-disk) to mount your disk.

## Intel driver

For those with an Intel video card or an integrated Intel video card alongside dedicated GPUs, you can use virtual monitors generated by the Intel driver. Start by creating and editing the file `/usr/share/X11/xorg.conf.d/20-intel.conf` with the following lines:

```sh
Section "Device"
    Identifier "intelGpu"
    Driver "intel"
    Option "VirtualHeads" "1"
EndSection
```

In some configurations, like Intel + Nvidia video cards, the above configuration might not be sufficient to keep both cards working. In such cases, you'll also need to create and edit the file `/usr/share/X11/xorg.conf.d/19-nvidia.conf` with the following lines:

```sh
Section "Device"
    Identifier "nvidiaGpu"
    Driver "nvidia"
EndSection
```

After rebooting or logging out, the `xrandr` output should list a monitor called **VIRTUAL1**:

```sh
VIRTUAL1 disconnected (normal left inverted right x axis y axis)
  1920x1080 (0x1ef) 173.000MHz -HSync +VSync
        h: width  1920 start 2048 end 2248 total 2576 skew    0 clock  67.16KHz
        v: height 1080 start 1083 end 1088 total 1120           clock  59.96Hz
```

To configure resolutions for your virtual display, use `xrandr` and `cvt`. For example, let's add a 3840x1600 4k resolution:

```sh
cvt 3840 1600

output:
# 3840x1600 59.96 Hz (CVT) hsync: 99.42 kHz; pclk: 521.75 MHz
Modeline "3840x1600_60.00"  521.75  3840 4128 4544 5248  1600 1603 1613 1658 -hsync +vsync
```

With the `cvt` output use `xrandr` to create and set the new resolution:

```sh
xrandr --newmode "3840x1600"  521.75  3840 4128 4544 5248  1600 1603 1613 1658 -hsync +vsync
xrandr --addmode VIRTUAL1 "3840x1600"
xrandr --output VIRTUAL1 --mode "3840x1600"
xrandr
```

The output of the previous commands should show the virtual monitor `VIRTUAL1` as **connected**. Now, you can adjust monitor position and rotation in the system video setting. Note that in the same output, there is a `VIRTUAL2` disconnected monitor. You can repeat the following steps to add more virtual monitors, changing x by their virtual display number:

```sh
xrandr --addmode VIRTUALx "3840x1600"
xrandr --output VIRTUALx --mode "3840x1600"
xrandr
```

You can also get some scripts of these steps shared on the [Discord server](https://discord.com/channels/428916969283125268/1100046559183380480) on the script folder.

## Nvidia driver

If you have a system with a dedicated Nvidia graphics card, it doesn't inherently support virtual monitors as easily as the Intel driver. However, you can employ a clever workaround that involves configuring the Nvidia driver to force disconnected video outputs as if they were connected.

To get started, you'll need to create and edit the Xorg configuration file. You can use the `nvidia-xconfig` command to help you with this. Open a terminal and run:

```sh
sudo nvidia-xconfig
```

This command generates the `/etc/X11/xorg.conf` file, which is crucial for configuring the Nvidia driver. It also saves the previous configuration as `/etc/X11/xorg.conf.bkp`.

If you're using a laptop with an integrated Intel graphics card that controls the laptop monitor, you need to add the Intel device to the Xorg configuration to enable the use of your laptop monitor. To do this, open the `/etc/X11/xorg.conf` file with a text editor. In this file, add the following lines under the

```sh
"Device" section:
Section "Device"
    Identifier     "intelGpu"
    Driver         "intel"
    VendorName     "Intel Corporation"
    BusID          "PCI:0:2:0"
EndSection
```

The `BusID` value should match the output of the `lspci | grep VGA` command, which will list the available graphics devices. Look for the entry that corresponds to your integrated Intel graphics card and use that value.

For example, the `lspci | grep VGA` command might yield an output like this:

```sh
00:02.0 VGA compatible controller: Intel Corporation CoffeeLake-H GT2 [UHD Graphics 630]
01:00.0 VGA compatible controller: NVIDIA Corporation TU106M [GeForce RTX 2070 Mobile] (rev a1)
```

In this case, you'd use `"PCI:0:2:0"` as the `BusID`.

You'll need to identify the available monitors and their names to force-disconnect video outputs as connected. You can use the `xrand`r command for this. Run `xrandr | awk /connected/`. The output will display the connected and disconnected monitors. Note that the name of the monitor related to the physical port (such as HDMI) usually works as expected, while other types, like DisplayPort (DP), might not work and create video issues. Finding the working combination is a process of trial and error.

For example, the output could look like this:

```sh
DP-0 connected 1080x1920+3640+0 right (normal left inverted right x axis y axis) 477mm x 268mm
DP-1 disconnected (normal left inverted right x axis y axis)
DP-2 connected 2560x1600+1080+164 (normal left inverted right x axis y axis) 0mm x 0mm
DP-3 disconnected (normal left inverted right x axis y axis)
HDMI-0 connected 1080x1920+0+10 right (normal left inverted right x axis y axis) 0mm x 0mm
eDP1 connected primary 1920x1080+1720+1764 (normal left inverted right x axis y axis) 340mm x 190mm
VIRTUAL1 disconnected (normal left inverted right x axis y axis)
```

My laptop has just two physical video outputs, namely `HDMI-0` and `DP-0`, with `eDP1` as the laptop's built-in monitor. I successfully managed to establish a stable connection with `DP-0`, `DP-2`, and `HDMI-0` without encountering any video-related issues.

To force-disconnect monitors as connected, you'll need to edit the `/etc/X11/xorg.conf` file. Locate the `"Screen"` section associated with the Nvidia device. It should resemble this:

```sh
Section "Screen"
    Identifier     "Screen0"
    Device         "Nvidia0"
    Monitor        "Monitor0"
    DefaultDepth    24
SubSection     "Display"
        Depth       24
    EndSubSection
EndSection
```

Edit this section to add the necessary options. The key options to add are:

- `ConnectedMonitor`: This option specifies the names of the monitors you want to treat as connected. Use the names you noted earlier. For example:

```sh
Option "ConnectedMonitor" "DP-0, HDMI-0, DP-2"
```

- `ModeValidation`: options to disable driver verifications on display resolution and refresh rate.

Here's an example of the updated `"Screen"` section:

```sh
Section "Screen"
    Identifier     "Screen0"
    Device         "Nvidia0"
    Monitor        "Monitor0"
    DefaultDepth    24
    Option         "ConnectedMonitor" "DP-0, HDMI-0, DP-2"
    Option         "ModeValidation" "NoDFPNativeResolutionCheck,NoVirtualSizeCheck,NoMaxPClkCheck,NoHorizSyncCheck,NoVertRefreshCheck,NoWidthAlignmentCheck"
    SubSection     "Display"
        Depth       24
    EndSubSection
EndSection
```

That's it. Now, you should reboot your computer or log out to apply the changes. Once you've done this, you'll be able to adjust the resolution, position, and rotation of your monitors in the system's video settings.

If you find that you're missing a particular resolution option or want to set a custom resolution, you can create a new mode for your monitor. For example, to add a custom "3840x1600" display resolution for `DP-2` monitor, use the `cvt` and `xrandr` commands:

```sh
xrandr --newmode "3840x1600"  521.75  3840 4128 4544 5248  1600 1603 1613 1658 -hsync +vsync
xrandr --addmode DP-2 "3840x1600"
xrandr --output DP-2 --mode "3840x1600"
xrandr
```

You can create custom resolutions with the `cvt` command:

```sh
cvt 3840 1600

output:
# 3840x1600 59.96 Hz (CVT) hsync: 99.42 kHz; pclk: 521.75 MHz
Modeline "3840x1600_60.00"  521.75  3840 4128 4544 5248  1600 1603 1613 1658 -hsync +vsync
```

## EVDI module

The EVDI (External Virtual Device Interface) kernel module is an integral part of the DisplayLink solution and offers the ability to create virtual video outputs. However, it's worth noting that using the EVDI module involves a few more steps, and one of the most critical considerations is the impact of secure boot. Only signed kernel modules can be loaded when secure boot is active. Below are the steps to set up virtual monitors using the EVDI module on your Linux system.

### Dependencies

Before you can begin with the EVDI module, ensure you have the necessary dependencies installed based on your Linux distribution:

- For Debian-based systems (such as Ubuntu):

```sh
sudo apt install dkms libdrm-dev linux-headers-$(uname -r)
```

- For Fedora/RHEL-based systems:

```sh
sudo dnf install dkms libdrm-devel kernel-headers-$(uname -r)
```

- For Steam Deck:

```sh
sudo pacman -Syy base-devel holo-rel/linux-headers linux-neptune-headers holo-rel/linux-lts-headers git glibc gcc gcc-libs linux-api-headers libarchive libdrm dkms
```

- For Arch-based systems: You can use the AUR (Arch User Repository) to simplify the compilation and installation process. You can use an AUR helper like yay to compile and install the EVDI module:

```sh
yay -S evdi-git
```

### Compiling and Installing the EVDI Module

Now, you'll need to compile and install the EVDI kernel module:

```sh
git clone https://github.com/DisplayLink/evdi.git
cd evdi/module
make
sudo make install_dkms
```

### Module Activation

After you've successfully installed the EVDI module, you'll need to load it into the kernel. However, an important consideration related to secure boot is that only signed kernel modules can be loaded. To check if secure boot is active on your system, run the following command:

```sh
mokutil --sb-state
```

If secure boot is active and you want to use the EVDI module, you may need to disable secure boot in your BIOS settings. Be aware that this decision has security implications, so proceed with caution. Another option is to sign the module, but I will not cover it in this guide.

Assuming you've addressed secure boot as needed, you can now load the EVDI module:

```sh
sudo modprobe evdi initial_device_count=4
```

### Virtual monitors setup

After successfully loading the EVDI module, you'll notice that new virtual monitors have been detected by your system when you run `xrandr | awk /connected/`:

```sh
DP-0 connected 1080x1920+3640+0 right (normal left inverted right x axis y axis) 477mm x 268mm
DP-1 disconnected (normal left inverted right x axis y axis)
DP-2 connected 2560x1600+1080+164 (normal left inverted right x axis y axis) 0mm x 0mm
DP-3 disconnected (normal left inverted right x axis y axis)
HDMI-0 connected 1080x1920+0+10 right (normal left inverted right x axis y axis) 0mm x 0mm
DVI-I-5-2 disconnected (normal left inverted right x axis y axis)
DVI-I-4-1 disconnected (normal left inverted right x axis y axis)
DVI-I-3-4 disconnected (normal left inverted right x axis y axis)
DVI-I-2-3 disconnected (normal left inverted right x axis y axis)
eDP1 connected primary 1920x1080+1720+1764 (normal left inverted right x axis y axis) 340mm x 190mm
VIRTUAL1 disconnected (normal left inverted right x axis y axis)
```

You can see that there are virtual monitors labeled as `DVI-I-X-Y`, which are currently disconnected.

The next step is to force-connect these virtual monitors. This can be achieved through two methods: using a Python wrapper provided in the EVDI repository or using the `xrandr` command and setting a kernel variable. The `xrandr` method is explored here, as it doesn't require additional dependencies.

The following steps provide information to set up one virtual `DVI-I-X-Y` monitor, but you can repeat these steps for each additional virtual monitor you want to create.

- List the providers and get their numbers with `xrandr --listproviders`:

```sh
Provider 0: id: 0x1b8 cap: 0x1, Source Output crtcs: 4 outputs: 5 associated providers: 5 name:NVIDIA-0
Provider 1: id: 0x4d5 cap: 0x2, Sink Output crtcs: 1 outputs: 1 associated providers: 1 name:modesetting
Provider 2: id: 0x4b3 cap: 0x2, Sink Output crtcs: 1 outputs: 1 associated providers: 1 name:modesetting
Provider 3: id: 0x491 cap: 0x2, Sink Output crtcs: 1 outputs: 1 associated providers: 1 name:modesetting
Provider 4: id: 0x46f cap: 0x2, Sink Output crtcs: 1 outputs: 1 associated providers: 1 name:modesetting
Provider 5: id: 0x295 cap: 0xb, Source Output, Sink Output, Sink Offload crtcs: 4 outputs: 2 associated providers: 1 name:Intel
```

- Set the provider output source on the `DVI-I-X-Y` providers with the `xrandr` command. For example, to set a DVI-I-x provider as the source output with `xrandr --setprovideroutputsource 1 0`. In this example, "1" is the number of the `DVI-I-X-Y` provider, and "0" is the source output. You can repeat this step for other `DVI-I-X-Y` providers as needed.
- Set the desired resolution for the virtual display using the `xrandr` command. For example, to set a resolution of "1920x1080," run `xrandr --addmode DVI-I-2–3 "1920x1080"`, where `DVI-I-2–3` is one of my `DVI-I-X-Y` monitors from `xrandr` output. You can create custom resolutions with `cvt` and `xrandr`, for example, create a custom "3840x1600" display resolution:

```sh
cvt 3840 1600

output:
# 3840x1600 59.96 Hz (CVT) hsync: 99.42 kHz; pclk: 521.75 MHz
Modeline "3840x1600_60.00"  521.75  3840 4128 4544 5248  1600 1603 1613 1658 -hsync +vsync
```

```sh
xrandr --newmode "3840x1600" 521.75 3840 4128 4544 5248 1600 1603 1613 1658 -hsync +vsync
```

- Apply the resolution to the virtual monitor using the `xrandr` command: `xrandr --output DVI-I-2-3 --mode "1920x1080"`, where `DVI-I-2–3` is one of my `DVI-I-X-Y` monitors from `xrandr` output.
- To complete the setup, you need to set the provider as "connected." First, you'll need to find the provider's path using the following command:

```sh
sudo ls /sys/kernel/debug/dri/

output:
0  1  128  129 2  3  4  5
```

- This command will list the available providers. You'll find provider directories with numerical names. Next, you can choose one of the provider directories corresponding to your `DVI-I-X` provider. For example:

```sh
sudo ls /sys/kernel/debug/dri/2

output:
clients  crtc-0  DVI-I-1  framebuffer  gem_names  internal_clients  name  state
```

- In this example, `DVI-I-2–3` is connected to the provider `2`. To force the `DVI-I-2–3` provider as connected, run the following command, replacing the path with the appropriate provider path:

```sh
sudo sh -c "echo on > /sys/kernel/debug/dri/2/DVI-I-1/force"
xrandr --listproviders

output:
DP-0 connected 1080x1920+3640+0 right (normal left inverted right x axis y axis) 477mm x 268mm
DP-1 disconnected (normal left inverted right x axis y axis)
DP-2 connected 2560x1600+1080+164 (normal left inverted right x axis y axis) 0mm x 0mm
DP-3 disconnected (normal left inverted right x axis y axis)
HDMI-0 connected 1080x1920+0+10 right (normal left inverted right x axis y axis) 0mm x 0mm
DVI-I-5-2 disconnected (normal left inverted right x axis y axis)
DVI-I-4-1 disconnected (normal left inverted right x axis y axis)
DVI-I-3-4 disconnected (normal left inverted right x axis y axis)
DVI-I-2-3 connected (normal left inverted right x axis y axis)
eDP1 connected primary 1920x1080+1720+1764 (normal left inverted right x axis y axis) 340mm x 190mm
VIRTUAL1 disconnected (normal left inverted right x axis y axis)
```

Repeat the step to the other desired virtual providers.

### Limitations

In my laptop with Intel + Nvidia video cards, I have some visual artifacts around the cursor, but some users have reported that they were able to use it with Nvidia video cards without problems.

## Headless ghost adapter

The Headless Ghost Adapter is a hardware solution that allows you to simulate that unused video output ports are connected to high-resolution virtual displays, typically at 4K resolution, depending on the specific adapter. This option is an excellent choice for individuals who require multiple virtual monitors on their Linux system, don't want to change system files, and have available video outputs on their graphics card.

You can easily find this product online by searching for `Headless ghost adapter` or you can join [Immersed discord server](https://immersed.com/community) and ask for recommendations.

## Scripts

Users of Immersed Linux Agent may encounter occasional crashes after making display changes. To mitigate this, you can use a simple script to restart Immersed when needed automatically. This can save you from manually restarting the agent, ensuring a smoother experience.

```sh
while :
do
echo "Immersed starting"
./Immersed-x86_64.AppImage
echo "Immersed stopped"
sleep 1
done
```

## References

- <https://wiki.archlinux.org/title/Extreme_Multihead>
- <https://unix.stackexchange.com/a/585078>
- <https://github.com/kbumsik/VirtScreen/issues/16#issuecomment-865128729>
- <https://github.com/dianariyanto/virtual-display-linux/issues/9#issuecomment-786389065>
- <https://www.reddit.com/r/Fedora/comments/yxkm3w/comment/iwpy5jv/?utm_source=share&utm_medium=web2x&context=3>
- <https://github.com/DisplayLink/evdi>

## Contributors

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->
