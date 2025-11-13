# 模式二远程图片同步功能设计

## 背景
- Waveshare PhotoPainter 固件在启动时根据 `PhotPainterMode` 决定进入哪种模式，其中 `0x02` 会调用 `User_Network_mode_app_init()` 作为“网络模式”入口【F:main/main.cc†L32-L69】。
- 现有模式二提供热点 + 网页上传图片能力，需要替换为“自动拉取 HTTPS 图片并展示”的新流程。
- 模式一已经具备 SPI 墨水屏展示能力，模式三的图片生成流程已经实现 TF 卡文件写入、事件组同步等基础设施，可作为文件落地与任务协同的参考【F:main/boards/waveshare-s3-PhotoPainter/esp32-s3-PhotoPainter.cc†L59-L118】。

## 新功能目标
1. 进入模式二后，设备通过 HTTPS 接口获取远端最新图片序列号（例如 `25.bmp`）。
2. 对比 TF 卡上现有最大序号，若云端更新，则从指定 HTTPS 源下载对应图片，支持断点续传并写入 TF 卡。
3. 将下载完成的图片交由模式一的显示能力渲染到 SPI 墨水屏。
4. 删除原有热点启动、网页上传等逻辑。

## 整体流程
1. **模式入口改造**
   - 在 `User_Network_mode_app_init()` 中移除热点相关调用，彻底删除原有 AP/Web 上传模块。
   - 进入模式二后首先通过 `TfCardImageStore`（或新增工具函数）打开 TF 卡根目录下 `/wifi.txt`，按 UTF-8 读取首行 SSID、次行密码，并写入 Wi-Fi STA 初始化配置；若文件缺失或格式错误，则在屏幕提示并终止后续流程。
   - 成功联网后构造并启动“远程图片同步”任务。

2. **远程序列号获取**
   - 新增 `RemoteImageSync`（暂定名）模块，封装 HTTPS GET 请求，请求链接 1（需求给定），链接常量定义在新的头文件（例如 `remote_image_endpoints.h`）中，通过可修改的 `constexpr char kRemoteIndexUrl[]` 对外暴露。
   - 响应体仅包含文件名（如 `25.bmp`）。解析出数值部分 `25` 作为最新序号。
   - 将序列号与 TF 卡本地记录进行对比（可读取 TF 卡目录或维护 `latest_seq.txt`）。

3. **断点续传下载**
   - 若远端序列号更大：
     - 以 `esp_http_client` 配置 HTTPS + 证书校验，构造带 `Range` 头的下载流程，支持从 `.part` 临时文件恢复。
     - 下载链接按照序列号拼接（需求给定的 HTTPS 地址 2），同样通过头文件中的 `constexpr char kRemoteImageBaseUrl[]` 存储，方便后续运维修改。
     - 数据块落盘后立即 `fwrite` 到 TF 卡，周期性 `fsync`，避免断电损坏。
     - 下载完成后将 `.part` 重命名为 `{seq}.bmp` 并更新本地最新序号。
   - 如果远端序列号未更新，直接进入显示阶段。

4. **显示触发**
   - 下载成功后调用模式一中用于显示 SD/TF 图片的公共接口（例如现有 Mode1 图片浏览/刷新函数）。
   - 复用显示接口时，确保在墨水屏刷新完成后显式调用“进入休眠”API（模式一已有封装时直接复用，没有则在公共层新增 `EpdEnterSleep()`），避免长时间保持刷新电压导致面板损坏。
   - 若模式一接口以事件组驱动，可仿照模式三中 `xEventGroupSetBits(...)` 的做法，将新图片加入展示队列【F:main/boards/waveshare-s3-PhotoPainter/esp32-s3-PhotoPainter.cc†L65-L118】。

5. **错误处理与重试**  
   - 处理 HTTPS 请求失败、证书异常、TF 卡写入错误等情况：
     - 记录日志并在 `RemoteImageSync` 中实现指数退避重试。
     - 将失败状态通过显示层提示（例如调用 `Display::SetStatus()` 更新提示语）。
   - 若下载过程中网络断开，保存当前已下载字节并在网络恢复时自动续传。

## 模块拆分与任务
| 模块 | 主要职责 | 备注 |
| --- | --- | --- |
| `User_Network_mode_app_init()` | 模式二入口：联网、启动同步任务、对接显示层 | 参考 `main/main.cc` 中的调用点保持接口不变【F:main/main.cc†L61-L67】 |
| `RemoteImageSync` (新建) | 远程序列号获取、下载调度、状态机 | 建议放置在 `main/boards/waveshare-s3-PhotoPainter/` 专用实现中，以便访问模式三的共享资源，并内置读取 `/wifi.txt` 的辅助函数 |
| `HttpRangeDownloader` (新建) | 基于 `esp_http_client` 的断点续传下载器 | 可封装成独立类，供后续 OTA 或其他功能复用 |
| `TfCardImageStore` (复用/改造) | TF 卡路径管理、序列号记录、文件落地 | 如模式三已有相应封装，直接复用或抽取公共函数 |
| Display Hook | 新图像显示触发 | 复用模式一展示函数或事件 |

## 数据与状态管理
- **远端状态**：最新序列号、下载 URL 由远程接口返回；建议缓存于 RAM，并在成功展示后持久化于 NVS（键如 `mode2_latest_seq`）；对应的 HTTPS 端点常量在头文件中维护，便于运营修改。
- **本地状态**：
  - TF 卡根目录建议固定为 `/sdcard/pictures/`；
  - `/wifi.txt` 保存 STA SSID/密码，采用 UTF-8 编码，读取失败时需在显示层提示；
  - 维护 `latest_seq.txt` 或通过扫描目录确定最大序号；
  - 下载中间态使用 `{seq}.bmp.part` 命名，并在恢复时检查文件长度以设置 Range 起点。
- **同步信号**：沿用模式三的事件组/信号量设计，如 `ai_IMG_Group`、`ai_img_while_semap` 用于通知显示线程更新资源，确保模式二与模式三不会互相干扰【F:main/boards/waveshare-s3-PhotoPainter/esp32-s3-PhotoPainter.cc†L59-L118】。

## 待办清单
1. 重构 `User_Network_mode_app_init()`：删除热点 AP、网页服务初始化；从 `/wifi.txt` 解析 UTF-8 SSID/密码并完成 Wi-Fi STA 启动。
2. 新建 `RemoteImageSync`：包含序列号请求、差异判断、断点续传下载、状态持久化，以及重试策略。
3. 在同步完成时调用模式一的显示函数或事件，保证图片立即出现在墨水屏上，并在刷新结束后执行墨水屏休眠。
4. 新增可配置的远端链接头文件，暴露索引和图片基础 URL 常量供维护调整。
5. 更新证书存储与异常提示逻辑，确保 HTTPS 请求合法通过。

## 验收标准
- 进入模式二后设备自动联网并请求最新序列号，联网凭据来源于 TF 卡 `/wifi.txt`。
- 远端有新图片时能够完整下载到 TF 卡；过程中断网重新上电仍可续传成功。
- 下载完成后图片会自动在墨水屏显示，刷新结束后可确认墨水屏进入休眠状态，且原热点/Web UI 不再可用。
- 多次进入模式二时仅同步增量图片，不重复下载已存在文件。
- 远端索引与图片 URL 常量通过头文件可配置，修改后无需触及核心逻辑即可生效。
