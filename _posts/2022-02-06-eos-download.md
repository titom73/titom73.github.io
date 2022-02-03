---
title: eos-download introduction
date: 2022-02-01
categories: [Arista]
tags: [arista, python]     # TAG names should always be lowercase
---
In this post, we will see how to use [eos-download](https://github.com/titom73/arista-downloader) script to automate your EVE-NG instance with new [Arista EOS](https://www.arista.com/en/products/eos#:~:text=Arista%20EOS%20is%20a%20fully,across%20the%20Arista%20switching%20family.) version for your labs.

One of the pain point when you want to test a design is to prepare VMs with correct software version and do the provisioning. So __eos-download__ tries to help provisioning software in EVE-NG to let you start quickly to work on your network things.

## Install script

Script installation is based on standard python packaging mechanism and can be installed with either setup_tools or pip. All the code is python `>=3.8` compliant.

```bash
$ pip3 install git+https://github.com/titom73/arista-downloader
```

From here, your script is installed in your `$PATH` and can be called from your shell. Even if you can install it on any device using python, we will do that on EVE-NG server as our goal here is to provision VM on EVE-NG, but you can also use that script for general purpose.

## Configuration

There is no configuration file to create and everything is done by CLI argument. Besides that, some options use environment variables to reduce size of the CLI and make process easier. it is the case for token authentication.

Talking about authentication, you need to get a token to access Arista download site. You can generate this token in your [account page](https://www.arista.com/en/users/profile).

## Usage

### Generic usage

`eos-download` script provides only few options:

```bash
eos-download -h
usage: eos-download [-h] [--token TOKEN]
    --version VERSION
    [--image IMAGE]
    [--destination DESTINATION]
    [--eve]
    [--noztp]
    [--verbose VERBOSE]

EOS downloader script.

optional arguments:
  -h, --help            show this help message and exit
  --token TOKEN         arista.com user API key - can use ENV:ARISTA_TOKEN
  --image IMAGE         Type of EOS image required
  --version VERSION     EOS version to download from website
  --destination DESTINATION
                        Path where to save EOS package downloaded
  --eve                 Option to install EOS package to EVE-NG
  --noztp               Option to deactivate ZTP when used with EVE-NG
  --verbose VERBOSE     Script verbosity
```

As explained earlier, this script can be used for general purposes and gives you option to download almost any flavor of EOS (`--image`) from [Arista website](https://www.arista.com/en/support/software-download):

- International version (`--image INT`)
- 64 bits version (`--image 64`)
- 2GB flash platform (`--image 2GB`)
- 2GB running International (`--image 2GB-INT`)
- Virtual EOS image (`--image vEOS`)
- Virtual Lab EOS (`--image vEOS-lab`)
- Virtual Lab EOS running 64B (`--image vEOS64-lab`)
- Docker version of EOS (`--image cEOS`)
- Docker version of EOS running in 64 bits (`--image cEOS64`)

In our case, we will just download `vEOS-LAB` flavor

For the version, we will download EOS in version `4.25.1F`. So the easiest CLI will be:

```bash
root@eve-ng:~/demo# eos-download \
    --token fake-token-here \
    --image vEOS-lab \
    --version 4.25.1F

2022-02-03 12:52:34.135 | INFO     | eos_downloader:authenticate:360 - Authenticated on arista.com
2022-02-03 12:52:35.023 | INFO     | eos_downloader:_parse_xml:122 - Found vEOS-lab-4.25.1F.vmdk at /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.1F/vEOS-lab/vEOS-lab-4.25.1F.vmdk
2022-02-03 12:52:35.024 | INFO     | eos_downloader:_download_file:314 - File found on arista server: /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.1F/vEOS-lab/vEOS-lab-4.25.1F.vmdk
2022-02-03 12:52:58.429 | INFO     | eos_downloader:_parse_xml:122 - Found vEOS-lab-4.25.1F.vmdk.sha512sum at /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.1F/vEOS-lab/vEOS-lab-4.25.1F.vmdk.sha512sum
2022-02-03 12:53:00.277 | WARNING  | eos_downloader:download_local:399 - Downloaded file is correct.
root@eve-ng:~/demo#
```

As you can see, script authenticate against Arista website, finds the link for download and do a sha512 comparison to validate download is not corrupted.

But file is downloaded in your current location:

```bash
root@eve-ng:~/demo# ls
vEOS-lab-4.25.1F.vmdk  vEOS-lab-4.25.1F.vmdk.sha512sum
root@eve-ng:~/demo#
```

Ok, but what if we don't want to type our token each time ? Then, just configure an environment variable automatically loaded by the script:

```bash
# Configure your token. it can also be something installed in your .bashrc / .zshrc
root@eve-ng:~/demo# export ARISTA_TOKEN=fake-token-here

# Run script
root@eve-ng:~/demo# eos-download \
    --image vEOS-lab \
    --version 4.25.1F

2022-02-03 12:52:34.135 | INFO     | eos_downloader:authenticate:360 - Authenticated on arista.com
2022-02-03 12:52:35.023 | INFO     | eos_downloader:_parse_xml:122 - Found vEOS-lab-4.25.1F.vmdk at /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.1F/vEOS-lab/vEOS-lab-4.25.1F.vmdk
2022-02-03 12:52:35.024 | INFO     | eos_downloader:_download_file:314 - File found on arista server: /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.1F/vEOS-lab/vEOS-lab-4.25.1F.vmdk
2022-02-03 12:52:58.429 | INFO     | eos_downloader:_parse_xml:122 - Found vEOS-lab-4.25.1F.vmdk.sha512sum at /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.1F/vEOS-lab/vEOS-lab-4.25.1F.vmdk.sha512sum
2022-02-03 12:53:00.277 | WARNING  | eos_downloader:download_local:399 - Downloaded file is correct.
root@eve-ng:~/demo#
```

### Download & Install in EVE-NG

To install image in EVE-NG in just one command, just use the following one:

```bash
root@eve-ng:~/demo# eos-download \
    --token fake-token-here \
    --image vEOS-lab \
    --version 4.25.7M \
    --eve

2022-02-03 12:57:02.462 | INFO     | eos_downloader:authenticate:360 - Authenticated on arista.com
2022-02-03 12:57:03.373 | INFO     | eos_downloader:_parse_xml:122 - Found vEOS-lab-4.25.7M.vmdk at /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.7M/vEOS-lab/vEOS-lab-4.25.7M.vmdk
2022-02-03 12:57:03.374 | INFO     | eos_downloader:_download_file:314 - File found on arista server: /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.7M/vEOS-lab/vEOS-lab-4.25.7M.vmdk
2022-02-03 12:57:25.747 | INFO     | eos_downloader:provision_eve:418 - Converting VMDK to QCOW2 format
2022-02-03 12:57:26.353 | INFO     | eos_downloader:provision_eve:420 - Applying unl_wrapper to fix permissions
Feb 03 12:57:26 Feb 03 12:57:26 Online Check state: Valid
```

As you can, your EOS image is automatically available in your EVE-NG interface:

![New package available](/assets/img/eos-download/eve-ng-new-image.png)

But this image still has ZTP configured. It means during the startup process, all interfaces will be active to send DHCP request and get information to find startup configuration or register to a [Cloudvision](https://www.arista.com/en/cg-cv/cv-introduction-to-cloudvision) instance.

But sometimes, we need VM to boot and get pre-configured configuration set in EVE-NG. For that, just use flag `--noztp` in the script

```bash
root@eve-ng:~/demo# eos-download \
    --token fake-token-here \
    --image vEOS-lab \
    --version 4.25.7M \
    --eve \
    --noztp

2022-02-03 13:03:03.384 | INFO     | eos_downloader:authenticate:360 - Authenticated on arista.com
2022-02-03 13:03:03.883 | INFO     | eos_downloader:_parse_xml:122 - Found vEOS-lab-4.25.7M.vmdk at /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.7M/vEOS-lab/vEOS-lab-4.25.7M.vmdk
2022-02-03 13:03:03.883 | INFO     | eos_downloader:_download_file:314 - File found on arista server: /support/download/EOS-USA/Active Releases/4.25/EOS-4.25.7M/vEOS-lab/vEOS-lab-4.25.7M.vmdk
2022-02-03 13:03:24.189 | INFO     | eos_downloader:provision_eve:418 - Converting VMDK to QCOW2 format
2022-02-03 13:03:24.829 | INFO     | eos_downloader:provision_eve:420 - Applying unl_wrapper to fix permissions
Feb 03 13:03:24 Feb 03 13:03:24 Online Check state: Valid
2022-02-03 13:03:32.779 | INFO     | eos_downloader.eos:_disable_ztp:52 - Mounting volume to disable ZTP
2022-02-03 13:03:41.252 | INFO     | eos_downloader.eos:_disable_ztp:61 - Unmounting volume in /opt/unetlab/addons/qemu/veos-lab-4.25.7m-noztp
2022-02-03 13:03:41.290 | INFO     | eos_downloader.eos:_disable_ztp:64 - Volume has been successfully unmounted at /opt/unetlab/addons/qemu/veos-lab-4.25.7m-noztp
```

And then, your image is installed with a suffix configured to `-noztp` and also available in EVE-NG interface:

![New image with no ztp](/assets/img/eos-download/eve-ng-new-image-no-ztp.png)

## Conclusion

So here you go, now you are ready to lab !

![Lab Time](/assets/img/eos-download/eve-ng-lab-time.png)

And of course, feel free to report [any issue or feature request](https://github.com/titom73/arista-downloader/issues/new) in the [repository](https://github.com/titom73/arista-downloader/)
