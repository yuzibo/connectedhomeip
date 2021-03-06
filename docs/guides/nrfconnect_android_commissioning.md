# Commissioning nRF Connect Accessory using Android CHIPTool

This article describes how to use
[CHIPTool](../../src/android/CHIPTool/README.md) for Android smartphones to
commission a Nordic Semiconductor nRF52840 DK running
[nRF Connect Lock Example Application](../../examples/lock-app/nrfconnect/README.md)
onto a CHIP-enabled Thread network. The instructions are also valid for
[nRF Connect Lighting Example Application](../../examples/lighting-app/nrfconnect/README.md).

<hr>

-   [Overview](#overview)
-   [Requirements](#requirements)
-   [Building and programming OpenThread RCP firmware](#building-rcp-firmware)
-   [Configuring PC as Thread Border Router](#configuring-pc)
    -   [Forming Thread network](#forming-thread-network)
    -   [Configuring Wi-Fi hotspot](#configuring-hotspot)
-   [Building and programming nRF Connect Lock Example Application](#building-example)
-   [Building and installing Android CHIPTool](#building-chiptool)
-   [Preparing accessory device](#preparing-accessory)
-   [Commissioning accessory device](#commissioning-accessory)
-   [Sending CHIP commands](#sending-chip-commands)

<hr>

<a name="overview"></a>

## Overview

The commissioning process is composed of the following main stages:

-   CHIPTool discovers a CHIP accessory device over Bluetooth LE.
-   CHIPTool establishes a secure channel to the device over Bluetooth LE, and
    sends CHIP operational credentials and Thread provisioning data.
-   The accessory device joins a CHIP-enabled Thread network.

Bluetooth LE is only used during the commissioning phase. Afterwards, only the
IP connectivity between the smartphone and the accessory device is needed to
send operational messages. Since a typical smartphone does not have a Thread
radio built-in, extra effort is needed to prepare a fully-fledged testing
environment. This page describes how to build a Thread Border Router using a PC
with a spare Wi-Fi card and an
[OpenThread Radio Co-Processor](https://openthread.io/platforms/co-processor)
device.

The following diagram shows the connectivity between network components required
to allow communication between devices running the CHIPTool and Lock
applications:

```
               +--------------------+
               |    Smartphone      |
     +---------|  Android CHIPTool  |---------+
     |         +--------------------+         |
     |                                        | Wi-Fi
     |                                        |
     |                                +---------------+  Ethernet  +----------+
     |                                |      PC       |------------| Internet |
     |                                +---------------+            +----------+
     |                                        |
     | Bluetooth LE                           | USB
     |                                        |
     |                                +---------------+
     |                                |  nRF52840 DK  |
     |                                | OpenThread RCP|
     |                                +---------------+
     |                                        |
     |         +--------------------+         | Thread
     |         |    nRF52840 DK     |         |
     +---------|  Lock Application  |---------+
               +--------------------+
```

<hr>

<a name="requirements"></a>

## Requirements

You need the following hardware and software to build a Thread Border Router:

-   Two nRF52840 DK (PCA10056)

    -   One nRF52840 DK is needed to run
        [OpenThread Radio Co-Processor](https://openthread.io/platforms/co-processor)
        firmware and can be replaced with another compatible device like
        nRF52840 Dongle.

-   Smartphone compatible with Android 8.0 or later
-   PC with the following characteristics:

    -   Software: Ubuntu 20.04 and Docker installed
    -   Hardware: A spare Wi-Fi card

While this page references Ubuntu 20.04, all the procedures can be completed
using other popular operating systems.

<hr>

<a name="building-rcp-firmware"></a>

## Building and programming OpenThread RCP firmware

OpenThread RCP firmware is required to allow the PC to communicate with Thread
devices. Run the commands mentioned in the following steps to build and program
the RCP firmware onto an nRF52840 DK:

1.  Download and install the
    [nRF Command Line Tools](https://www.nordicsemi.com/Software-and-Tools/Development-Tools/nRF-Command-Line-Tools).
2.  Clone the OpenThread repository into the current directory:

        $ git clone https://github.com/openthread/openthread.git

3.  Enter the _openthread_ directory:

        $ cd openthread

4.  Install OpenThread dependencies:

        $ ./script/bootstrap

5.  Set up the build environment:

        $ ./bootstrap

6.  Build OpenThread for the nRF52840 DK:

         $ make -f examples/Makefile-nrf52840

    This creates an RCP image in the `bin/ot-rcp` directory.

7.  Convert the RCP image to HEX format:

        $ arm-none-eabi-objcopy -O ihex output/nrf52840/bin/ot-rcp output/nrf52840/bin/ot-rcp.hex

8.  Program the RCP firmware:

        $ nrfjprog --chiperase --program output/nrf52840/bin/ot-rcp.hex --reset

9.  Disable the Mass Storage feature on the device:

         $ JLinkExe
         J-Link>MSDDisable
         Probe configured successfully.
         J-Link>exit

    This is required, so that the feature
    [does not interfere](https://github.com/openthread/openthread/blob/master/examples/platforms/nrf528xx/nrf52840/README.md#mass-storage-device-known-issue)
    with core RCP functionalities. The setting remains valid even if you program
    another firmware onto the device.

10. Power-cycle the device to apply the changes.

<hr>

<a name="configuring-pc"></a>

## Configuring PC as Thread Border Router

To make your PC work as a Thread Border Router, complete the following tasks:

1. Form a Thread network using the OpenThread RCP device and configure IPv6
   packet routing to the network.
2. Configure a Wi-Fi hotspot using a spare Wi-Fi card on your PC.

<a name="forming-thread-network"></a>

### Forming Thread network

To form a Thread network, complete the following steps:

1.  Create an IPv6 network for the OpenThread Border Router (OTBR) container in
    Docker:

        $ docker network create --ipv6 --subnet fd11:db8:1::/64 -o com.docker.network.bridge.name=otbr0 otbr

2.  Start the OTBR container using the following command. In the last line,
    provide the device node name of the nRF52840 kit that is running the RCP
    firmware before `:/dev/radio` (in this case, the name is _/dev/ttyACM0_):

        $ docker run -it --rm --privileged --network otbr -p 8080:80 \
                --sysctl "net.ipv6.conf.all.disable_ipv6=0 net.ipv4.conf.all.forwarding=1 net.ipv6.conf.all.forwarding=1" \
                --volume /dev/ttyACM0:/dev/radio openthread/otbr --radio-url spinel+hdlc+uart:///dev/radio

3.  Open the `http://localhost:8080/` address in a web browser.
4.  Click **Form** in the menu to the left. The network forming creator window
    appears.
5.  Make sure that the On-Mesh Prefix is set to `fd11:22::`. This value is used
    later to configure the IPv6 packet routing.
6.  Click the **Form** button at the bottom of the window to form a new Thread
    network using the default settings.
7.  Run the following command to learn which IPv6 address was assigned to the
    OTBR container in Docker.

         $ docker network inspect otbr | grep IPv6Address

    After checking the output, the address will most likely be `fd11:db8:1::2`.

8.  To ensure that packets targeting Thread network nodes are routed through the
    OTBR container in Docker, run the following command, with _fd11:db8:1::2_
    replaced with the actual address obtained in the previous step:

        $ sudo ip -6 route add fd11:22::/64 dev otbr0 via fd11:db8:1::2

<a name="configuring-hotspot"></a>

### Configuring Wi-Fi hotspot

To configure a Wi-Fi hotspot using a spare Wi-Fi card on your PC, complete the
following steps:

1.  Open the Ubuntu settings widget by running the following command:

        $ gnome-control-center

2.  Go to the Wi-Fi settings.
3.  Click the three-dot icon at the title bar and select the **Turn On Wi-Fi
    Hotspot...** option.
4.  Enter your network name and password and click **Turn On**.
5.  Run the following command to assign a well-known IPv6 address to the hotspot
    interface:

        $ nmcli connection modify Hotspot ipv6.addresses fd11:db8:2::1/64

6.  Run the following command to install Routing Advertisement Daemon (radvd),
    which enables the IPv6 auto-configuration of devices that connect to the
    hotspot:

        $ sudo apt install radvd

7.  Learn the hotspot interface name by running the following command:

        $ nmcli connection show Hotspot | grep interface-name

8.  Add the following lines into `/etc/radvd.conf`, with _wlo1_ replaced with
    the hotspot interface name obtained in the previous step:

        interface wlo1
        {
          MinRtrAdvInterval 3;
          MaxRtrAdvInterval 4;
          AdvSendAdvert on;
          AdvManagedFlag on;
          prefix fd11:db8:2::/64
          {
             AdvValidLifetime 14300;
             AdvPreferredLifetime 14200;
          };
        };

9.  Start the radvd service by running the following command:

        $ systemctl start radvd

To automatically start the radvd service on every reboot, run the following
command:

    $ systemctl enable radvd

> If you use Ubuntu 18.04, a DHCP server is not configured automatically when
> creating a Wi-Fi hotspot. As a result, devices that connect to the hotspot
> will not be assigned an IPv4 address and may not work properly. To address the
> problem, install and configure a DHCP server on the hotspot interface. For
> example, you can use
> [isc-dhcp-server](https://help.ubuntu.com/community/isc-dhcp-server).

<hr>

<a name="building-example"></a>

## Building and programming nRF Connect Lock Example Application

See
[nRF Connect Lock Example Application README](../../examples/lock-app/nrfconnect/README.md)
to learn how to build and program the example onto an nRF52840 DK.

<hr>

<a name="building-chiptool"></a>

## Building and installing Android CHIPTool

To build the CHIPTool application for your smartphone, read
[Android CHIPTool README](../../src/android/CHIPTool/README.md).

After building, install the application by completing the following steps:

1.  Install the Android Debug Bridge (adb) package by running the following
    command:

        $ sudo apt install android-tools-adb

2.  Enable **USB debugging** on the smartphone. See the
    [Configure on-device developer options](https://developer.android.com/studio/debug/dev-options)
    guide on the Android Studio hub for detailed information.
3.  If the **Install via USB** option is supported for your Android version,
    turn it on.
4.  Plug the smartphone into an USB port on your PC.
5.  Run the following command to install the application, with _chip-dir_
    replaced with the path to the CHIP source directory:

        $ adb install -r chip-dir/src/android/CHIPTool/app/build/outputs/apk/debug/app-debug.apk

6.  Navigate to settings on your smartphone and grant **Camera** and
    **Location** permissions to CHIPTool.

CHIPTool is now ready to be used for commissioning.

<hr>

<a name="preparing-accessory"></a>

## Preparing accessory device

To prepare the accessory device for commissioning, complete the following steps:

1.  Use a terminal emulator to connect to the UART console of the accessory
    device. For details, see the
    [Using CLI in nRF Connect SDK examples](nrfconnect_examples_cli.md) guide.
    This will grant you access to the application logs.
2.  Hold **Button 1** on the accessory device for more than 6 s to trigger the
    factory reset of the device.
3.  Find a message similar to the following one in the application logs:

        I: 666[SVR] Copy/paste the below URL in a browser to see the QR Code:
                https://dhrishi.github.io/connectedhomeip/qrcode.html?data=CH%3AI34DV%2A-00%200C9SS0

4.  Open the URL in a web browser to have the commissioning QR code generated.
5.  Push **Button 4** on the device to start Bluetooth LE advertising.

<hr>

<a name="commissioning-accessory"></a>

## Commissioning accessory device

To commission the accessory device onto the Thread network created in the
[Forming Thread network](#Forming-a-Thread-network) section, complete the
following steps:

1. Enable **Bluetooth** and **Location** services on your smartphone.
2. Connect the smartphone to the Wi-Fi Hotspot created in the
   [Configuring Wi-Fi hotspot](#Configuring-a-Wi-Fi-hotspot) section.
3. Open the CHIPTool application on your smartphone.
4. Tap the **PROVISION CHIP DEVICE WITH THREAD** button and scan the
   commissioning QR code. Several notifications will appear, informing you of
   commissioning progress with scanning, connection, and pairing. At the end of
   this process, the Thread network settings screen appears.
5. In the Thread network settings screen, use the default settings and tap the
   **SAVE NETWORK** button to send a Thread provisioning message to the
   accessory device.

You will see the "Network provisioning completed" message when the accessory
device successfully joins the Thread network.

<hr>

<a name="sending-commands"></a>

## Sending CHIP commands

Once the device is commissioned, the following screen appears:

![CHIPTool device control screen](../../docs/images/CHIPTool_device_commissioned.jpg)

This means that the provisioning is completed successfully and you are connected
to the device. Check the connection with the following steps:

1. Verify that the text box on the screen is not empty and contains the IPv6
   address of the accessory device.
2. Tap the following buttons to change the lock state:

    - **ON** and **OFF** buttons lock and unlock the door, respectively.
    - **TOGGLE** changes the lock state to the opposite.

The **LED 2** on the device turns on or off based on the changes of the lock
state.
