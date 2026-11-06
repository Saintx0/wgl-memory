# 手机上装「罔两 · 记忆」

**仓库已建好（公开、无简介）**：https://github.com/Saintx0/wgl-memory

手机上：扩展 → 安装扩展 → 填上面这个地址 → 刷新即可。**不需要任何 token。**

---

结论先说：**TauriTavern 桌面版不监听任何端口**（查过进程，`netstat` 里一条 LISTENING 都没有），
所以「PC 开着、手机浏览器连过来」这条路在 TauriTavern 上不成立。

手机端只有三条路，按推荐顺序：

---

## 方案一：Git 仓库（已建好，公开）

```
https://github.com/Saintx0/wgl-memory
```

**手机上（TauriTavern）：** 扩展 → 安装扩展 → 填地址 → 刷新。零配置。

**以后更新：** 电脑上 `git push`，手机上点扩展的「更新」。

仓库是公开的，但没写简介、没加 topics，不挂首页——只有拿到链接的人才会看到。
里面只有 JS 和文档，没有任何角色卡、聊天记录或密钥。

想改回私有：`gh api -X PATCH repos/Saintx0/wgl-memory -F private=true`。
已经装好的设备照常运行，只是以后「更新」会要求认证。

## 方案二：adb 直推（不建仓库，但要连一次电脑）

Android 上 Tauri 应用的数据在 app 私有目录，文件管理器进不去，`adb` 可以。

```bash
# 1. 手机开「开发者选项 → USB 调试」，插线
adb devices

# 2. 先看看目录结构（确认路径）
adb shell run-as com.tauritavern.client ls files/

# 3. 推上去
adb push wgl-memory /sdcard/Download/wgl-memory

# 4. 在手机 shell 里挪到位（如果 run-as 可用）
adb shell
run-as com.tauritavern.client sh -c 'mkdir -p files/data/extensions/third-party'
run-as com.tauritavern.client sh -c 'cp -r /sdcard/Download/wgl-memory files/data/extensions/third-party/'
exit
```

如果 `run-as` 提示不是 debuggable 包，说明这一版没开调试签名——那就退回方案一。

---

## 方案三：换成独立 SillyTavern 服务端（想在手机浏览器里玩才需要）

如果你要的是「PC 当主机，手机浏览器直接开」，那不该用 TauriTavern 桌面版，而是：

1. PC 上跑一个标准 SillyTavern（Node 版），它会监听端口
2. 把 `wgl-memory` 放到 `SillyTavern/public/scripts/extensions/third-party/wgl-memory/`
3. 手机浏览器打开 `http://电脑局域网IP:8000`

要注意两点：

- 酒馆默认开着白名单（`whitelistMode`），要在 `config.yaml` 里放行局域网段，否则会看到
  `forbidden-by-whitelist.html`
- 端口要在防火墙放行

> TauriTavern 的 Android 版本身就是原生应用，直接装它更省事，不需要这条路。

---

## 它能不能跟着数据同步走？

**不能。** TauriTavern 的「多设备同步」同步的是用户数据（角色卡、聊天、预设、世界书），
第三方扩展目录 `data/extensions/third-party/` 不在同步范围内——这也是为什么柏宝书要每家单独装一次。

所以最稳的做法还是方案一：一个仓库，多台设备各装一次，之后靠 Git 更新。

---

## 装完怎么确认

不管哪条路，验证方法一样：

1. 扩展设置里出现「罔两 · 记忆」
2. 点「预览」，看到 `【事件日志】` 之类的结构
3. 状态行显示 `setExtensionPrompt · 注入 N 字 · 第 N 轮已收`

如果状态显示「未连接」，把控制台里 `[罔两·记忆]` 开头那行发出来。

---

## 一个替代思路

手机端如果只想临时用，可以把 `index.js` 的内容整个复制进**酒馆助手的脚本库**（你用着 4.11.3）。
酒馆助手支持全局脚本，会在前端执行，也能调 `setExtensionPrompt`——但 `document.getElementById('extensions_settings2')`
那种面板写法要改成它自己的 UI 才能显示设置项。

这条路我还没验证过，先给方案一，稳定。
