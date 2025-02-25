Certainly! Here's a guide to retrieving various system information on macOS 12+ using Swift. This guide utilizes native APIs and provides code snippets for each requirement.

### 1. Swift Version
Swift version is typically known at compile time, but you can use compiler flags to embed it.

```swift
let swiftVersion = "Swift " + String(describing: Swift.version)
print(swiftVersion)
```

### 2. OS Name and Version
Use `ProcessInfo` to get OS details.

```swift
import Foundation

let processInfo = ProcessInfo.processInfo
let osName = processInfo.operatingSystemVersionString
let osVersion = "\(processInfo.operatingSystemVersion.majorVersion).\(processInfo.operatingSystemVersion.minorVersion).\(processInfo.operatingSystemVersion.patchVersion)"
print("OS Name: \(osName)")
print("OS Version: \(osVersion)")
```

### 3. OS Architecture
Retrieve architecture using `Host`.

```swift
import Foundation

#if arch(x86_64)
let architecture = "x86_64"
#elseif arch(arm64)
let architecture = "arm64"
#else
let architecture = "Unknown"
#endif
print("OS Architecture: \(architecture)")
```

### 4. Host Information
Use `Host` APIs.

```swift
import Foundation

let hostName = Host.current().localizedName ?? "Unknown"
print("Host Name: \(hostName)")
```

### 5. Kernel Name and Version
Use `uname` from libc.

```swift
import Foundation

import Darwin

var uts = utsname()
uname(&uts)
let kernelName = String(bytes: Data(bytes: &uts.sysname, count: Int(_SYS_NAMELEN)), encoding: .utf8)?.trimmingCharacters(in: .controlCharacters) ?? "Unknown"
let kernelVersion = String(bytes: Data(bytes: &uts.release, count: Int(_SYS_NAMELEN)), encoding: .utf8)?.trimmingCharacters(in: .controlCharacters) ?? "Unknown"
print("Kernel Name: \(kernelName)")
print("Kernel Version: \(kernelVersion)")
```

### 6. Uptime

```swift
import Foundation

let uptime = processInfo.systemUptime
print("Uptime: \(uptime) seconds")
```

### 7. Display Resolutions and Refresh Rates
Use Core Graphics.

```swift
import CoreGraphics

if let screens = NSScreen.screens as? [NSScreen] {
    for (index, screen) in screens.enumerated() {
        let resolution = screen.frame.size
        let refreshRate = screen.deviceDescription[NSDeviceDescriptionKey("NSDeviceRefreshRate")] as? Double ?? 0
        print("Display \(index + 1): \(resolution.width)x\(resolution.height), Refresh Rate: \(refreshRate)Hz")
    }
}
```

### 8. CPU Information
Use `sysctl` for CPU details.

```swift
import Foundation

import sysctl

func getCPUInfo() -> (name: String, cores: Int, frequency: Double) {
    var size = 0
    sysctlbyname("machdep.cpu.brand_string", nil, &size, nil, 0)
    var cpuName = [CChar](repeating: 0, count: size)
    sysctlbyname("machdep.cpu.brand_string", &cpuName, &size, nil, 0)
    let name = String(cString: cpuName)

    var cores: Int = 0
    size = MemoryLayout<Int>.size
    sysctlbyname("hw.ncpu", &cores, &size, nil, 0)

    var freq: Double = 0
    size = MemoryLayout<Double>.size
    sysctlbyname("hw.cpufrequency", &freq, &size, nil, 0)
    freq = freq / 1_000_000_000 // GHz

    return (name, cores, freq)
}

let cpuInfo = getCPUInfo()
print("CPU Name: \(cpuInfo.name)")
print("CPU Cores: \(cpuInfo.cores)")
print("CPU Frequency: \(cpuInfo.frequency) GHz")
```

### 9. GPU Information
Use IOKit to get GPU details.

```swift
import Foundation
import IOKit
import IOKit.graphics

func getGPUInfo() -> [(name: String, integrated: Bool)] {
    var gpuInfos: [(String, Bool)] = []
    let matching = IOServiceMatching("IOPCIDevice")
    var iterator: io_iterator_t = 0
    if IOServiceGetMatchingServices(kIOMasterPortDefault, matching, &iterator) == KERN_SUCCESS {
        var service = IOIteratorNext(iterator)
        while service != 0 {
            if let model = IORegistryEntrySearchCFProperty(service, kIOServicePlane, "model" as CFString, nil, IOOptionBits(kIORegistryIterateRecursively)) as? String {
                let integrated = model.contains("Intel") || model.contains("Apple")
                gpuInfos.append((model, integrated))
            }
            service = IOIteratorNext(iterator)
        }
        IOObjectRelease(iterator)
    }
    return gpuInfos
}

let gpus = getGPUInfo()
for gpu in gpus {
    print("GPU Name: \(gpu.name), Integrated: \(gpu.integrated)")
}
```

*Note:* GPU core count and frequency aren't readily accessible via public APIs.

### 10. System Memory
Use `ProcessInfo`.

```swift
import Foundation

let physicalMemory = processInfo.physicalMemory
print("Total System Memory: \(ByteCountFormatter.string(fromByteCount: Int64(physicalMemory), countStyle: .memory))")

// Used memory requires more detailed APIs, often using host_statistics
import MachO

func getUsedMemory() -> UInt64 {
    var size = mach_msg_type_number_t(MemoryLayout<vm_statistics64_data_t>.stride / MemoryLayout<integer_t>.stride)
    var vmStats = vm_statistics64()
    let host = mach_host_self()
    let result = withUnsafeMutablePointer(to: &vmStats) {
        $0.withMemoryRebound(to: integer_t.self, capacity: Int(size)) {
            host_statistics64(host, HOST_VM_INFO64, $0, &size)
        }
    }
    if result != KERN_SUCCESS {
        return 0
    }
    let used = UInt64(vmStats.active_count + vmStats.inactive_count + vmStats.wire_count) * UInt64(vm_page_size)
    return used
}

let usedMemory = getUsedMemory()
print("Used System Memory: \(ByteCountFormatter.string(fromByteCount: Int64(usedMemory), countStyle: .memory))")
```

### 11. System Storage
Use `FileManager`.

```swift
import Foundation

let fileURL = URL(fileURLWithPath: "/")
if let values = try? fileURL.resourceValues(forKeys: [.volumeTotalCapacityKey, .volumeAvailableCapacityKey]) {
    if let total = values.volumeTotalCapacity, let available = values.volumeAvailableCapacity {
        let used = total - available
        print("Total Storage: \(ByteCountFormatter.string(fromByteCount: Int64(total), countStyle: .file))")
        print("Used Storage: \(ByteCountFormatter.string(fromByteCount: Int64(used), countStyle: .file))")
    }
}
```

### 12. Battery Level and Power Adapter
Use IOKit.

```swift
import Foundation
import IOKit.ps

func getPowerInfo() -> (batteryLevel: Double?, isCharging: Bool?) {
    if let snapshot = IOPSCopyPowerSourcesInfo()?.takeRetainedValue(),
       let sources = IOPSCopyPowerSourcesList(snapshot)?.takeRetainedValue() as? [CFTypeRef],
       let source = sources.first,
       let description = IOPSGetPowerSourceDescription(snapshot, source)?.takeUnretainedValue() as? [String: Any] {

        let isCharging = description["IsCharging"] as? Bool
        let currentCapacity = description["CurrentCapacity"] as? Int
        let maxCapacity = description["MaxCapacity"] as? Int
        let batteryLevel = currentCapacity != nil && maxCapacity != nil ? Double(currentCapacity!) / Double(maxCapacity!) : nil
        return (batteryLevel, isCharging)
    }
    return (nil, nil)
}

let powerInfo = getPowerInfo()
if let level = powerInfo.batteryLevel {
    print("Battery Level: \(Int(level * 100))%")
} else {
    print("Battery Level: Not applicable")
}

if let charging = powerInfo.isCharging {
    print("Is Charging: \(charging)")
} else {
    print("Power Adapter: Not connected or not applicable")
}
```

### Summary
This guide covers retrieving various system information using Swift on macOS 12+. Some details like GPU core count and frequency may require private APIs or additional permissions. Always ensure your app has the necessary entitlements and handles permissions appropriately.

**Dependencies:**
- Import frameworks such as `Foundation`, `IOKit`, `CoreGraphics`, and `AppKit` where necessary.
- For GPU and memory usage, raw system calls are used, which may require additional error handling in production code.

**Note:**
- Accessing certain hardware information may require elevated privileges or may not be fully supported due to sandboxing and privacy restrictions.
- Ensure your project links against the necessary frameworks like `IOKit`.

Feel free to integrate these snippets into your project and expand upon them as needed!
