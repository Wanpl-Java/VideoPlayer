# `.replay.gz` 倍速播放 — 改动说明

> 版本：0.6.0  
> 日期：2026-07-03  
> 目标：为 Gua 协议录像（`.replay.gz` / `.part.gz`）增加倍速播放能力

---

## 一、功能概述

| 项目 | 说明 |
|------|------|
| 影响格式 | `.replay.gz`、`.part.gz`（RDP/VNC 等 Gua 协议录像） |
| 倍速选项 | 0.75x / 1x / 1.5x / 2x |
| 操作入口 | Gua 播放器控制栏右侧下拉框 |
| 倍速范围 | 底层限制 0.25x ~ 4x（UI 仅暴露常用档位） |

MP4、`.cast.gz` 播放器未改动，原有倍速能力保持不变。

---

## 二、改动文件清单

| 文件 | 类型 | 说明 |
|------|------|------|
| `patches/guacamole-common-js-jumpserver+1.1.0-c.patch` | 新增 | Guacamole 库倍速补丁 |
| `src/views/guaPlayer/index.vue` | 修改 | 倍速 UI 与交互逻辑 |
| `src/lang/modules/zh.ts` | 修改 | 中文文案 |
| `src/lang/modules/en.ts` | 修改 | 英文文案 |
| `package.json` | 修改 | 依赖、postinstall、Windows 打包配置 |

---

## 三、核心改动：Guacamole 库补丁

**依赖：** `guacamole-common-js-jumpserver@1.1.0-c`  
**补丁文件：** `patches/guacamole-common-js-jumpserver+1.1.0-c.patch`

### 3.1 新增内部变量

```javascript
var playbackSpeed = 1; // 默认正常速度
```

### 3.2 修改帧调度逻辑（`continuePlayback`）

将帧间真实等待时间按倍速缩放：

```javascript
// 修改前
var nextRealTimestamp = next.timestamp - startVideoTimestamp + startRealTimestamp;

// 修改后
var nextRealTimestamp = (next.timestamp - startVideoTimestamp) / playbackSpeed + startRealTimestamp;
```

### 3.3 新增公开 API

```javascript
// 设置倍速（0.25 ~ 4），播放中切换会重新校准时间轴
recording.setSpeed(rate);

// 获取当前倍速
recording.getSpeed();
```

**播放中切换倍速的处理逻辑：**

1. 取消当前待执行的 seek 定时器（`abortSeek()`）
2. 以当前帧时间戳重置 `startVideoTimestamp`
3. 以当前时刻重置 `startRealTimestamp`
4. 调用 `continuePlayback()` 继续播放

---

## 四、Gua 播放器 UI 改动

**文件：** `src/views/guaPlayer/index.vue`

### 4.1 模板

控制栏右侧新增倍速下拉框（`n-select`）：

```vue
<n-select
  v-model:value="playbackSpeed"
  :options="speedOptions"
  size="small"
  class="speed-select"
  :disabled="isProcessing"
  @update:value="handleSpeedChange"
/>
```

### 4.2 脚本

```typescript
const playbackSpeed = ref(1);

const speedOptions = [
  { label: '0.75x', value: 0.75 },
  { label: '1x', value: 1 },
  { label: '1.5x', value: 1.5 },
  { label: '2x', value: 2 }
];

const handleSpeedChange = (speed: number) => {
  if (!recording?.setSpeed) return;
  recording.setSpeed(speed);
};
```

组件卸载时将 `playbackSpeed` 重置为 `1`。

### 4.3 样式

- `.controls-right` 宽度由 `200px` 调整为 `280px`，增加 `gap: 8px`
- 新增 `.speed-select { width: 88px; }`

---

## 五、国际化

| Key | 中文 | 英文 |
|-----|------|------|
| `playbackSpeed` | 播放速率 | Playback Speed |

> 当前下拉框选项为固定 label（如 `1.5x`），i18n key 已预留供后续扩展。

---

## 六、工程配置改动（`package.json`）

### 6.1 脚本

```json
"postinstall": "patch-package"
```

`yarn install` / `npm install` 后自动应用 Guacamole 补丁。

### 6.2 开发依赖

```json
"patch-package": "^8.0.1"
```

### 6.3 Windows 打包配置（构建 exe 时调整）

```json
"win": {
  "sign": null,
  "signAndEditExecutable": false,
  "target": {
    "target": "nsis",
    "arch": ["x64"]
  }
}
```

| 配置项 | 原因 |
|--------|------|
| `sign: null` | 无代码签名证书，跳过签名 |
| `signAndEditExecutable: false` | 避免 Windows 无符号链接权限导致 winCodeSign 解压失败 |
| `arch: ["x64"]` | 仅打 64 位包（原含 ia32 + x64） |

---

## 七、构建产物

```
release/JumpServerVideoPlayer-0.6.0-x64.exe
```

- 平台：Windows x64
- 大小：约 82 MB
- 含倍速播放功能

---

## 八、开发与构建命令

```bash
# 安装依赖（自动打补丁）
npm install --legacy-peer-deps

# 本地开发
npm run dev

# 仅构建 Windows exe
npx vite build
npx electron-builder --win --x64
```

---

## 九、已知限制

1. **音频同步**：RDP 录像若含音频，倍速后可能不同步（多数场景无音频，影响有限）
2. **高倍速性能**：2x 以上帧渲染更密集，长录像可能出现 CPU 占用升高
3. **补丁维护**：升级 `guacamole-common-js-jumpserver` 版本后需重新验证并更新 patch 文件
4. **未签名安装包**：Windows 可能提示「未知发布者」，需手动允许运行

---

## 十、Git 提交建议

提交时请包含以下文件：

```
patches/guacamole-common-js-jumpserver+1.1.0-c.patch
src/views/guaPlayer/index.vue
src/lang/modules/zh.ts
src/lang/modules/en.ts
package.json
package-lock.json   # 若使用 npm
```

`node_modules/` 无需提交，补丁由 `postinstall` 自动应用。
