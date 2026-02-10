# Compiling Heimdall for Android/ARM (Termux)

This guide shows how to compile the patched Heimdall on Android devices using Termux. This creates an ARM64 binary with the SVE-2016-7930 exploit patch applied.

## Why This Matters

The original G900A-TWRP-ROM repo provides a pre-compiled x64 Heimdall binary for desktop Linux. This guide enables you to compile and use Heimdall directly on Android devices via Termux, which is useful if you don't have access to a desktop computer.

**Note:** Even with this ARM binary, you still need either:
- A rooted Android device (for USB hardware access), OR
- A desktop/laptop computer

The binary alone won't work on non-rooted Android devices due to USB permission restrictions.

## Prerequisites

- Termux (install from [F-Droid](https://f-droid.org/packages/com.termux/), NOT Google Play - the Play Store version is outdated)
- ~500MB free storage space
- Stable internet connection for downloading dependencies

## Compilation Steps

### 1. Install Dependencies

```bash
pkg update && pkg upgrade
pkg install git cmake libusb clang make pkg-config
```

### 2. Clone Heimdall Repository

```bash
cd ~/
git clone https://github.com/Benjamin-Dobell/Heimdall.git
cd Heimdall
```

### 3. Apply the SVE-2016-7930 Exploit Patch

```bash
# Download the patch from the original exploit repo
curl -O https://raw.githubusercontent.com/frederic/SVE-2016-7930/master/heimdall-increase_fileTransferSequenceMaxLength.patch

# Apply the patch
patch -p1 < heimdall-increase_fileTransferSequenceMaxLength.patch
```

This patch increases `fileTransferSequenceMaxLength` from 30 to 300, which is critical for the buffer overflow exploit to work.

### 4. Fix CMake Version Requirements

Both libpit and heimdall need their CMake minimum version updated for compatibility with modern CMake:

```bash
# Fix libpit
cd ~/Heimdall/libpit
sed -i 's/cmake_minimum_required(VERSION [0-9.]*)/cmake_minimum_required(VERSION 3.5)/' CMakeLists.txt

# Fix heimdall
cd ~/Heimdall/heimdall
sed -i 's/cmake_minimum_required(VERSION [0-9.]*)/cmake_minimum_required(VERSION 3.5)/' CMakeLists.txt
```

### 5. Configure Heimdall Build System

The heimdall CMakeLists.txt needs several modifications to work in Termux:

```bash
cd ~/Heimdall/heimdall

# Add CMake module path so it can find LargeFiles.cmake
sed -i '2 a set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/../cmake")' CMakeLists.txt

# Replace find_package(libusb) with pkg-config detection
sed -i 's/find_package(libusb REQUIRED)/find_package(PkgConfig REQUIRED)\npkg_check_modules(LIBUSB REQUIRED libusb-1.0)\ninclude_directories(${LIBUSB_INCLUDE_DIRS})/' CMakeLists.txt

# Fix library variable name for pkg-config
sed -i 's/${LIBUSB_LIBRARY}/${LIBUSB_LIBRARIES}/' CMakeLists.txt
```

### 6. Fix libpit Linking

```bash
cd ~/Heimdall/heimdall

# Point to the actual libpit.a file we'll build
sed -i 's|target_link_libraries(heimdall PRIVATE pit)|target_link_libraries(heimdall PRIVATE ${CMAKE_SOURCE_DIR}/../libpit/build/libpit.a)|' CMakeLists.txt

# Add libpit include directory
sed -i '/include_directories(${LIBUSB_INCLUDE_DIRS})/a include_directories(${CMAKE_SOURCE_DIR}/../libpit)' CMakeLists.txt
```

### 7. Build libpit (Heimdall's Dependency)

```bash
cd ~/Heimdall/libpit
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make

# Verify libpit.a was created
ls -lh libpit.a
```

You should see a ~13KB file called `libpit.a`.

### 8. Build Heimdall

```bash
cd ~/Heimdall/heimdall
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make
```

If successful, you'll see `[100%] Built target heimdall`.

### 9. Verify the Build

```bash
./heimdall version
```

Should output: `v1.4.2`

```bash
./heimdall help
```

Should display the help menu with available commands.

## Using the Compiled Binary

Your compiled heimdall binary is located at:
```
~/Heimdall/heimdall/build/heimdall
```

### Basic Usage Example

```bash
# Detect device in Download Mode (requires root on Android)
tsu  # Enter root shell
~/Heimdall/heimdall/build/heimdall detect

# Flash TWRP recovery
~/Heimdall/heimdall/build/heimdall flash --RECOVERY twrp.img --no-reboot
```

## Pre-compiled Binary

For convenience, a pre-compiled ARM64 binary is included in this repository as `heimdall-arm64`. 

To use it:
```bash
chmod +x heimdall-arm64
./heimdall-arm64 version
```

**Important:** This binary was compiled in Termux on Android ARM64. It may not work on all ARM devices. If it doesn't work for you, follow the compilation steps above.

## Important Notes

### Root Requirement
- Heimdall requires **direct USB hardware access** to communicate with devices in Download Mode
- On Android, this requires **root access** (kernel-level permissions)
- Shizuku/Dhizuku will NOT work (they only provide ADB-level permissions)
- If your device cannot be rooted, you must use a desktop/laptop computer instead

### USB OTG Cable Required
- You need a USB OTG (On-The-Go) cable/adapter to connect the target device to your Android phone
- The target device must be in Download Mode (Odin mode)

### Alternative: Desktop Usage
If you cannot root your Android device, it's easier to:
1. Use any desktop/laptop (Windows/Mac/Linux)
2. Download the appropriate Heimdall binary for your OS
3. Flash your device from there

The ARM/Android compilation is primarily useful for users who:
- Have a rooted Android device
- Don't have access to a computer
- Want to flash Samsung devices on-the-go

## Troubleshooting

### "unable to find library -lpit"
- Make sure you built libpit first (step 7)
- Verify `~/Heimdall/libpit/build/libpit.a` exists

### "Could not find libusb"
- Install libusb: `pkg install libusb`
- Make sure pkg-config is installed: `pkg install pkg-config`

### "Permission denied" when flashing
- You need root access: run `tsu` first
- Make sure your Android device is rooted
- Verify USB OTG is working: `ls /dev/bus/usb/`

### Heimdall can't detect device
- Ensure target device is in Download Mode
- Check USB OTG connection
- Verify you're running as root (`tsu`)
- Try: `ls -la /dev/bus/usb/` to see if device appears

## Credits

- **Frédéric Basse** - Original [SVE-2016-7930 exploit](https://github.com/frederic/SVE-2016-7930) and Heimdall patch
- **Benjamin Dobell** - Original [Heimdall](https://github.com/Benjamin-Dobell/Heimdall) developer
- **justaCasualCoder** - [G900A-TWRP-ROM](https://github.com/justaCasualCoder/G900A-TWRP-ROM) project
- **TheGreatOleander** - ARM/Android compilation guide and binary

## License

Heimdall is licensed under the MIT License. See the original Heimdall repository for details.

## Additional Resources

- [Original SVE-2016-7930 Paper (PDF)](https://www.sstic.org/media/SSTIC2017/SSTIC-actes/attacking_samsung_secure_boot/SSTIC2017-Article-attacking_samsung_secure_boot-basse.pdf)
- [Heimdall Documentation](https://github.com/Benjamin-Dobell/Heimdall)
- [Main G900A-TWRP-ROM README](README.md) - for complete device-specific instructions
