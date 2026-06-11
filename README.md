# coreboot-x61

This repo is a simple guide to compiling and flashing coreboot to your Thinkpad X61.
I take no responsability if you brick your laptop while attempting to flash coreboot.
I will use a raspberry pi pico flashed with the serprog firmware from libreboot as a programmer.
You can use whatever programmer you want, just make sure it is good and does not cause problems for power (**OUTPUT VOLTAGE NEEDS TO BE 3.3V**)

**Always keep a backup of your BIOS!!!!!!**

## Setup
We will assume you are using Ubuntu/Debian-based distros
### Update system
```sh
sudo apt update
sudo apt upgrade -y
```

### Install dependencies
```sh
sudo apt-get install -y bison build-essential curl flex git gnat libncurses-dev libssl-dev zlib1g-dev pkgconf flashrom python-is-python3
```

### Clone repo and submodules
```sh
git clone --recursive https://review.coreboot.org/coreboot ~/coreboot
cd ~/coreboot
```

### Extracting the blobs
```sh
make -c util/ifdtool
cd ..
sudo flashrom -p serprog:dev=/dev/ttyACM0,spispeed=16M -r dump1.bin
sudo flashrom -p serprog:dev=/dev/ttyACM0,spispeed=16M -r dump2.bin
diff dump1.bin dump2.bin
```
You will probably get an error about multiple chip definitions. If that happens, you will need to read the model on the chip and specify the chip model to the flashrom command. Remember the model as you will need it again when flashing.

Make sure the diff commands does not output anything. If it says that files differ, disconnect the programmer from pc, detach clip from chip and reattach it again.

If both dumps are identical then you can continue:
```sh
~/coreboot/util/ifdtool/ifdtool -x dump1.bin
```
This command will extract 5 blobs. Remove flashregion_1_bios.bin, as we will not need it anymore and flashregion_4 as it will most certainly be empty.
Don't forget to keep one dump in case you want to revert to the original bios.

### Patch libgfxinit
At the moment when writing this, libgfxinit support is still kindof spotty. You will be able to use only tty in linux. Don't know about Windows, *BSDs or what other OSes do people use on the x61. If you want to use vgabios, skip this portion.
```sh
cd 3rdparty/libgfxinit
git fetch https://review.sourcearcade.org/libgfxinit refs/changes/99/499/5 && git reset --hard FETCH_HEAD
cp repo/gm965_audio_fix.patch ~/coreboot/3rdparty/libgfxinit
patch -p1 < gm965_audio_fix.patch
cd ../..
```

### Setup the toolchain
```sh
make crossgcc-i386 CPUS=$(nproc)
```

### Configure
If you use libgfxinit, use the normal config.
If you use vgabios, use the vgabios config.
**For both configs, you will need to edit the paths to your ifd, gbe and me blobs extracted from your original bios!**
After renaming your chosen config to .config, just run
```sh
make nconfig
```
to populate the rest of the config

### Compiling
The added flag for the make command will make sure that the libgfxinit checkout we did earlier will remain
```sh
make UPDATED_SUBMODULES=1 -j$(nproc)
```

### Flashing
```sh
sudo flashrom -p serprog:dev=/dev/ttyACM0,spispeed=16M -w build/coreboot.rom
```

# Credits
I give credits to:

    - the coreboot team for creating coreboot
    - Arthur Heymans (u/avph on reddit) for porting the x61 to coreboot and libgfxinit