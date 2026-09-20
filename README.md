# Corne + ESB Split Transport

这是基于 [unified-zmk-config-template](https://github.com/zmkfirmware/unified-zmk-config-template)
填充出来的 **zmk-config + ZMK module 二合一仓库**：里面既有键盘配置，也有
[badjeff/zmk-feature-split-esb](https://github.com/badjeff/zmk-feature-split-esb)
（Enhanced ShockBurst split transport）的源码，可以直接在 GitHub Actions 上编译出
带 ESB 功能的固件。

## 拓扑

```
        [主机 PC] ←— USB —→  nice_nano（corne_esb_dongle，central，无按键）
                                   ↑ ESB 2.4G（1ms 级别）
                     ┌─────────────┴─────────────┐
        corne_left（peripheral id=1）        corne_right（peripheral id=2）
```

- dongle 走 USB HID 连主机，左右手只通过 ESB 把按键事件发给 dongle；
- **BLE 被完全关闭**（`CONFIG_ZMK_BLE=n`）：ESB 模块目前无法与 BLE 同时编译；
- keymap 只有 dongle 用得上（central 决定行为），但左右手固件也必须带同样的布局。

## 目录结构

```
├── build.yaml                          # 构建矩阵（dongle / left / right 三个固件）
├── config/
│   ├── west.yml                        # zmk(main) + sdk-nrf + nrfxlib
│   ├── corne.keymap                    # 42 键 6 列 keymap（三个固件共用）
│   ├── corne.overlay                   # ESB 收发地址（三个固件共用）
│   ├── corne_left.conf                 # 左手 peripheral（id=1）
│   ├── corne_right.conf                # 右手 peripheral（id=2）
│   └── corne_esb_dongle.conf           # dongle central（USB + ESB 接收）
├── boards/shields/corne_esb_dongle/    # 无按键 dongle shield（mock kscan + 物理布局）
├── src/  dts/  CMakeLists.txt  Kconfig # ESB 模块本体（视为本仓库的一个 ZMK 模块）
└── zephyr/module.yml                   # 让 CI 把本仓库当模块加载
```

命名规则要点（ZMK 的查找顺序）：

- `config/corne.keymap`、`config/corne.overlay`、`config/corne.conf` 会同时作用于
  `corne_left`、`corne_right`，以及我们自己的 `corne_esb_dongle`；
- `config/corne_left.conf` / `corne_right.conf` / `corne_esb_dongle.conf` 只作用于对应固件；
- overlay 只会取第一个命中的文件（`corne_esb_dongle.overlay` → `corne.overlay`），
  所以 ESB 地址写一份就够，而 `.conf` 是叠加生效的。

## 编译

仓库 push 到 GitHub 后，`.github/workflows/build.yml` 会调用 ZMK 官方的
`build-user-config.yml@main` 自动编译；也可以在 Actions 页面手动 `workflow_dispatch`。

产物（artifact `firmware`）里会有：

| 文件（`<shield>-<board>-zmk.uf2`） | 用途 |
| --- | --- |
| `corne_esb_dongle-nice_nano@2.0.0__zmk-zmk.uf2` | 接收器（插主机 USB 的那块） |
| `corne_left-nice_nano@2.0.0__zmk-zmk.uf2` | 左手 |
| `corne_right-nice_nano@2.0.0__zmk-zmk.uf2` | 右手 |

（board 名里的 `/` 会被换成 `_`，所以是 `nice_nano@2.0.0__zmk`。）

## 刷机

1. 三块 nice_nano 分别双击 reset 键，进入 UF2 bootloader（出现 U 盘）；
2. 把对应 uf2 拖进去：
   - 接收器那块刷 `corne_esb_dongle`；
   - 左右手分别刷 `corne_left` / `corne_right`；
3. 先把 dongle 插到主机 USB，再打开左右手电源。首次配对不需要任何操作
   （ESB 用的是固定地址，不是 BLE 那种配对）。

## 常用修改

| 想改什么 | 改哪里 |
| --- | --- |
| 键位 / 层 | `config/corne.keymap` |
| ESB 收发地址（多套设备共存时必须改） | `config/corne.overlay` |
| 哪个 peripheral 是几号 | `config/corne_left.conf`、`config/corne_right.conf` 的 `ZMK_SPLIT_ESB_PERIPHERAL_ID` |
| 发射功率、重传、跳频 | 三个 `.conf` 里的 `CONFIG_ZMK_SPLIT_ESB_*`（**收发两端必须一致**） |
| 再加一块外设（例如鼠标 / 轨迹球） | `build.yaml` 加一项 + `corne_esb_dongle.conf` 里的 `ZMK_SPLIT_ESB_PERIPHERAL_COUNT` / `CONFIG_ZMK_SPLIT_BLE_CENTRAL_PERIPHERALS` 调大 + 见下面「接入鼠标 / 轨迹球」 |
| 省电（一段时间后深睡） | 取消 `corne_left.conf` / `corne_right.conf` 里 `CONFIG_ZMK_SLEEP` 的注释 |

## 接入鼠标（键鼠套装：一块键盘 + 一个鼠标，都由这个 dongle 收）

鼠标在 ESB 网络里是一个 peripheral，它的信息分成**两条独立的通道**：

| 通道 | 走什么 | dongle 侧由谁定义 |
| --- | --- | --- |
| 指针移动 / 滚轮 | `zmk,input-split`（input 事件，reg=4） | `corne_esb_dongle.overlay` 的 `mouse_split` + `mouse_listener`；滚轮靠占位 `zmk,keymap-sensors` |
| 5 个鼠标按键 | **key position（位置 0~4）** | `config/corne.keymap` 的前 5 个位置 |

**鼠标固件侧**（本仓库已在 `01_twinkleeKin-ariska-main` 里改好）

1. 用同一套 ESB 实现（本仓库 `src/` 这个模块 / badjeff 的 `zmk-feature-split-esb`），
   且 `CONFIG_ZMK_BLE=n`、`CONFIG_ZMK_SPLIT_BLE=n`、`CONFIG_ZMK_SPLIT_WIRED=n`、
   `CONFIG_ZMK_SPLIT_ESB=y`；
2. ESB 地址三件套与 dongle 的 `config/corne.overlay` **完全一致**（逐字节）；
3. ESB 参数一致（`..._PROTO_TX_ACK`、`..._MSG_POSTFIX_CRC`、重传 delay/count、
   `..._CTLR_TX_PWR_PLUS_8`、`..._RF_CH_HOP*`、`CONFIG_ESB_MAX_PAYLOAD_LENGTH=48`、
   `CONFIG_ESB_TX_FIFO_SIZE=1`）；
4. `CONFIG_ZMK_SPLIT_ESB_PERIPHERAL_ID` 唯一：left = 1、right = 2、鼠标 = 3。
   同一网络最多 8 台设备同时通信（射频 pipe 数），实际占用 `id % 8`，
   所以任意两台设备的 `id % 8` 不能相同；
5. `CONFIG_ZMK_POINTING=y`，并给自己的指针设备建 `zmk,input-split` 节点
   （`device = <&mou0>`、`reg = <4>`）；
6. **按键部分保持原样**：仍然是 `zmk,kscan-gpio-direct` 上报位置 0~4，
   不需要任何改动。

**dongle 侧 —— 本仓库已按 Lariska 鼠标配好**

1. `config/corne.overlay`：在 Corne 的 `matrix-transform` map **前面插了 5 个占位**，
   把 Corne 的 42 键整体推到位置 **5~46**，把 **0~4 让给鼠标的 5 个按键**
   （ZMK 里 map 的下标就是 keymap 的位置号，所以这是唯一能"不改鼠标却让两边不撞车"
   的做法）；
2. `config/corne.keymap`：现在是 **47 个位置 × 5 层**——
   前 5 个位置是 `&mkp LCLK / RCLK / MCLK / u_lt_mouse 3 MB4 / u_lt_mouse 4 MB5`，
   后面 42 个是原来的 Corne 键位；
   - 层 3 = 按住鼠标第 4 键（滚轮变音量），层 4 = 按住鼠标第 5 键（滚轮），
     层号从鼠标原来的 1/2 改成 3/4，免得和 Corne 的 lower/raise 冲突；
3. `boards/shields/corne_esb_dongle/corne_esb_dongle.overlay`：
   - `mouse_split@4` + `mouse_listener`（指针）
   - 占位 `encoder`（`alps,ec11`，挂在 P0.09/P0.10 这两个没用的 NFC 脚上）
     + `zmk,keymap-sensors`，用来接收鼠标的滚轮 sensor 事件；
4. `config/corne_esb_dongle.conf`：`CONFIG_ZMK_POINTING=y`、`CONFIG_ZMK_INPUT_SPLIT=y`、
   `CONFIG_EC11=y`、`CONFIG_ZMK_SPLIT_ESB_AUTO_HEAL_KEY_POS_MAX=47`、
   `CONFIG_ZMK_SPLIT_ESB_PERIPHERAL_COUNT=4`（必须 **大于** 最大的 `PERIPHERAL_ID`）。

不需要配对或对码：ESB 用固定地址，dongle 插上、鼠标开机就应该通了。

**排查手段**：给 dongle 的构建项加 `snippet: zmk-usb-logging`（`build.yaml` 里），
并临时打开 `CONFIG_ZMK_SPLIT_ESB_LOG_LEVEL_DBG=y`，就能通过 USB 看它收到了什么。
如果滚轮不动，先看 dongle 日志里有没有 sensor 事件（占位编码器那个节点是关键）。

## 注意事项

- **必须使用 ZMK main（0.4 / Zephyr 4.1 一代）**：ESB 模块的 `zmk/split/transport`
  接口、`nice_nano@2.0.0//zmk` 这类 board 名字都只在这一代存在。`config/west.yml`
  里的 `zmk` 已经 pin 到官方 commit `9ebbeff0`（2026-09-15），**鼠标固件项目用的是
  同一个 commit**；要升级就两边一起升，并同步 `.github/workflows/build.yml` 里的 `@main`。
- `sdk-nrf` 必须用 `badjeff` 的 `v3.1-branch+zmk-fixes`（NCS 3.1 + ZMK 适配补丁），
  用官方分支会在 CMake 配置校验阶段失败。
- ESB 是不可加密的 2.4G 私有协议，也是**固定地址**：同一环境下两套设备要用不同的
  `config/corne.overlay` 地址，否则会互相干扰。
- 本仓库 root 的 `CMakeLists.txt` / `Kconfig` / `src` / `dts` 就是 ESB 模块本身。
  想改回「模块放独立仓库」的形式，就把这些文件删掉，并在 `config/west.yml` 里加回
  `zmk-feature-split-esb` project（注释里有示例）。

## 本地构建（可选）

本机没装 west / Zephyr SDK 的话可以跳过，直接用 GitHub Actions。本地构建需要：

```shell
# 在仓库根目录
west init -l config
west update
west zephyr-export
west build -s zmk/app -b nice_nano@2.0.0//zmk -d build/dongle -- \
  -DZMK_CONFIG="$(pwd)/config" -DSHIELD="corne_esb_dongle" \
  -DZMK_EXTRA_MODULES="$(pwd)"
```

## 致谢 / 许可

- ESB split transport 模块：<https://github.com/badjeff/zmk-feature-split-esb>
  （`src/` 下文件头部的 SPDX 标识沿用上游：Nordic-5-Clause / MIT）
- 仓库骨架：<https://github.com/zmkfirmware/unified-zmk-config-template>
- Corne shield / keymap：ZMK 官方 `app/boards/shields/corne`
