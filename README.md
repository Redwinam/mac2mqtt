# Mac2MQTT

Mac2MQTT 让你的 macOS 电脑通过 MQTT 接入 Home Assistant，支持音量、静音、系统睡眠/唤醒、保持唤醒、屏保、电池状态、用户活动传感器，以及（可选）显示器亮度与媒体播放信息。

## 功能概览
- 状态发布：`<前缀>/status/#`
- 命令订阅：`<前缀>/command/#`
- 在线/离线：`<前缀>/status/alive`（保留消息，`online`/`offline`）
- 音量与静音：系统音量 0–100、静音开关
- 保持唤醒：通过 `caffeinate` 阻止系统睡眠
- 系统按钮：睡眠、关机、显示休眠、显示唤醒、屏保
- 电池传感器：笔记本电池百分比
- 用户活动传感器：实时检测输入，10 秒未操作自动 `inactive`
- 可选显示器亮度：BetterDisplay CLI
- 可选媒体信息：media-control（播放/暂停、当前播放信息）

## 我们的实际部署步骤
### 1. 准备 MQTT（Home Assistant）
- 在 HA 的 Mosquitto 插件中添加账户（例如：`hass`/`password`），启用 1883 端口
- 启用 MQTT 自动发现，前缀使用 `homeassistant`

### 2. 安装与目录结构
- 将二进制与配置放到同一目录（必须）：`~/mac2mqtt`
- 复制：`mac2mqtt`、`mac2mqtt.yaml`
- 配置示例：
```yaml
mqtt_ip: 192.168.31.227
mqtt_port: 1883
mqtt_user: hass
mqtt_password: password
mqtt_ssl: false
hostname: macbook-pro
mqtt_topic: mac2mqtt/macbook-pro
discovery_prefix: homeassistant
```
说明：程序启动时会从“可执行文件所在目录”读取 `mac2mqtt.yaml`，因此两者必须在同一目录。

### 3. 后台常驻（LaunchAgent）
- 使用提供的 `com.hagak.mac2mqtt.plist` 安装到 `~/Library/LaunchAgents`
- 为避免网络未就绪时崩溃，建议在 `ProgramArguments` 中加入端口探测（已集成）：
```xml
<array>
  <string>/bin/sh</string>
  <string>-lc</string>
  <string>until /usr/bin/nc -z homeassistant.local 1883 2>/dev/null; do sleep 5; done; exec /Users/USERNAME/mac2mqtt/mac2mqtt</string>
</array>
```
- 加载：`launchctl load ~/Library/LaunchAgents/com.hagak.mac2mqtt.plist`
- 日志：`/tmp/mac2mqtt.job.out`、`/tmp/mac2mqtt.job.err`

### 4. 验证接入
- 在线状态：`mac2mqtt/macbook-pro/status/alive` 应为 `online`（保留）
- 自动发现：设备配置发布到 `homeassistant/device/macbook-pro/config`
- 在 HA 的“设备与服务”中可看到 `macbook-pro` 与相关实体

## 可选能力安装
### 显示器亮度（BetterDisplay）
- 安装：`brew install --cask betterdisplay`
- 启动 BetterDisplay 应用，在偏好设置启用 CLI 访问
- 验证 CLI：`betterdisplaycli get -identifiers`

### 媒体播放信息（media-control）
- 安装：`brew install media-control`
- 验证：`media-control get`（即使未播放也会返回 JSON）

## 常用命令
- 重载服务：`launchctl unload ~/Library/LaunchAgents/com.hagak.mac2mqtt.plist && launchctl load ~/Library/LaunchAgents/com.hagak.mac2mqtt.plist`
- 查看在线：`mosquitto_sub -h <MQTT_IP> -u <user> -P <pass> -t mac2mqtt/macbook-pro/status/alive -v`
- 查看日志：`tail -n 100 /tmp/mac2mqtt.job.err`、`tail -n 100 /tmp/mac2mqtt.job.out`

## Home Assistant 中的实体（自动发现）
- 音量滑块、静音开关
- 保持唤醒开关
- 系统按钮：睡眠/关机/显示休眠/显示唤醒/屏保
- 电池传感器
- 用户活动传感器（设备类 `occupancy`）
- 显示器亮度滑块（启用 BetterDisplay CLI 后）
- 播放/暂停按钮与“当前播放”传感器（安装 media-control 后）

## 主题与命令示例
- 状态前缀：`mac2mqtt/macbook-pro/status/#`
- 命令前缀：`mac2mqtt/macbook-pro/command/#`
- 示例：
  - 设置音量：发布到 `.../command/volume`，载荷如 `42`
  - 静音：发布到 `.../command/mute`，载荷 `true`/`false`
  - 保活：发布到 `.../command/keepawake`，载荷 `true`/`false`
  - 系统动作：发布到 `.../command/set`，载荷 `sleep`/`displaysleep`/`displaywake`/`screensaver`/`shutdown`

## 常见问题与实践建议
- 仅运行单实例：同时运行二进制与 LaunchAgent 会因重复 `clientId` 导致互踢，HA 中实体频繁“不可用”
- 缺少 `switchaudiosource`：在外置或虚拟声卡场景，`osascript` 可能返回 `missing value`，已通过 `switchaudio-osx` 回退路径处理；安装：`brew install switchaudio-osx`
- 网络未就绪：已在 LaunchAgent 增加端口探测，避免在网络不可达时刷日志或崩溃
- 自动发现前缀：保持 `homeassistant`，程序会发布保留的设备配置到对应主题

## 构建（可选）
```bash
brew install go
go mod download
go build -o mac2mqtt mac2mqtt.go
chmod +x mac2mqtt
```

## 维护与更新
- 本指南以你的实际安装过程为准，后续改动将直接提交到你的 fork：`Redwinam/mac2mqtt`
- 如需变更主题前缀或主机名，编辑 `~/mac2mqtt/mac2mqtt.yaml` 并重载服务

### Manual Configuration

If you prefer manual configuration, here's a sample:

`configuration.yaml`:

```yaml
script:
  air2_sleep:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/sleep"
          payload: "sleep"

  air2_shutdown:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/shutdown"
          payload: "shutdown"

  air2_displaysleep:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/displaysleep"
          payload: "displaysleep"

mqtt:
  sensor:
    - name: air2_alive
      icon: mdi:laptop
      state_topic: "mac2mqtt/bessarabov-osx/status/alive"

    - name: "air2_battery"
      icon: mdi:battery-high
      unit_of_measurement: "%"
      state_topic: "mac2mqtt/bessarabov-osx/status/battery"

  media_player:
    - name: "air2_media_player"
      icon: mdi:music
      state_topic: "mac2mqtt/bessarabov-osx/status/media_player"
      value_template: "{{ value_json.state }}"
      json_attributes_topic: "mac2mqtt/bessarabov-osx/status/media_player"
      json_attributes_template: "{{ {'title': value_json.title, 'artist': value_json.artist, 'album': value_json.album, 'app_name': value_json.app_name, 'duration': value_json.duration, 'position': value_json.position} | tojson }}"
      availability_topic: "mac2mqtt/bessarabov-osx/status/alive"
      payload_available: "online"
      payload_not_available: "offline"
```

## MQTT topics structure

The program is working with several MQTT topics. All topics are prefixed with `mac2mqtt` + `COMPUTER_NAME`.
For example, the topic with the current volume on my machine is `mac2mqtt/bessarabov-osx/status/volume`

`mac2mqtt` send info to the topics `mac2mqtt/COMPUTER_NAME/status/#` and listen for commands in topics
`mac2mqtt/COMPUTER_NAME/command/#`.

### PREFIX + `/status/alive`

There can be `true` or `false` in this topic. If `mac2mqtt` is connected to MQTT server there is `true`.
If `mac2mqtt` is disconnected from MQTT there is `false`. This is the standard MQTT thing called Last Will and Testament.

### PREFIX + `/status/volume`

The value ranges from 0 (inclusive) to 100 (inclusive)—the current volume of the computer.

The value of this topic is updated every 60 seconds.

### PREFIX + `/status/mute`

There can be `true` or `false` in this topic. `true` means that the computer volume is muted (no sound),
`false` means that it is not muted.

### PREFIX + `/status/battery`

The value ranges from 0 (inclusive) to 100 (inclusive) and represents the current level of the battery. Returns empty if there is no battery.

The value of this topic is updated every 60 seconds.

### PREFIX + `/status/media_player`

Contains JSON with current media player information. Only available if Media Control is installed.

Example:
```json
{
  "state": "playing",
  "title": "Song Title",
  "artist": "Artist Name",
  "album": "Album Name",
  "app_name": "Spotify",
  "duration": 180,
  "position": 45,
  "media_title": "Song Title",
  "media_artist": "Artist Name",
  "media_album": "Album Name"
}
```

States: `playing`, `paused`, `idle`

### PREFIX + `/status/media_state`

The current state of media playback: `playing`, `paused`, or `idle`.

### PREFIX + `/status/media_title`

The title of the currently playing media.

### PREFIX + `/status/media_artist`

The artist of the currently playing media.

### PREFIX + `/status/media_album`

The album of the currently playing media.

### PREFIX + `/status/media_app`

The name of the application playing media (e.g., "Spotify", "Apple Music").

### PREFIX + `/status/media_duration`

The total duration of the media in seconds.

### PREFIX + `/status/media_position`

The current position in the media in seconds.

### PREFIX + `/status/user_activity`

The current user activity state: `active` or `inactive`.

This sensor monitors system idle time and provides instant updates when user interaction is detected (mouse movement, keyboard input, etc.). The state changes to `active` immediately upon any user interaction and automatically switches to `inactive` after 10 seconds of no activity.

**Features:**
- **Instant detection**: No polling delays - activity is detected immediately
- **Automatic timeout**: Switches to inactive after exactly 10 seconds of inactivity  
- **System-level monitoring**: Uses macOS IOHIDSystem to track all user input
- **Event-driven**: Updates are published only when state changes occur
- **Home Assistant integration**: Appears as an occupancy sensor with device class `occupancy`

This is perfect for automation scenarios like:
- Turning off lights when user is away from computer
- Pausing media when user steps away
- Triggering screensaver or sleep modes
- Presence detection for home automation

The current position in the media in seconds.

### PREFIX + `/command/volume`

You can send integer numbers from 0 (inclusive) to 100 (inclusive) to this topic. It will set the volume on the computer.

### PREFIX + `/command/mute`

You can send `true` or `false` to this topic. When you send `true` the computer is muted. When you send `false` the computer
is unmuted.

### PREFIX + `/command/runshortcut`

You can send the name of a shortcut to this topic. It will run this shortcut in the Shortcuts app.

### PREFIX + `/command/set`

You can send `screensaver` to this topic. It will turn start your screensaver. Sending some other value will do nothing.

You can send `displaywake` to this topic. It will turn on the display. Sending some other value will do nothing.

You can send  `sleep` to this topic, and it will put the computer to sleep. Sending other values will do nothing.

You can send `shutdown` to this topic. It will try to shut down the computer. The way it is done depends on the user who ran the program. If the program is run by `root` the computer will shut down, but if it is run by an ordinary user the computer will not shut down if there is another user who logged in. Sending some other value but `shutdown` will do nothing.

You can send `displaysleep` to this topic. It will turn off the display. Sending some other value will do nothing.


## Management Scripts

After installation, you can use these helpful scripts to manage Mac2MQTT:

### Using Make (Recommended)
```bash
make status       # Check service status
make logs         # Show recent logs
make configure    # Reconfigure settings
make test         # Test MQTT connection
make uninstall    # Uninstall mac2mqtt
```

### Using Scripts Directly
```bash
./status.sh       # Check service status
./status.sh --logs      # Show recent logs
./status.sh --follow    # Follow logs in real-time
./configure.sh    # Reconfigure settings
./uninstall.sh    # Uninstall mac2mqtt
```

### Check Status
Shows if the service is running, configuration details, and dependency status.

### View Logs
View recent logs or follow them in real-time.

### Reconfigure Settings
Allows you to change MQTT settings without reinstalling.

### Uninstall
Completely removes Mac2MQTT from your system.

## Building

To build this program yourself, follow these steps:

1. Clone this repo
2. Make sure you have installed go, for example with `brew install go`
3. Install its dependencies with `go install`
4. Build with `go build mac2mqtt.go`

It outputs a file `mac2mqtt`. Make the binary executable (`chmod +x mac2mqtt`) and run `./mac2mqtt`.
