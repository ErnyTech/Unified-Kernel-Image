# Unified Kernel Image
<p align="center">
  <img src="https://raw.githubusercontent.com/ErnyTech/Unified-Kernel-Image/master/Uefi_logo.svg" />
</p>

### What are the Unified Kernel Images?
Unified Kernel Images are EFI executables that contain the following components:
  - An EFI stub loader
  - A kernel image (usually linux but technically it can also be extended to other kernels)
  - The kernel command line
  - An initramfs image
  - The os release file to describe the image (optional but recommended)
  - A splash image in BMP format that will be displayed after starting the EFI executable (optional)
  
With all these components the image is able to load the kernel and initramfs without the need for further configuration (for example configuring the entries in the bootloader to specify the kernel cmdline).

### Advantages of Unified Kernel Image
  - No need for bootloader configuration
  - One file to boot Linux
  - It doesn't need a bootloader, just create an entry in the EFI boot manager
  - Simple to sign for UEFI Secure Boot, thanks to this it is possible to ensure that an unauthorized person does not interfere with the kernel boot process
  - It allows you to set up a splash image at kernel boot in a very simple way
  
### Disadvantages of the Unified kernel image
  - Currently the distro tools for creating initramfs and kernel images often do not support generating the Unified Kernel Images
  - You have to regenerate the image every time you want to change the kernel cmdline, splash image or os release file (not very relevant, it only takes a few seconds, grub-mkconfig is sometimes slower)
  - A Unified Kernel Image can only be used on PCs with UEFI support (obviously)
  
### How to configure bootloaders to boot Unified Kernel Images
  - ### systemd-boot:
	No configuration required, just copy the images to ${esp}/EFI/Linux (where ${esp} is the path of your EFI partition).
	
  - ### rEFInd:  
    No configuration needed, rEFInd will automatically find the Unified Kernel Images and will be able to run it.
    However rEFInd is currently unable to read the .osrel section (which contains the OS release file) so the image name will not appear (no issues to boot)
    
  - ### GRUB:  
    You need to configure the bootloader as follows (consider using another more comfortable bootloader than GRUB):
    ```
    menuentry "Arch Linux" {
		insmod fat
		insmod chain
		search --no-floppy --set=root --fs-uuid <FILESYSTEM_UUID>
		chainloader /EFI/Linux/Arch-linux.efi
    }
    ```
    
### How to generate Unified Kernel Images
Just use objcopy to create an EFI PE executable thanks to EFI STUB:

```
# Set os release file path of your distro, this is the default path in ArchLinux
OS_RELEASE=/usr/lib/os-release

# Create file for kernel command line, provide at least the root parameter
echo "root=UUID=cadd8a20-fe22-45fd-b550-a7471e3065fc rw" > kernel-command-line.txt

# Set the splash image, /sys/firmware/acpi/bgrt/image is the vendor logo taken from ACPI in BMP image format
SPLASH="/sys/firmware/acpi/bgrt/image"

# Set the kernel image path (enter the path for your operating system)
VMLINUZ="/vmlinuz-linux"

# Set the initrd image path (enter the path for your operating system)
INITRD="/initramfs-linux.img"

# Set the EFI STUB path, the default value should be good for the most users 
STUB="/usr/lib/systemd/boot/efi/linuxx64.efi.stub"

objcopy \
    --add-section .osrel="$OS_RELEASE" --change-section-vma .osrel=0x20000 \
    --add-section .cmdline="kernel-command-line.txt" --change-section-vma .cmdline=0x30000 \
    --add-section .splash="$SPLASH" --change-section-vma .splash=0x40000 \
    --add-section .linux="$VMLINUZ" --change-section-vma .linux=0x2000000 \
    --add-section .initrd="$INITRD" --change-section-vma .initrd=0x3000000 \
    "$STUB" "linux.efi"
```

This command will generate a Unified Kernel Image ready to be started named "linux.efi"

# How to automate the procedure on Arch Linux
Since having to write in each kernel update the command above is quite boring I thought how to automate the procedure.

The following instructions are only compatible if the following prerequisites are met:
  - A distribution based on ArchLinux using pacman as the package manager
  - mkinitcpio installed and operational
  - No hooks in /etc/pacman.d/hooks affecting mkinitcpio or kernels 
  - systemd-boot installed, I only tested with this
  
### 1) Install mkunified, this is the script that will take care of generating the images
```
pacaur -S mkunified-git
```
mkunified is the script that will generate the unified kernel images using the kernel images installed in `/etc/mkunified.d/images`, the general configuration file `/etc/mkunified.conf` and the various presets present in `/etc/mkunified.d`.

The mkunified-git package will install hooks in `/usr/share/libalpm/hooks` and `/usr/share/libalpm/scripts`, these hooks will generate preset files for you using the template in `/usr/share/mkunified/hook.preset` and will install the kernel image in `/etc/mkunified.d/images` each time a new kernel package is installed/updated.

Using mkunified, in most cases, you will only have to configure the file `/etc/mkunified.conf`, all the rest will be done automatically by libalpm hooks if you install a packaged kernel in the standard way used on Archlinux (if you need to install your own image you will have to copy it to `/etc/mkunified.d/images` and create the preset based on `/usr/share/mkunified/hook.preset`).

### 2) Configure mkunified
The mkunified configuration is done by editing the `/etc/mkunified.conf` file, by default the configuration is almost ready to work in most cases.

The essential things to change are as follows:
  - EFI, you need to specify your EFI partition by default it is configured as follows: `EFI="/boot"`
  - CMDLINE, this is the kernel cmdline it is absolutely necessary to change this configuration! You must at least enter the root parameter by entering the UUID of your root partition, Use `sudo blkid` to get the UUID
  - SPLASH_ENABLE, This will enable splash screen support by default is enabled with the following configuration: `SPLASH_ENABLE=true`
  - SPLASH, specifies where to get the splash screen image (must be an absolute path pointing to an image in BMP format), by default it gets the vendor image from the ACPI with the following configuration: `SPLASH="/sys/firmware/acpi/bgrt/image"`
  - Pre-initrd/microcode, to install a microcode to load in early boot it is necessary to add a pre-initrd image containing the microcode, in the configuration there is already a preconfiguration for Intel and AMD just remove the comment (#) corresponding to the ideal preconfiguration for you to enable the loading of the microcode
  
### 3) Generate the Unified Kernel Image
Just reinstall the kernel package you want to use, for example:
```
sudo pacman -S linux
```
This will take care of starting the hooks, generating the presets and generating the images for the first time.

Then if you want to regenerate the images (for example after changing the configuration) just run the following command:
```
# To regenerate all images
sudo mkunified -P 

# To regenerate specific preset (the name are the same as mkinitcpio)
sudo mkunified -p <preset>
```

### 4) (Optional) Remove the generation of classic images
You need to create an empty hook that will override the classic mkinitcpio hooks:
```
sudo touch /etc/pacman.d/hooks/60-mkinitcpio-remove.hook

sudo touch /etc/pacman.d/hooks/90-mkinitcpio-install.hook
```

You can also delete /etc/mkinitcpio.d as it will no longer be used:
```
sudo rm -r /etc/mkinitcpio.d
```

Later you can also delete the vmlinuz and initramfs images from your boot partition, I no longer need it because you will be using the unified kernel images

