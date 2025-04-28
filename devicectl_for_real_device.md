是的，`xcrun devicectl` **有明确的版本限制**。

主要限制在于：

1.  **Xcode 版本:** `devicectl` 是在 **Xcode 15** 中首次引入的。因此，您**必须**安装并使用 Xcode 15 或更高版本才能找到并运行 `xcrun devicectl` 命令。如果您使用的是 Xcode 14 或更早版本，这个命令是不可用的。
    *   您可以通过 `xcodebuild -version` 来查看当前活动的 Xcode 版本。
    *   确保您的命令行工具指向了正确的 Xcode 版本 (`xcode-select -p` 应指向 Xcode 15 或更高版本的路径)。

2.  **macOS 版本:** Xcode 本身有其支持的 macOS 版本要求。例如，Xcode 15 要求 **macOS Ventura 13.5 或更高版本**。因此，间接地说，您需要运行一个能够支持 Xcode 15 或更高版本的 macOS 才能使用 `xcrun devicectl`。

3.  **目标设备操作系统版本:** 虽然 `devicectl` 本身运行在 macOS 上，但它需要与目标设备（iOS, iPadOS 等）进行通信。
    *   通常，新版本的 Xcode 会支持较新和部分较旧的设备操作系统版本。`devicectl` 的功能可能在与非常旧的（Xcode 不再完全支持的）设备操作系统交互时受到限制或无法工作。
    *   此外，某些设备端的操作可能需要特定的操作系统功能。例如，在 **iOS 16 及更高版本**上，您需要先在设备上手动启用**开发者模式 (Developer Mode)**，`devicectl`（以及其他开发工具）才能执行安装、启动、调试等操作。这不是 `devicectl` 本身的版本限制，而是目标设备操作系统的安全要求。

**总结:**

*   **核心要求:** Xcode 15 或更高版本。
*   **系统依赖:** 运行支持所需 Xcode 版本的 macOS (例如 macOS Ventura 13.5+ for Xcode 15)。
*   **设备兼容性:** 目标设备运行的操作系统版本应被您使用的 Xcode 版本所支持，并且可能需要满足特定条件（如启用开发者模式）。

如果您无法运行 `xcrun devicectl`，首先应该检查您的 Xcode 和 macOS 版本是否满足要求。


# 1. 确保设备连接

DEVICE=$(xcrun devicectl list devices | grep -m1 "iPhone" | awk '{print $1}')

# 2. 安装测试应用
# xcrun devicectl device install app \
  --device iPhone13 \
  Demo.app


----------------install----------------

# xcrun devicectl device install app --device ACE Demo.app
15:54:18  Acquired tunnel connection to device.
15:54:18  Enabling developer disk image services.
15:54:18  Acquired usage assertion.
13%... 19%... 32%... 45%... 48%... 49%... 56%... 59%... 60%... 62%... 66%... 68%... 72%... 76%... 80%... 84%... 88%... 92%... 96%... Complete!
App installed:
• bundleID: com.softminilab.qrcode.scan.demo.Demo
• installationURL: file:///private/var/containers/Bundle/Application/95A754DA-5D5C-42B7-9629-E11EFA4A0D09/Demo.app/
• launchServicesIdentifier: unknown
• databaseUUID: 27DD41EA-8179-4339-AE25-26EFBEE080F6
• databaseSequenceNumber: 42676
• options:

----------------list----------------

# xcrun xctrace list devices
== Devices ==
0x2ab70001b1 (0B7F69DD-C968-5F1C-B0D1-5D6AD5998C69)
ACE (18.1) (00008101-001E64211E11001E)

== Simulators ==
13 pro max  15.0 Simulator (15.0) (1EB69E0F-2373-4160-8536-95435B206945)
Apple Watch SE (40mm) (2nd generation) Simulator (11.0) (335BB5A5-FEF1-4B92-B1FC-8A44FCF4897C)
Apple Watch SE (40mm) (2nd generation) (9.0) (39A4F0FA-7155-4044-941A-81F6E175C7DC)


----------------uninstall----------------
xcrun devicectl device uninstall app --device ACE com.softminilab.qrcode.scan.demo.Demo
16:06:09  Enabling developer disk image services.
16:06:09  Acquired usage assertion.
App uninstalled.


------------launch------------

# xcrun devicectl device process launch --device ACE com.softminilab.qrcode.scan.demo.Demo
16:12:15  Enabling developer disk image services.
16:12:15  Acquired usage assertion.
Launched application with com.softminilab.qrcode.scan.demo.Demo bundle identifier.


------------current runing PID------------
# xcrun devicectl device info processes --device ACE | grep 'Demo.app'


------------Kill------------
# xcrun devicectl device process terminate --device ACE --pid 3732

16:34:47  Enabling developer disk image services.
16:34:47  Acquired usage assertion.
Sent signal to terminate process sent to pid 3732


https://gist.github.com/insidegui/b570ec998b9e2aeb730f4e142f0593d1