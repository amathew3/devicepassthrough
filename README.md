# Device Passthrough

Device passthrough for TDX guests requires updates to the host kernel
and QEMU. The patch sets below track the bleeding edge state of
pre-release Kernel and QEMU development branches. They are provided here
for preview and test purposes only, and may be update frequently. Any
feedback or issues should be reported as a reply to the latest upstream
posting of the given patch. See the linux-coco, linux-kvm, and qemu-devel
archives on lore.kernel.org for the most recent public posting of these patches. Note that
there some temporary workarounds and shortcuts included while formal
replacements are in development.

Wait for these patches to be accepted by their respective upstream
projects, Next, wait for new releases of those upstream project versions to be picked up
by your chosen Linux distribution provider before using them for any production
use case.

## Repo Layout

* `tdx-kvm` - Contains an mbox file with host side TDX patches for KVM. This can be applied using:
  ```
  mkdir ~/nvidia_setup
  git config --global user.email youremail@yourdomain.com
  git config --global user.name "Your Name"
  cd ~/nvidia_setup
  git clone https://github.com/canonical/tdx.git
  git clone -b device-passthrough https://github.com/intel/tdx-linux.git
  git clone -b kvm-coco-queue-20240512 https://git.kernel.org/pub/scm/linux/kernel/git/vishal/kvm.git
  cd ~/nvidia_setup/kvm
  cp -rf ../tdx-linux/tdx-kvm .
  git am --empty=drop tdx-kvm/tdx_kvm_baseline_<sha>.mbox
  ```
  The baseline (**as noted in the filename above**) for these patches is the
`kvm-coco-queue` branch in the [KVM
repo](https://git.kernel.org/pub/scm/virt/kvm/kvm.git/). Since this is a
rebasing branch, the commit is not guaranteed to be present in kvm.git. A
snapshot of the older version of kvm-coco-queue can be found, for
example,
[here](https://git.kernel.org/pub/scm/linux/kernel/git/vishal/kvm.git/log/?h=kvm-coco-queue-20240807).

* `tdx-qemu` - Contains an mbox file with TDX patches for QEMU. This can be applied using:
  ```
  git clone https://gitlab.com/qemu-project/qemu
  cd qemu
  git checkout -b rc0 v9.1.0-rc0
  cp -rf ../tdx-qemu .
  git am --empty=drop tdx-qemu/tdx_qemu_baseline_<sha>.mbox
  ```
  The baseline for these should be found in [qemu.git](https://git.qemu.org/git/qemu.git).

* `tdx-edk2` - The EDK2 enabling we need so far is already in an upstream tag. This simply contains a file with the stable tag name. from the [EDK2 repo](https://github.com/tianocore/edk2.git)

## Building the components
### Installing the packages needed for kernel build.
```
  sudo apt update
  sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev debhelper-compat=12 meson ninja-build \ 
  libglib2.0-dev python3-pip nasm iasl
  cd ~/nvidia_setup/kvm
  make menuconfig
```
+ Save the default config file.
+ Check the below config values.
* `tdx-kvm`:
  * Config options:
    ```
    Enables:
    CONFIG_INTEL_TDX_HOST
    CONFIG_KVM
    CONFIG_KVM_INTEL
    CONFIG_TDX_GUEST_DRIVER
    CONFIG_HYPERV

    Disables:
    CONFIG_KEXEC
    CONFIG_CRASH_DUMP
    ```
    
  * Build and install the kernel on the host machine. Add module options:
    ```
    make -j$(nproc)
    make modules -j$(nproc)
    sudo make modules_install 
    sudo make install
    sudo sh -c “echo options kvm_intel tdx=on > /etc/modprobe.d/tdx.conf”
    sudo grubby --update-kernel=ALL --args="console=ttyS0,115200 kvm_intel.tdx=on nohibernate"
    sudo update-grub
    ```

* `tdx-qemu`: 
  * Config options:
    ```
    cd ~/nvidia_setup/qemu
    ./configure --enable-slirp --enable-kvm --target-list=x86_64-softmmu
    make -j$(nproc)
    sudo make install
    ```

* `tdx-edk2`:
  * Clone the [EDK2 repo](https://github.com/tianocore/edk2) and checkout the tag as noted in this repo.
  * Build the OVMF image:
    ```
    cd ~/nvidia_setup
    git clone -b edk2-stable202405 https://github.com/tianocore/edk2
    cd edk2
    git submodule update –init
    ```
+ Copy the below content to a file `build_ovmf.sh`
    ```
    #!/bin/bash
    rm -rf Build
    make -C BaseTools
    . edksetup.sh
    cat <<-EOF > Conf/target.txt
    	ACTIVE_PLATFORM = OvmfPkg/OvmfPkgX64.dsc
    	TARGET = DEBUG
    	TARGET_ARCH = X64
    	TOOL_CHAIN_CONF = Conf/tools_def.txt
    	TOOL_CHAIN_TAG = GCC6
    	BUILD_RULE_CONF = Conf/build_rule.txt
    	MAX_CONCURRENT_THREAD_NUMBER = $(nproc)
    EOF
    build clean
    build

    if [ ! -f Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd ]; then
    	echo "Build failed, OVMF.fd not found"
    	exit 1
    fi

    cp Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd ./OVMF.fd
    ```
+ Run the script.
  ```
   chmod +x ./build_ovmf.sh
   ./build_ovmf.sh
   sudo reboot
  ```
### Libvirt cofiguration for using the newly built qemu.

+ Apply the following settings to the file /etc/libvirt/qemu.conf
```
 user = <your_user_name>
 group = <your_group>
 dynamic_ownership = 0
 security_driver = "none"
```
+ Restart the libvirtd service.
```
  systemctl restart libvirtd
```
## Enabling CC mode on GPU.
```
  cd ~/nvidia_setup/nvtrust 
  git submodule update --init 
  cd host_tools/python

  sudo python3 ./nvidia_gpu_tools.py --gpu-name=H100 --query-cc-mode
  sudo python3 ./nvidia_gpu_tools.py --gpu-name=H100 --set-cc-mode=on --reset-after-cc-mode-switch
  lspci -d 10de: -nn
  sudo modprobe vfio
  sudo modprobe vfio_pci
  sudo sh -c "echo 10de 2331 > /sys/bus/pci/drivers/vfio-pci/new_id"
```
* Please use device number returned in `lspci -d 10de: -nn` for  setting the new_id.


* Follow the steps mentioned in [wiki](https://github.com/canonical/tdx/blob/noble-24.04/README.md) to prepare a guest image.
```
  cd ~/nvidia_setup/tdx/guest_tools/images
  sudo ./create-td-image.sh
```

* Additional notes on Host and Guest setup and booting can be found in the [wiki](https://github.com/intel/tdx-linux/wiki/Instruction-to-set-up-TDX-host-and-guest). One thing not mentioned on the wiki: make sure TDX guest has **"clearcpuid=mtrr"** in its kernel command line.


## Specific notes for device passthrough to TD
* `Host and Guest Kernel`
  * Prepare host, guest kernel and qemu according to the above info.
  * For guest kernel to support SPDM session establishment make sure the following configuration options are enabled.
    ```
    CONFIG_CRYPTO_ECC=y
    CONFIG_CRYPTO_ECDH=y
    CONFIG_CRYPTO_ECDSA=y
    CONFIG_CRYPTO_ECRDSA=y
    ```
* `Boot TD with GPU passthrough`
  * To passthrough the GPU card to TD, say 38:00.0, run the `run_td.sh` script with `-d` as an argument.
    ```
    cd ~/nvidia_setup/tdx/guest_tools
    sudo ./run_td.sh -d
    ```
