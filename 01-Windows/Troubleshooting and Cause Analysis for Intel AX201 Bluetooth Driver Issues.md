# Troubleshooting and Cause Analysis for Intel AX201 Bluetooth Driver Issues
# Windows
# bluetooth
<<<<<<< HEAD
# Wireless-AX201
# Gigabyte Z690
=======
# Gigabyte z690
# Wireless-AX201


>>>>>>> ddd672a86bebf329736325ade973814a89e8eb27
## I. Description of the Issue
On the **Windows 11 Home Edition 22H2** system, Bluetooth is not functioning properly, with the following symptoms:
- The Bluetooth switch cannot be turned on or appears grayed out
- There is a yellow exclamation mark next to the Bluetooth device in Device Manager
- Official driver installation fails or the device is not recognized

> Information about the faulty computer:
> - Motherboard: Gigabyte Z690 AORUS ELITE AX DDR4
> - CPU: Intel i7-12700KF
> - Memory: 64GB
> - Operating System: Windows 11 Home Edition 22H2
> - Bluetooth chip: Intel Wireless-AX201
> - Testing environment: Tested both on the local machine and in a virtual machine

---

## II. Order of Driver Uninstallation and Installation (Key Steps)

### 1. Uninstallation Order
1. **Wi-Fi Driver**
   - Device Manager → Network Adapters → Intel Wireless-AX201 → Uninstall device
   - Check the box “Delete driver software for this device”
2. **Bluetooth Driver**
   - Device Manager → Bluetooth → Intel Wireless Bluetooth → Uninstall device
   - Check the box “Delete driver software for this device”
3. **Reboot the computer**

### 2. Installation Order
1. **Install Wi-Fi Driver**
   - Official driver download: [Intel Wireless Driver for Windows 11/10](https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-io/wireless.html)
   - Install using administrator privileges and reboot after installation (optional)
2. **Install Bluetooth Driver**
   - Official driver download: [Intel Bluetooth Driver for Windows 11/10](https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-io/wireless.html)
   - Reboot after installation

### 3. Verification
- There should be no yellow exclamation mark next to the network adapters and Bluetooth device in Device Manager
- The Windows Bluetooth settings switch should be available
- Test pairing with Bluetooth devices to ensure normal functionality

---

## III. Troubleshooting Steps (For Reference)

1. Confirm the chip model and system version
2. Check if Bluetooth/wireless functions are enabled in the BIOS/UEFI
3. Uninstall previous drivers (Wi-Fi first, then Bluetooth)
4. Install official drivers (Wi-Fi first, then Bluetooth)
5. Start related Bluetooth services:
   - `Bluetooth Support Service`
   - `BTHUSB`
   - `Bluetooth User Support Service`

---

## IV. Cause Analysis of the Issue (Specific to Z690 Motherboard)

1. **Hardware Level**
   - The AX201 chip uses a shared driver suite for both Wi-Fi and Bluetooth
   - If the BIOS configuration does not enable the CNVi/wireless module, the device may not be recognized

2. **Driver Level**
   - Incorrect installation order of drivers (Bluetooth first, then Wi-Fi) can prevent dependencies from initializing correctly
   - Driver versions may not be compatible with Windows 11 22H2

3. **System Level**
   - Bluetooth services might not be running properly
   - Residual old driver files or registry entries could cause issues

4. **Virtualization Environment**
   - Virtual machines may not fully support Bluetooth direct access, leading to recognition failures

---

## V. Tips and Recommendations

- When working with the AX201 Bluetooth on a Z690 motherboard, always check the BIOS settings first
- Uninstall and install drivers in the order of Wi-Fi → Bluetooth
- Use official driver versions that match your system to avoid conflicts caused by Windows Update
- In a virtualization environment, you may need a USB Bluetooth adapter

---

## VI. Reference Links
- [Intel Wireless & Bluetooth Official Drivers](https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-io/wireless.html)