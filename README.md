# 180° SBS VR Viewer for RayNeo Glasses

一个纯前端、免安装的 180°/360° SBS（左右分屏）VR 视频播放器，通过浏览器 **WebHID** 直接读取
RayNeo AR 眼镜的姿态传感器（IMU）实现头动追踪，并能通过 HID 命令控制眼镜的 3D/2D 显示模式。
附带一个用于抓包、调试眼镜 HID 协议的调试工具。

A pure front-end, install-free 180°/360° SBS (side-by-side) VR video player. It reads head
orientation directly from a RayNeo AR glasses' IMU via the browser's **WebHID** API, and can
send HID commands to switch the glasses between 3D and 2D display modes. Includes a companion
debug tool for sniffing and testing the glasses' HID protocol.

---

## 目录 / Contents

- [中文说明](#中文说明)
- [English](#english)

---

## 中文说明

### 文件说明

| 文件 | 用途 |
|---|---|
| `180_sbs_vr_viewer_fixed.html` | 主播放器：加载本地 180°/360° SBS 视频，用手机陀螺仪或眼镜 HID 做头动追踪 |
| `rayneo_hid_debug_fixed.html` | HID 调试工具：查看眼镜发来的原始报文，发送自定义命令，识别应答帧 |

两个都是单文件 HTML，双击直接在浏览器打开即可，不需要安装、不需要联网、不上传任何数据。

### 环境要求

- **浏览器**：Chrome 或 Edge（WebHID 只有 Chromium 系浏览器支持，Safari/Firefox 均不支持）
- **打开方式**：本地双击打开（`file://`）即可用于视频播放；如果要用 HID 功能，**部分浏览器要求
  页面运行在 `https://` 或 `localhost` 下**，本地文件直接打开如果 HID 连接不上，请改用本地静态
  服务器（如 `npx serve`）访问
- **眼镜**：目前只验证过 RayNeo Air 2（VID `0x1bbb`，PID `0xaf50`）；其他型号会被识别为"未知型号"，
  不会自动发送任何命令，但视频播放、手机模式头动追踪不受影响

### 主播放器功能

- **180°/360° 投影**：默认按 180° 半球显示 SBS 视频；工具栏里的 **Proj** 滑块可以从 0°（纯平面）
  调到 360°（整球），180° 时和默认效果完全一致
- **头动追踪两种来源**：
  - **Phone**：用手机/电脑浏览器自带的陀螺仪（`deviceorientation`），支持横竖屏
  - **HID**：直接读取眼镜通过 USB HID 上报的 IMU 数据，延迟更低、精度更高
- **FOV / Split**：分别调节视野角度和左右眼分屏比例
- **View**：Both（双眼）/ Left（左眼）/ Right（右眼）三种显示模式，方便单屏调试
- **Center / Reset**：把当前朝向记为正前方 / 恢复所有参数到默认值
- **右键 = Center**：在画面上点右键等同于点 Center 按钮，同时会阻止浏览器自带右键菜单弹出
- **点击画面 = 隐藏/显示 UI**：点一下播放区域，工具栏、原生播放器控件、提示文字整体隐藏/显示，
  只留纯净画面
- **Glasses 3D / Normal**：连接 HID 后，识别出眼镜型号会自动发送开启 IMU 输入的命令；两个按钮
  可以切换眼镜的 3D（左右分屏）和 Normal 显示模式
- **3D 模式下工具栏双份显示**：点 3D 后，工具栏和底部提示会在左右半屏各显示一份（内容自动保持
  同步），这样透过眼镜镜片看，两只眼能融合成一份完整的 UI，而不是被从中间切开；切回 Normal 后
  自动恢复单份

### 眼镜命令与协议说明（重要，请务必阅读）

眼镜的 HID 协议**没有官方文档**，目前用到的命令都是通过抓包第三方软件（[Verto XR](https://vertoxr.com)）
的通信数据反推出来的，命令格式统一是 `sendReport(reportId=0, [0x66, 操作码, 0x00])`：

| 操作码 | 含义 | 可信度 |
|---|---|---|
| `0x00` | 查询设备信息（会收到一帧含固件版本字符串的应答） | 推测，根据应答帧内容推断 |
| `0x01` | 开启 HID 输入（IMU 数据流） | **已确认** |
| `0x06` | 切换到 3D（左右分屏）模式 | 用户报告，未独立验证 |
| `0x07` | 切换到正常（2D）模式 | 用户报告，未独立验证 |
| 其他 | 未知 | **请不要盲目尝试**，未知命令可能改动眼镜的持久状态 |

- **没有找到"停止 IMU 输出"的命令**。断开 HID 连接后，眼镜仍会继续向外发送 IMU 数据，如需彻底
  停止，目前只能拔插 USB。代码里 `GLASSES_STOP_CMD` / profile 表的 `cmd.stop` 字段留空，等日后
  抓到对应命令直接填进去即可。
- **怎么给别的型号加协议**：打开 `180_sbs_vr_viewer_fixed.html`，搜索 `GLASSES_PROFILES`，照着
  数组里 RayNeo Air 2 那一条的格式（`match` 函数按 `vendorId`/`productId` 识别、`cmd` 里填各个命令
  的字节）加一条即可，不用改其他逻辑。
- **怎么抓自己眼镜的协议**：推荐用 `rayneo_hid_debug_fixed.html` 里的"自定义原始命令"发送框配合
  官方 App/Verto 一起摸索，或者在 Verto 网页版的浏览器控制台里挂 `HIDDevice.prototype.sendReport`
  的钩子，把它发出去的字节打印出来。调试工具会自动识别"疑似应答帧"（不符合已知 IMU 帧特征的
  数据包），帮助你判断某条命令有没有生效。

### 已知限制

- 大分辨率（比如单眼 4K/8K）视频如果播放卡顿，很可能是浏览器硬件解码器跟不上，属于系统层面的
  限制，不是这个页面能解决的；已经做了减少重复纹理上传、去掉不必要的格式转换、默认关闭抗锯齿
  等优化，但解码本身的瓶颈无法通过前端代码绕开
- 3D/Normal 切换命令来自用户实测报告，没有做过独立验证
- 手机横屏模式的姿态换算公式经过验证（对比矩阵运算、three.js 官方 DeviceOrientationControls），
  但没有连真机实测

---

## English

### Files

| File | Purpose |
|---|---|
| `180_sbs_vr_viewer_fixed.html` | Main player: loads a local 180°/360° SBS video, tracks head orientation via phone gyroscope or glasses HID |
| `rayneo_hid_debug_fixed.html` | HID debug tool: inspect raw reports from the glasses, send custom commands, auto-flag reply/ack frames |

Both are single-file HTML apps — just double-click to open in a browser. No install, no network
access required, no data is uploaded anywhere.

### Requirements

- **Browser**: Chrome or Edge (WebHID is Chromium-only; Safari and Firefox are not supported)
- **How to open**: double-clicking the file (`file://`) works fine for video playback. For HID
  features, **some browsers require the page to be served over `https://` or `localhost`** — if
  HID won't connect from a local double-click, serve the file with a local static server (e.g.
  `npx serve`) instead
- **Glasses**: only verified against RayNeo Air 2 (VID `0x1bbb`, PID `0xaf50`). Other models are
  reported as "unrecognized" and no command is auto-sent, but video playback and phone-mode head
  tracking still work normally

### Player features

- **180°/360° projection**: SBS video is shown on a 180° hemisphere by default. The **Proj**
  slider in the toolbar goes from 0° (flat plane) to 360° (full sphere); at 180° it's pixel-identical
  to the original default
- **Two head-tracking sources**:
  - **Phone**: the browser's built-in gyroscope (`deviceorientation`), works in both portrait and
    landscape
  - **HID**: reads IMU data reported directly by the glasses over USB HID — lower latency, higher
    precision
- **FOV / Split**: adjust field of view and the left/right eye split ratio independently
- **View**: Both / Left / Right display modes, useful for single-eye debugging
- **Center / Reset**: re-centers the current orientation as "forward" / restores all settings to
  defaults
- **Right-click = Center**: right-clicking the canvas acts as Center and suppresses the browser's
  native context menu
- **Click canvas = toggle UI**: clicking the video area hides/shows the toolbar, native video
  controls, and status text together, leaving just the rendered scene
- **Glasses 3D / Normal**: once HID is connected and the glasses model is recognized, the "enable
  IMU input" command is sent automatically; the two buttons switch the glasses between 3D
  (side-by-side) and Normal display modes
- **Duplicated toolbar in 3D mode**: after switching to 3D, the toolbar and status text are each
  mirrored into the left and right half of the screen (kept in sync automatically), so each eye
  sees a complete UI through the lenses instead of half of one; switching back to Normal restores
  the single copy

### Glasses commands & protocol notes (please read)

There is **no official documentation** for the glasses' HID protocol. The commands used here were
reverse-engineered by sniffing traffic from a third-party app ([Verto XR](https://vertoxr.com)).
The command format is consistently `sendReport(reportId=0, [0x66, opcode, 0x00])`:

| Opcode | Meaning | Confidence |
|---|---|---|
| `0x00` | Query device info (glasses reply with a frame containing a firmware version string) | Inferred from the reply frame's content |
| `0x01` | Enable HID input (IMU data stream) | **Confirmed** |
| `0x06` | Switch to 3D (side-by-side) mode | User-reported, not independently verified |
| `0x07` | Switch to Normal (2D) mode | User-reported, not independently verified |
| anything else | Unknown | **Do not guess-send these** — unverified commands may alter persistent state on the glasses |

- **No "stop IMU output" command has been found.** After closing the HID connection, the glasses
  keep streaming IMU data; unplugging/replugging USB is currently the only way to fully stop it.
  `GLASSES_STOP_CMD` / the `cmd.stop` field in the profile table is left empty — fill it in once
  that command is found.
- **Adding support for another model**: open `180_sbs_vr_viewer_fixed.html`, search for
  `GLASSES_PROFILES`, and add an entry following the RayNeo Air 2 one's shape (`match` identifies
  the device by `vendorId`/`productId`, `cmd` holds the byte sequences) — no other logic needs to
  change.
- **Sniffing your own glasses' protocol**: use the "custom raw command" box in
  `rayneo_hid_debug_fixed.html` alongside the official app or Verto, or hook
  `HIDDevice.prototype.sendReport` from the browser console on Verto's web version to log what it
  sends. The debug tool automatically flags "suspected reply frames" (packets that don't match the
  known IMU frame shape) to help confirm whether a command had any effect.

### Known limitations

- Stuttering with very high-resolution video (e.g. 4K/8K per eye) is most likely a hardware video
  decoder bottleneck, which is a platform-level limitation this page can't work around. Some
  mitigations are already in place (avoiding redundant texture uploads, dropping an unnecessary
  format conversion, antialiasing off by default), but they can't fix a decode-bound bottleneck.
- The 3D/Normal switch commands come from user-reported testing and haven't been independently verified.
- The landscape-mode phone-orientation formula has been checked against an independent rotation-matrix
  computation and three.js's own `DeviceOrientationControls`, but hasn't been tested on a real device.

---

## 免责声明 / Disclaimer

这里用到的眼镜 HID 命令是逆向工程得到的，不是厂商官方文档，请自行承担使用风险。
The glasses HID commands used here were reverse-engineered and are not from official vendor
documentation. Use at your own risk.
