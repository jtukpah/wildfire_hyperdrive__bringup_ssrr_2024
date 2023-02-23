# Bootstrapping a New PC

Installing the Hyper-Drive compute system requires careful configuration to ensure agreement between kernel versions and available drivers.

These instruction presume the PC being prepared is an [Intel NUC 11 Enthusiast NUC11PHKi7C](https://www.intel.com/content/www/us/en/products/sku/202783/intel-nuc-11-enthusiast-kit-nuc11phki7c/specifications.html)

### Install Ubuntu 18

In the system BIOS disable secure boot.

Create a bootable USB to install Ubuntu from following these [instructions](https://www.linuxtechi.com/ubuntu-18-04-lts-desktop-installation-guide-screenshots/). By default the NUC comes with Windows 10 installed

If the system fails to boot after the Ubuntu install, force it into recovery mode by pressing the `esc` key during the boot phase. Enable networking from the root menu, and then run the following commands.

```
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt-get update
sudo apt-get install nvidia-384
```

The integrated RTX2060 GPU does not work well with the Intel recommended drivers. This repository appears to work consistently with the integrated GPU (Intel Iris Xe). At this point on the USB-C ports appear to be working.

### Downgrade Kernel

The IMEC HSI drivers will only compile with Kernel 4.15 - the original kernel for Ubuntu 18.04.

```
sudo apt-get autoremove --purge 'linux-image-5.4.0-*-generic' 'linux-image-unsigned-5.4.0-*-generic' 'linux-modules-5.4.0-*-generic' 'linux-hwe-5.4-headers-5.4.0-*' linux-generic-hwe-18.04 linux-image-generic-hwe-18.04 linux-headers-generic-hwe-18.04
# allow removal of running 5.4 kernel in the ncurses blue prompt - answer 'No'

sudo apt-get install --install-recommends linux-generic 
sudo apt-get install --install-recommends linux-image-generic linux-headers-generic
```

Next, append the following two lines to your grub file at `/etc/default/grub` and then run `sudo update-grub` to allow the system to pick up the changes.

```
GRUB_SAVEDEFAULT=true
GRUB_DEFAULT=saved
```

The next time the PC reboots, select "Advanced Options for Ubuntu" in the Grub menu, followed by the first option for the kernel version 4.15. This setting will now be loaded by default every time.

### Fix Network Devices

After downgrading the kernel, the ethernet and wifi hardware will become detached and unavailable for connections. Using a USB ethernet adapter or USB WiFi dongle, reinstall a compatible WiFi driver.

```
sudo add-apt-repository ppa:canonical-hwe-team/backport-iwlwifi
sudo apt-get update
sudo apt-get install backport-iwlwifi-dkms 
```

### Install ROS Melodic

In general, the instructions here are sufficient, but they are repeated for conciseness here.

```
sudo sh -c 'echo "deb http://packages.ros.org/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt install curl # if you haven't already installed curl
curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
sudo apt update
sudo apt install ros-melodic-desktop-full
echo "source /opt/ros/melodic/setup.bash" >> ~/.bashrc
source ~/.bashrc
sudo apt install python-rosdep python-rosinstall python-rosinstall-generator python-wstool build-essential
sudo apt install python-rosdep
sudo rosdep init
rosdep update
sudo apt install python-pip
pip install -U catkin_tools
sudo apt-get install ros-melodic-catkin python-catkin-tools
```

### Create Catkin WS

Assuming the ROS install was successful, create an empty catkin repository and verify the workspace builds successfully.

```
mkdir -p catkin_ws/src
cd catkin_ws/src
catkin_init_workspace
cd ..
catkin build
```

### Install Packages

To complete this step, this system needs an SSH key. Generate one using `ssh-keygen` and associate it with your GitHub profile. With this authentication setup, grab the required repos. *Note you will need to follow the instructions for installation in each of the respective repos.*
```
cd catkin_ws/src
git clone git@github.com:RIVeR-Lab/hyper_drive.git
git clone git@github.com:RIVeR-Lab/hyperdrive_bringup.git
git clone git@github.com:RIVeR-Lab/spectrometer_drivers.git
git clone https://github.com/eric-wieser/ros_numpy.git
catkin build
```

### Install the HSI Camera Drivers

```
cd catkin_ws/src/hyperdrive_bringup/resources
sudo dpkg -i hsi_mosaic-linux-x86_64-installer.deb 
cd /opt/imec/hsi-mosaic/resources/installers
tar xzf XIMEA_Linux_SP.tgz
cd package/
sudo apt-get update && sudo apt-get install build-essential linux-headers-"$(uname -r)"
sudo ./install
sudo dpkg -i eBUS_SDK_Ubuntu-x86_64-6.1.1-5002.deb 
```

### Install QT15
The following kind soul hosts the QT15.5.2 library as a ppa (personal package archive). Credit to [Stephan Binner](https://launchpad.net/~beineri/+archive/ubuntu/opt-qt-5.15.2-bionic).
```
sudo add-apt-repository ppa:beineri/opt-qt-5.15.2-bionic
sudo apt update
sudo apt-get install qt515base
sudo apt-get install qt515svg
```

### Update the .bashrc file
Append the following lines to the end of your `.bashrc ` file.
```
export PATH="$PATH:/opt/imec/hsi-mosaic/bin:/opt/pleora/ebus_sdk/Ubuntu-x86_64/bin"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/opt/imec/hsi-mosaic/bin:/opt/pleora/ebus_sdk/Ubuntu-x86_64/lib"
# export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/opt/qt515/lib/:/opt/imec/hsi-mosaic/bin:/opt/pleora/ebus_sdk/Ubuntu-x86_64/lib" source /opt/ros/melodic/setup.bash
source /opt/qt515/bin/qt515-env.sh

alias cs='cd ~/catkin_ws'
alias cb='cd ~/catkin_ws/ && catkin build'
alias cbs='cd ~/catkin_ws/ && catkin build && source ~/catkin_ws/devel/setup.bash'
alias bringup_camera='source ~/catkin_ws/devel/setup.bash && roslaunch imec_driver master.launch use_imec:=true use_ximea:=true gui:=true'
```

### Things TODO
[] Get HDMI and Display Port outputs working from GPU
[] Keyboard and mouse are occasionally very slow
[] Ethernet port is not working (use USB adapter)
[] Add Alvium Camera