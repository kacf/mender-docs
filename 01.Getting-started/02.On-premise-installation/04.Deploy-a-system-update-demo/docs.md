---
title: Deploy a system update demo
taxonomy:
  category: docs
---

In this tutorial we will deploy a full rootfs update to
a physical device, the Raspberry Pi 3 or BeagleBone Black, using the
Mender server.

## Prerequisites

The test environment should be set up and working successfully
as described in [Install a Mender demo server](../create-a-test-environment).

We also strongly recommend that you complete the tutorial that comes with the Mender GUI so
that you have a basic understanding of how Mender works before moving on to connecting a physical device.

### A device to test with

You need one or more BeagleBone Black or Raspberry Pi 3.
To make it easy to provision the device we will use
a SD card to store the OS, so you will need one SD card
(4 GB or larger) per device.

### Disk image and Artifacts

Get the disk image and Artifacts for your board(s) from [the Downloads
section](../../../downloads#disk-images).

!!! It is possible to use this tutorial with _any_ physical board, as long as you have integrated Mender with it. In this case you cannot use the demo Artifacts we provide in this tutorial, but you need to build your own artifacts as described in [Building a Mender Yocto Project image](../../../artifacts/yocto-project/building).

### Mender-Artifact tool

Download the prebuilt `mender-artifact` binary for your platform following the links
in [Downloads section](../../../downloads#mender-artifact-tool).

Please see [Modifying a Mender Artifact](../../../artifacts/modifying-a-mender-artifact)
for a more detailed overview.

### Network connectivity

The device needs to have network set up
so it can connect directly to your workstation
(where you have the Mender server running).

! By default the Mender client will use ports **443** and **9000** to connect to the server. You can test the connection from your client later with networking tools like `telnet`.

If you have just one device, you could connect your
workstation and the device using a direct
Ethernet cable and use static IP addresses at both ends.
For multiple devices, you need a router or switch.

For the rest of the tutorial we will assume
`$IP_OF_MENDER_SERVER_FROM_DEVICE` will expand to the IP address
that your device(s) can connect to the Mender server.

!!! If you are using `bash`, you can set a variable to make the rest of the tutorial easier, for example `IP_OF_MENDER_SERVER_FROM_DEVICE="192.168.10.1"`.

! Using static IP addresses with one device and workstation is quite easy. If you are using several devices, we strongly recommend using a setup with dynamic IP assignment like a router with DHCP support. Otherwise you need to take care to preserve the unique IP address configuration of each device when provisioning the storage and deploying rootfs updates.

## Prepare the disk image

! Please make sure to set a shell variable that expands correctly with `$IP_OF_MENDER_SERVER_FROM_DEVICE` or edit the commands below accordingly.

Locate the demo _disk image_ (`*.sdimg`) you downloaded for your device.
This image contains _all the partitions_ of the storage device, as described in [Partition
layout](../../../devices/general-system-requirements#partition-layout).

You can decompress a `.xz` image like the following:

```bash
unxz <PATH-TO-YOUR-DISK-IMAGE>.sdimg.xz
```

Or, if it is a `.gz` image, like this:

```bash
gunzip <PATH-TO-YOUR-DISK-IMAGE>.sdimg.gz
```

!!! The Mender images come with a predetermined size for the root filesystems, which may be too small for some use cases where a lot of space is required for applications. If you are building your own disk image by following [Building a Mender Yocto Project image](../../../artifacts/yocto-project/building), you can configure the desired space usage with the Yocto Project variable [MENDER_STORAGE_TOTAL_SIZE_MB](../../../artifacts/yocto-project/variables#mender_storage_total_size_mb).

We need to change some configuration settings in this image so that
the Mender client successfully connects to your Mender
server when it starts.

### Insert the address of Mender server

!!! If you are using a Raspbian image, you can skip this section and jump to [the next section](#set-a-static-device-ip-address-and-subnet).

First set a shell variable describing the image name, by replacing `<sdimg>` in this snippet:

```bash
MENDER_IMGPATH=<sdimg>
```

Then run this command:

```bash
mender-artifact cat $MENDER_IMGPATH:/etc/hosts | sed "\$a ${IP_OF_MENDER_SERVER_FROM_DEVICE} docker.mender.io s3.docker.mender.io" > tmpf; mender-artifact cp tmpf $MENDER_IMGPATH:/etc/hosts && rm tmpf
```

Then you can check the contents of your 'etc/hosts' file by

```bash
mender-artifact cat $MENDER_IMGPATH:/etc/hosts
```

You should see output similar to the following:

> ```
> 192.168.10.1 docker.mender.io s3.docker.mender.io
> ```

### Set a static device IP address and subnet

This section assumes you use a static IP setup. If your device uses a DHCP setup, this section can be skipped.
In this section, we assume that `$IP_OF_MENDER_CLIENT` is
the IP address you assign to your device.

!!! If you are using `bash`, you can set a variable before running the command below, for example `IP_OF_MENDER_CLIENT="192.168.10.2"`.

Run the command below to fill the `systemd`
networking configuration files of the rootfs partitions:

```bash
echo -n "\
[Match]
Name=eth0

[Network]
Address=$IP_OF_MENDER_CLIENT
Gateway=$IP_OF_MENDER_SERVER_FROM_DEVICE
" | mender-artifact cp $MENDER_IMGPATH:/etc/systemd/network/eth.network
```

! If you have a static IP address setup for several devices, you need several disk images so each get different IP addresses.

## Wifi connectivity

The raspberrypi demo image comes with Wifi connectivity enabled by default, thus the only thing needed in order for your device to connect to your network is setting the correct `<ssid>` and `<password>` in the `wpa_supplicant-nl80211@wlan0.conf` file on your device. First set your `<password>` and `<ssid>` path as shell variables:

```bash
NW_SSID=<ssid>
NW_PASSWORD=<password>
```

And then running:

```bash
mender-artifact cat "$MENDER_IMGPATH":/etc/wpa_supplicant/wpa_supplicant-nl80211-wlan0.conf | sed "s#psk=\"password\"#psk=\"$NW_PASSWORD\"#" | sed "s#ssid=\"ssid\"#ssid=\"$NW_SSID\"#" > tmpf; mender-artifact cp tmpf "$MENDER_IMGPATH":/etc/wpa_supplicant/wpa_supplicant-nl80211-wlan0.conf && rm tmpf
```

should have your wpa configuration set up correctly on start up.

## Write the disk image to the SD card

Please see [Write the disk image to the SD card](../../../artifacts/provisioning-a-new-device#write-the-disk-image-to-the-sd-card)
for steps how to provision the device disk using the `*.sdimg`
image you downloaded and modified above.

If you have several devices, please write the disk image to all their SD cards.

## Boot the device

! Make sure that the Mender server is running as described in [Install a Mender demo server](../create-a-test-environment) and that the device can reach it on the IP address you configured above (`$IP_OF_MENDER_SERVER_FROM_DEVICE`). You might need to set a static IP address where the Mender server runs and disable any firewalls.

First, insert the SD card you just provisioned into the device.

For the **BeagleBone Black only** (N/A to Raspberry Pi 3): Before powering on the BeagleBone Black, press the
_S2 button_ (as shown below) and keep the button pressed for about 5 seconds while booting (power is connected). This will make the BeagleBone
Black boot from the SD card instead of internal storage.

![Booting BeagleBone Black from SD card](beaglebone_black_sdboot.png)

!! If the BeagleBone Black boots from internal storage, the rollback mechanism of Mender will not work properly. However, the device will still boot so this condition is hard to detect.

!!! There is no need to press the S2 button when rebooting, just when power is lost and it is powered on again.

Now **connect the device to power**.

## Run Mender setup

Once the device has booted, log in as root. If it is not possible to log in
directly as root, you need to log in as a normal user first. On Raspbian this
user is "pi", and the password is "raspberry". Then switch to a root account
using this command:

```bash
sudo -i
```

On the Yocto based Beaglebone Black image, you can log in directly as root with
no password, so the above command is not needed.

Once you have logged in as root, run the Mender setup command, like this:

```bash
mender setup
```

This will start the text based interactive setup of the Mender client. Below you
can see a typical session, with example answers given throughout.

<!-- Why "html" in the below block? "text" would be the most correct, but it has
bugs and inserts unwanted spaces in the beginning -->
```html
Mender Client Setup
===================

Setting up the Mender client: The client will regularly poll the server to check
for updates and report its inventory data.
Get started by first configuring the device type and settings for communicating
with the server.


The device type property is used to determine which Mender Artifact are
compatible with this device.
Enter a name for the device type (e.g. raspberrypi3-raspbian): [raspberrypi]

Are you connecting this device to hosted.mender.io? [Y/n] n

Demo mode uses short poll intervals and assumes the default demo server setup.
(Recommended for testing.)
Do you want to run the client in demo mode? [Y/n] y

Set the IP of the Mender Server: [127.0.0.1] 1.2.3.4
Mender setup successfully.
```

In the question about "IP of the Mender Server", use the value of
`$IP_OF_MENDER_SERVER_FROM_DEVICE` that you defined earlier. It is not possible
to use the variable itself in the setup, you have to type the IP value. In the
example above, the value is `1.2.3.4`, but it will be different in your setup.

After the setup has been done, restart the client with one of the commands
below.

For Raspbian:

```bash
systemctl restart mender-client
```

For Yocto Project images:

```bash
systemctl restart mender
```

## See the device in the Mender UI

If you refresh the Mender server UI (by default found at [https://localhost/](https://localhost/?target=_blank)),
you should see one or more devices pending authorization. If you do not see your device listed in the UI, please review [troubleshooting steps.](../../../troubleshooting/device-runtime#mender-server-connection-issues)

Once you **authorize** these devices, Mender will auto-discover
inventory about the devices, including the device type (e.g. beaglebone)
and the IP addresses, as shown in the example with a BeagleBone Black below.
Which information is collected about devices is fully configurable; see the documentation on [Identity](../../../client-configuration/identity) and [Inventory](../../../client-configuration/inventory) for more information.

![Mender UI - Device information for BeagleBone Black](device_information_bbb.png)

!!! If your device does not show up for authorization in the UI, you need to diagnose what went wrong. Most commonly this is due to problems with the network. You can test if your workstation can reach the device by trying to ping it, e.g. with `ping 192.168.10.2` (replace with the IP address of your device). If you can reach the device, you can ssh into it, e.g. `ssh root@192.168.10.2`. Otherwise, if you have a serial cable, you can log in to the device to diagnose. The `root` user is present and has an empty password in this test image. Check the log output from Mender with `journalctl -u mender-client` or `journalctl -u mender`. If you get stuck, please feel free to reach out on the [Mender Hub discussion forum](https://hub.mender.io/)!

## Prepare the Mender Artifact to update to

! Please make sure to set shell variables that expand correctly with `$IP_OF_MENDER_SERVER_FROM_DEVICE` (always) and `$IP_OF_MENDER_CLIENT` (if you are using static IP addressing) or edit the commands below accordingly.

In order to deploy an update, we need a Mender Artifact to update to.
A Mender Artifact is a file format that includes metadata like the
checksum and name, as well as the actual root file system that is
deployed. See [Mender Artifacts](../../../architecture/mender-artifacts) for
a complete description of this format.

Locate the `release_1` demo Artifact file (`.mender`) for your device that you [downloaded earlier](../../../downloads#disk-images).

We carry out exactly the same configuration steps for the Mender Artifact as we did for the disk image above:

! Please make sure to set a shell variable that expands the `.mender` file correctly with `$MENDER_FILE_IMGPATH` or edit the commands below accordingly.

```bash
mender-artifact cat $MENDER_FILE_IMGPATH:/etc/hosts | sed "\$a ${IP_OF_MENDER_SERVER_FROM_DEVICE} docker.mender.io s3.docker.mender.io" > tmpf; mender-artifact cp tmpf $MENDER_FILE_IMGPATH:/etc/hosts && rm tmpf
```

Then check the contents of the file

```bash
mender-artifact cat $MENDER_FILE_IMGPATH:/etc/hosts
```

You should see output similar to the following:

> ```
> 192.168.10.1 docker.mender.io s3.docker.mender.io
> ```

Finally, **only if you are using static IP addressing**, you need to set the
device IP address, as shown below (otherwise skip this step). Please note that the same
constraints as described in [Set a static device IP address and subnet](#set-a-static-device-ip-address-and-subnet)
for the disk image apply here.

```bash
echo -n "\
[Match]
Name=eth0

[Network]
Address=$IP_OF_MENDER_CLIENT
Gateway=$IP_OF_MENDER_SERVER_FROM_DEVICE
" | mender-artifact cp $MENDER_FILE_IMGPATH:/etc/systemd/network/eth.network
```

!!! The Mender client will roll back the deployment if it is not able to report the final update status to the server when it boots from the updated partition. This helps ensure that you can always deploy a new update to your device, even when fatal conditions like network misconfiguration occur.

You can also make any other modifications you wish in this image
prior to deploying it.

!!! NOTE if you are running the raspberrypi pi demo image, with Wifi enabled and setup as per [Wifi connectivity](#Wifi-connectivity), the network id and password will have to be set in the same way as done for the sdimg.

## Upload the artifact to the server

Before we can deploy the Artifact we prepared above, it needs
to be uploaded to the server.

Go to the Mender server UI, click the **Releases** tab and upload this Artifact.

## Deploy the Artifact

Now that we have the device connected and the Artifact
uploaded to the server, all that remains is to go to the
**Deployments** tab and click **Create a deployment**.

Select the Artifact you just uploaded and **All devices**, then
**Create deployment**.

!!! If you deploy across several device types (e.g. `beaglebone` and `raspberrypi`), the Mender server will skip these if no compatible artifact is available. This condition is indicated by the _noartifact_ status in the deployment report. Mender does this to avoid deployments of incompatible rootfs images. However, if you have Artifacts for these other device types, identified by the same Artifact name, then Mender will deploy to all the devices there are compatible Artifacts for.

## See the progress of the deployment

As the deployment progresses, you can click on it to view more details about the current status across all devices.
In the example below, we can see that a BeagleBone is installing the update.

![Mender UI - Deployment progress - BeagleBone Black](deployment_report_bbb.png)

Once the deployment completes, you should see its report in _Past deployments_.

**Congratulations!** You have used the Mender server to deploy your first physical device update!

## Deploy another update

In order to deploy another update, we need to create another Artifact
with a different Artifact Name (than the one already installed at the devices).
This is because Mender _skips a deployment_ for a device if it detects that
the Artifact is already installed, in order to avoid unnecessary deployments.

To change the name of our existing Artifact, we can simply use `modify` and the `-n` option
of the `mender-artifact` tool, first making a copy of the original. To do this,
run these two commands (adjust the Artifact file name accordingly):

```bash
cp $MENDER_FILE_IMGPATH myupdate_release_2.mender
mender-artifact modify myupdate_release_2.mender -n release-2
```

!!! Using`mender-artifact modify`, you can easily modify several configuration settings in existing disk image (`.sdimg`) and Mender Artifact (`.mender`) files, such as the server URI and certificate. See `mender-artifact help modify` for more options.

! Currently the `mender-artifact modify` command only supports modifying ext4 payloads.

Upload this modified Artifact file to your Mender server and deploy it to your device.
You should see that the Artifact Name has changed after the deployment.
Now that you have two Mender Artifact files that are configured for your
network with different names, you can deploy updates back and forth between them.

## Integrate Mender with your board

Now that you have seen how Mender works with a reference board, you might be wondering what it would take to port it to your own board.

To get support for robust system updates with rollback, Mender must be [integrated with production boards](../../../devices).

On the other hand, if you only need support for application updates (not full system updates), no board integration is required. In this case you can install Mender on an existing device and OS by following the documentation on [installing the Mender client](../../../client-configuration/installing).
