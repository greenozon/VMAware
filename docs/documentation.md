# Documentation

## Contents
- [`VM::detect()`](#vmdetect)
- [`VM::brand()`](#vmbrand)
- [`VM::check()`](#vmcheck)
- [`VM::percentage()`](#vmpercentage)
- [Flag table](#flag-table)
- [Non-technique flags](#non-technique-flags)

<br>

## `VM::detect()`

This is basically the main function you're looking for, which returns a bool. If the parameter is set to nothing, all the recommended checks will be performed. But you can optionally set what techniques are used.

```cpp
#include "vmaware.hpp"

int main() {
    /**
     * The basic way to detect a VM where most checks will be 
     * performed. This is the recommended usage of the library.
     */ 
    bool is_vm = VM::detect();


    /**
     * Essentially means only the brand, MAC, and hypervisor bit techniques 
     * should be performed. Note that the less flags you provide, the more 
     * likely the result will not be accurate. If you just want to check for 
     * a single technique, use VM::check() instead. Also, read the flag table
     * at the end of this doc file for a full list of technique flags.
     */
    bool is_vm2 = VM::detect(VM::BRAND | VM::MAC | VM::HYPERV_BIT);


    /**
     * All checks are performed including the cursor check, 
     * which waits 5 seconds for any human mouse interaction 
     * to detect automated virtual environments. This is the 
     * only technique that's disabled by default but if you 
     * want to include it, add VM::ALL which is NOT RECOMMENDED
     */ 
    bool is_vm3 = VM::detect(VM::ALL);


    /**
     * If you don't want the value to be memoized for whatever reason, 
     * you can set the VM::NO_MEMO flag and the result will not be cached. 
     * It's recommended to use this flag if you're only using one function
     * from the public interface a single time in total, so no unneccessary 
     * caching will be operated when you're not going to re-use the previous result. 
     */ 
    bool is_vm4 = VM::detect(VM::ALL | VM::NO_MEMO);


    /**
     * If you want to treat any technique that was detected as positive,
     * you can enable the VM::EXTREME flag which will return true if any
     * technique has detected a hit despite the certainty score. This is
     * not recommended for obvious reasons.
     */ 
    bool is_vm5 = VM::detect(VM::EXTREME);


    /**
     * This will essentially mean "perform all the default flags, but only disable
     * the VM::RDTSC technique". 
     */ 
    bool is_vm6 = VM::detect(VM::DEFAULT & ~(VM::RDTSC));
}
```

<br>

## `VM::brand()`
This will essentially return the VM brand as a `std::string`. The exact possible brand string return values are: 
- `VMware`
- `VirtualBox`
- `bhyve`
- `KVM`
- `QEMU`
- `QEMU/KVM`
- `Microsoft Hyper-V`
- `Microsoft x86-to-ARM`
- `Parallels`
- `Xen HVM`
- `ACRN`
- `QNX hypervisor`
- `Hybrid Analysis`
- `Sandboxie`
- `Docker`
- `Wine`
- `Virtual Apple`
- `Virtual PC`
- `Anubis`
- `JoeBox`
- `Thread Expert`
- `CW Sandbox`
- `SunBelt`
- `Comodo`
- `Bochs`


If none were detected, it will return `Unknown`. It's often NOT going to produce a satisfying result due to technical difficulties with accomplishing this, on top of being highly dependent on what mechanisms detected a VM. Don't rely on this function for critical operations as if it's your golden bullet. Roughly 75% of the time it'll simply return `Unknown`, assuming it is actually running under a VM.

```cpp
#include "vmaware.hpp"
#include <string>

int main() {
    const std::string result = VM::brand();

    if (result == "KVM") {
        // do KVM specific stuff
    } else if (result == "VirtualBox") {
        // do vbox specific stuff
    } else {
        // you get the idea
    }
}
```

<br>

## `VM::check()`
This takes a single flag argument and returns a `bool`. It's essentially the same as `VM::detect()` but it doesn't have a scoring system. It only returns the technique's effective output. The reason why this exists is because it allows end-users to have fine-grained control over what is being executed and what isn't. 

`VM::detect()` is meant for a range of techniques to be evaluated in the bigger picture with weights and biases in its scoring system, while `VM::check()` is meant for a single technique to be evaluated without any points or anything extra. It just gives you what the technique has found on its own. For example:

```cpp
#include "vmaware.hpp"
#include <iostream>

int main() {
    if (VM::check(VM::VMID)) {
        std::cout << "VMID technique detected a VM!\n";
    }

    if (VM::check(VM::HYPERVISOR_BIT)) {
        std::cout << "Hypervisor bit is set, most definitely a VM!\n";
    }

    // invalid, will throw an std::invalid_argument exception
    bool result = VM::check(VM::VMID | VM::HYPERVISOR_BIT);
}
```

<br>

## `VM::percentage()`
This will return a `std::uint8_t` between 0 and 100. It'll return the certainty of whether it has detected a VM based on all the techniques available as a percentage. The lower the value, the less chance it's a VM. The higher the value, the more likely it is. The parameters are treated the exact same way with the `VM::detect()` function.

```cpp
#include "vmaware.hpp"
#include <iostream>
#include <cstdint>

int main() {
    // uint8_t and unsigned char works too
    const std::uint8_t percent = VM::percentage();

    if (percent == 100) {
        std::cout << "Definitely a VM!\n";
    } else if (percent == 0) {
        std::cout << "Definitely NOT a VM";
    } else {
        std::cout << "Unsure if it's a VM";
    }

    // converted to std::uint32_t for console character encoding reasons
    std::cout << "percentage: " << static_cast<std::uint32_t>(percent) << "%\n"; 
}
```

<br>

# Flag table
VMAware provides a convenient way to not only check for VMs, but also have the flexibility and freedom for the end-user to choose what techniques are used with complete control over what gets executed or not. This is handled with a flag system.


| Flag alias | Description | Cross-platform? | Certainty | Root required? |
| ---------- | ----------- | --------------- | --------- | -------------- |
| `VM::VMID` | Check if the CPU manufacturer ID matches that of a VM brand | Yes | 100% |  |
| `VM::BRAND` | Check if the CPU brand string contains any indications of VM keywords | Yes | 50% |  |
| `VM::HYPERVISOR_BIT` | Check if the hypervisor bit is set (always false on physical CPUs) | Yes | 100% |  |
|`VM::CPUID_0X4` | Check if there are any leaf values between 0x40000000 and 0x400000FF that changes the CPUID output | Yes | 70% |  |
| `VM::HYPERVISOR_STR` | Check if brand string length is long enough (would be around 2 characters in a host machine while it's longer in a hypervisor) | Yes | 45% |  |
| `VM::RDTSC` | Benchmark RDTSC and evaluate its speed, usually it's very slow in VMs | Linux and Windows | 20% |  |
| `VM::SIDT5` | Check if the 5th byte after sidt is null | Linux | 45% |  |
| `VM::THREADCOUNT` | Check if there are only 1 or 2 threads, which is a common pattern in VMs with default settings (nowadays physical CPUs should have at least 4 threads for modern CPUs) | Yes | 35% |  |
| `VM::MAC` | Check if the system's MAC address matches with preset values for certain VMs | Linux and Windows | 90% |  |
| `VM::TEMPERATURE` | Check for the presence of CPU temperature sensors (mostly not present in VMs) | Linux | 15% |  
| `VM::SYSTEMD` | Get output from systemd-detect-virt tool | Linux | 70% |  |
| `VM::CVENDOR` | Check if the chassis has any VM-related keywords | Linux | 65% |  |
| `VM::CTYPE` | Check if the chassis type is valid (usually not in VMs) | Linux | 10% |  |
| `VM::DOCKERENV` | Check if any docker-related files are present such as /.dockerenv and /.dockerinit | Linux | 80% |  |
| `VM::DMIDECODE` | Get output from dmidecode tool and grep for common VM keywords | Linux | 55% | Yes |
| `VM::DMESG` | Get output from dmesg tool and grep for common VM keywords | Linux | 55% |  |
| `VM::HWMON` | Check if HWMON is present (if not, likely a VM) | Linux | 75% |  |
| `VM::CURSOR`  | Check if cursor isn't active (sign of automated VM environment) | Windows | 10% |  |
| `VM::VMWARE_REG` | Look for any VMware-specific registry data | Windows | 65% |  |
| `VM::VBOX_REG` | Look for any VirtualBox-specific registry data | Windows | 65% |  |
| `VM::USER` | Match the username for any defaulted ones | Windows | 35% |  |
| `VM::DLL` | Match for VM-specific DLLs | Windows | 50% |  |
| `VM::REGISTRY` | Look throughout the registry for all sorts of VMs | Windows | 75% |  |
| `VM::SUNBELT_VM` | Detect for Sunbelt technology | Windows | 10% |  |
| `VM::WINE_CHECK` | Find for a Wine-specific file | Windows | 85% |  |
| `VM::VM_FILES` | Find if any VM-specific files exists | Windows | 20% |  |
| `VM::HWMODEL` | Check if the sysctl for the hwmodel does not contain the "Mac" string | MacOS | 75% |  |
| `VM::DISK_SIZE` | Check if disk size is under or equal to 50GB | Linux | 60% |  |
| `VM::VBOX_DEFAULT` | Check for default RAM and DISK sizes set by VirtualBox | Linux and Windows | 55% | Yes |
| `VM::VBOX_NETWORK` | Check VBox network provider string | Windows | 70% |  |
| `VM::COMPUTER_NAME` | Check for computer name string | Windows | 40% |  |
| `VM::HOSTNAME` | Check if hostname is specific | Windows 25% |  |
| `VM::MEMORY` | Check if memory space is far too low for a physical machine | Windows | 35% |  |
| `VM::VM_PROCESSES` | Check for any VM processes that are active | Windows | 30% |  |
| `VM::LINUX_USER_HOST` | Check for default VM username and hostname for linux | Linux | 35% |  |
| `VM::VBOX_WINDOW_CLASS` | Check for the window class for VirtualBox | Windows | 10% |  |
| `VM::WMIC` | Check top-level default window level | Windows | 20% |  | 
| `VM::GAMARUE` | Check for Gamarue ransomware technique which compares VM-specific Window product IDs | Windows | 40% |  | 
| `VM::VMID_0X4` | Check if the CPU manufacturer ID matches that of a VM brand with leaf 0x40000000 | Yes | 100% |  |
| `VM::PARALLELS_VM` | Check for indications of Parallels VM | Windows | 50% |  |
| `VM::RDTSC_VMEXIT` | Check for RDTSC technique with VMEXIT | Yes | 50% |  |
| `VM::LOADED_DLLS` | Check for DLLs of multiple VM brands | Windows | 75% |  |
| `VM::QEMU_BRAND` | Check for QEMU CPU brand with cpuid | Yes | 100% |  | 
| `VM::BOCHS_CPU` | Check for Bochs cpuid emulation oversights | Yes | 95% |  |
| `VM::VPC_BOARD` | Check for VPC specific string in motherboard manufacturer | Windows | 20% |  | 
| `VM::HYPERV_WMI` | Check for Hyper-V wmi output | Windows | 80% |  |
| `VM::HYPERV_REG` | Check for Hyper-V strings in registry | Windows | 80% |  |
| `VM::BIOS_SERIAL` | Check if BIOS serial number is null | Windows | 60% |  |
| `VM::VBOX_FOLDERS` | Check for VirtualBox-specific string for shared folder ID | Windows | 45% |  |
| `VM::VBOX_MSSMBIOS` | Check VirtualBox MSSMBIOS registry for VM-specific strings | Windows | 75% |  |
| `VM::MAC_HYPERTHREAD` | Check if hyperthreading core count matches with physical expectations | MacOS | 10% |  |
| `VM::MAC_MEMSIZE` | Check if memory is too low for MacOS system | MacOS | 30% |  |
| `VM::MAC_IOKIT` | Check MacOS' IO kit registry for VM-specific strings | MacOS | 80% |  |
| `VM::IOREG_GREP` | Check for VM-strings in ioreg commands for MacOS | MacOS | 75% |  |
| `VM::MAC_SIP` | Check if System Integrity Protection is disabled (likely a VM if it is) | MacOS | 85% |  |
| `VM::KVM_REG` | Check for KVM-specific registry strings | Windows | 75% |  |
| `VM::KVM_DRIVERS` | Check for KVM-specific system files in system driver directory | Windows | 55% |  |
| `VM::KVM_DIRS` | Check for KVM-specific directories | Windows | 55% |  |



<br>

# Non-technique flags
| Flag | Description |
|------|-------------|
| `VM::ALL` | This will enable all the technique flags, including the cursor check that's disabled by default. |
| `VM::NO_MEMO` | This will disable memoization, meaning the result will not be fetched through a previous computation of the `VM::detect()` function. Use this if you're only using a single function from the `VM` struct for a performance boost.
| `VM::EXTREME` | This will disregard the weights/biases and its scoring system. It will essentially treat any technique that found a hit as a VM detection no matter how low that technique's certainty is, so if a single technique is positive then it will return true. | 
| `VM::DEFAULT` | This represents a range of flags which are enabled if no default argument is provided. The reason why this exists is to easily disable any bits manually (shown in the is_vm6 example in the `VM::detect()` section)
