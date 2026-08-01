# Intel AX201 Bluetooth Driver Troubleshooting and Cause Analysis
#Windows
#bluetooth
<<<<<<< HEAD
#Wireless-AX201
#技嘉Z690
=======
#技嘉z690
#Wireless-AX201


>>>>>>> ddd672a86bebf329736325ade973814a89e8eb27
## I. Fault Description
On **Windows 11 Home Edition 22H2** system, Bluetooth fails to function properly, exhibiting:  
- Bluetooth switch cannot be turned on or is grayed out  
- Yellow exclamation mark appears on Bluetooth devices in Device Manager  
- Official driver installation fails or device is not recognized  

> Fault computer information:  
> - Motherboard: Gigabyte Z690 AORUS ELITE AX DDR4  
> - CPU: Intel i7‑12700KF  
> - Memory: 64GB  
> - System: Windows 11 Home Edition 22H2  
> - Bluetooth chip: Intel Wireless-AX201  
> - Test environment: Attempted on both native machine and VM  

---

## II. Driver Uninstallation and Installation Sequence (Critical Steps)

### 1. Uninstallation Sequence
1. **Wi-Fi Driver**  
   - Device Manager → Network Adapters → Intel Wireless-AX201 → Uninstall Device  
   - Check **Delete driver software for this device**  
2. **Bluetooth Driver**  
   - Device Manager → Bluetooth → Intel Wireless Bluetooth → Uninstall Device  
   - Check **Delete driver software**  
3. **Restart the computer**  

### 2. Installation Sequence
1. **Install Wi-Fi Driver**  
   - Official driver download: [Intel Wireless Driver for Windows 11/10](https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-io/wireless.html)  
   - Install using administrator mode, restart after installation (optional)  
2. **Install Bluetooth Driver**  
   - Official driver download: [Intel Bluetooth Driver for Windows 11/10](https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-io/wireless.html)  
   - Restart after installation  

### 3. Verification
- No yellow exclamation marks on network adapters and Bluetooth devices in Device Manager  
- Windows Bluetooth settings switch is available  
- Test pairing with Bluetooth devices works normally  

---

## III. Troubleshooting Steps (Reference)

1. Confirm chip model and system version  
2. Check BIOS/UEFI for Bluetooth/Wireless function enabled  
3. Uninstall old drivers (Wi-Fi first, then Bluetooth)  
4. Install official drivers (Wi-Fi first, then Bluetooth)  
5. Start Bluetooth-related services:  
   - `Bluetooth Support Service`  
   - `BTHUSB`  
   - `Bluetooth User Support Service`  

---

## IV. Fault Cause Analysis (Z690 Motherboard)

1. **Hardware Level**  
   - AX201 chip combines Wi-Fi and Bluetooth in one chipset  
   - BIOS configuration not enabling CNVi/Wireless module causes device unrecognization  

2. **Driver Level**  
   - Incorrect driver order (Bluetooth first, then Wi-Fi) prevents dependency initialization  
   - Driver version mismatch with Windows 11 22H2  

3. **System Level**  
   - Bluetooth service not started  
   - Registry or old driver residue  

4. **Virtualization Environment**  
   - VM Bluetooth pass-through support is incomplete, leading to recognition failure  

---

## V. Experience and Recommendations

- When operating AX201 Bluetooth on Z690 motherboard, confirm BIOS settings first  
- Uninstallation/installation order: Wi-Fi → Bluetooth  
- Use official driver versions matching the system to avoid Windows Update override  
- VM environment may require USB Bluetooth adapter  

---

## VI. Reference Links
- [Intel Wireless & Bluetooth Official Drivers](https://www.intel.com/content/www/us/en/support/articles/000005511/network-and-io/wireless.html)