# 项目速览 — 用于 AI 编码代理的快速指南

此文件为 AI 代理在本仓库中立即上手开发所需的最小、可执行信息集合（架构、关键文件、运行/调试命令、项目约定与示例）。只记录从代码可发现的实际模式和举例。

1. 大体架构

- Android 原生应用（minSdk 16, compileSdk 26）。主要职责分离：UI 层由 `app/src/main/java/com/icodechef/android/tick/MainActivity.java` 负责，计时与后台工作由前台服务 `TickService` 处理，应用级状态机在 `TickApplication` 中维护。
- 数据持久化使用 SQLite，封装在 `app/src/main/java/com/icodechef/android/tick/database/TickDBAdapter.java` / `TickDBHelper.java`。
- 计时逻辑放在自实现的 `util/CountDownTimer.java`，声音与唤醒由 `util/Sound.java` 与 `util/WakeLockHelper.java` 管理。
- 自定义控件：`widget/TickProgressBar.java`、`widget/RippleWrapper.java`，设置偏好交互封装在 `util/SeekBarPreference.java`。

2. 关键运行 / 开发命令（可直接执行）

- 构建（在 Windows 上）：运行 `gradlew.bat assembleDebug`（仓库根目录）。
- 运行本地安装：`gradlew.bat installDebug`（需要已连接的设备或模拟器）。
- 单元测试：`gradlew.bat test`；仪器化测试（如果设备可用）：`gradlew.bat connectedAndroidTest`。
- 使用 adb 触发 Service 示例（在调试时有用）：
  - 启动计时：
    adb shell am startservice -n com.icodechef.android.tick/.TickService -a com.icodechef.android.tick.ACTION_START
  - 暂停计时：
    adb shell am startservice -n com.icodechef.android.tick/.TickService -a com.icodechef.android.tick.ACTION_PAUSE --es time_left "00:05:00"

3. 项目特有的约定与模式（重要，避免误改）

- 状态机集中在 `TickApplication`：常量 `STATE_*` 与 `SCENE_*` 控制界面可见性与业务流程，任何改变状态的逻辑应优先查阅此类实现。
- Intent action 常量在 `TickService` 中定义（例如 `ACTION_START`, `ACTION_PAUSE`, `ACTION_COUNTDOWN_TIMER`），Activity/Service 间通过发送这些 action 通信。不要硬编码字符串，引用常量。
- 通知与前台服务：`TickService` 使用 `startForeground` 与 `NotificationCompat.Builder` 更新通知，测试通知行为时请注意 `NOTIFICATION_ID = 1` 的复用与 `PendingIntent.FLAG_NO_CREATE` 用法。
- SharedPreferences key 约定：偏好键多为 `pref_key_*`，`SeekBarPreference` 通过 `getResourceEntryName(seekBar.getId())` 将控件 id 映射为存储 key（因此不要随意改动设置布局 id）。
- 数据库表为 `timer_schedule`（见 `TickDBAdapter`），插入后用 `update(id)` 标记结束时间；`getToday()` 与 `getAmount()` 有固定返回键 `times` 与 `duration`。

4. 通信与广播契约

- `TickService` 向外广播 `ACTION_COUNTDOWN_TIMER`，载荷包含 `MILLIS_UNTIL_FINISHED`（long）与 `REQUEST_ACTION`（ACTION_TICK / ACTION_FINISH / ACTION_AUTO_START），`MainActivity` 注册并响应此广播来更新 UI。
- 屏幕 on/off 事件用于控制 WakeLock（`WakeLockHelper.acquire/release`），服务在 `onCreate` 注册屏幕广播。

5. 常见修改点与示例

- 新的计时相关功能应在 `util/CountDownTimer` 与 `TickService` 中添加处理点；UI 更新逻辑放在 `MainActivity.reload()` / `update*()` 系列方法。
- 新的偏好设置：在布局里新增控件后，若使用 `SeekBarPreference`，请确保控件 id 与期望 key 一致（resource entry name 被用作 key）。
- 示例：从代码触发开始计时
  ```java
  Intent i = TickService.newIntent(context);
  i.setAction(TickService.ACTION_START);
  context.startService(i);
  ```

6. 依赖与构建注意事项

- AGP 与 SDK：根 build.gradle 注明 `com.android.tools.build:gradle:3.6.4`，模块 compileSdkVersion=26，构建通过仓库 `google(), jcenter(), mavenCentral()`。
- 资源：声音文件位于 `app/src/main/res/raw/`（例如 `ticking`, `workend`, `breakend`）。

7. 编辑器/提交注意

- 保持类的封装边界（Application 管理状态、Service 处理后台与通知、Activity 仅处理 UI），避免把持久化或唤醒逻辑移入 Activity。
- 尽量不要更改设置布局 id 或偏好键名，否则会导致偏好读写/SeekBar 映射失效。

如果以上有遗漏或需要补充的具体处，请告知我想优先补充的部分（例如更多广播示例、adb 命令、或特定文件的注释）。
