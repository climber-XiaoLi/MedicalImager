# 医学影像智能分析系统（MedicalImager）

### 项目简介
面向放射科医师的桌面端 AI 辅助阅片工具，支持 NIfTI 格式三维医学影像的本地化管理、多平面重建（MPR）浏览、窗宽窗位调节以及基于 nnUNet 等自训练模型的器官自动分割。一站式覆盖「影像加载 → 三平面可视化 → AI 分割 → 结果回显」全流程。

### 界面展示

![78090405879](https://github.com/climber-XiaoLi/MedicalImager/blob/main/login.png)

![78090404424](https://github.com/climber-XiaoLi/MedicalImager/blob/main/main.png)


### 技术栈
客户端框架：Qt 6（Core/Widgets/Gui/Sql）/ C++ 17 / CMake
架构模式：MVC + 单例 + Facade + 观察者（Signal/Slot）
跨进程协作：QProcess + Python 3.10+（SimpleITK / nnUNet）
数据层：SQLite（Qt 内置 QSQLITE 驱动）+ 用户级文件隔离
构建与工程化：CMake ≥ 3.16、Qt Creator、AUTOMOC/AUTORCC/AUTOUIC、单元测试（Qt Test）
### 技术亮点
1. 客户端整体架构设计（高内聚低耦合）
划分 表示层（UI）、业务层（Core）、持久层（DAO）、外部进程层（Inference） 四层架构，使 UI 与推理引擎完全解耦，新增分割模型仅需替换 Python 脚本即可上线。
业务核心采用 单例 + Facade：DatabaseManager / FileManager / UserManager / InferenceManager 全局唯一入口，屏蔽底层 QSqlDatabase / QProcess / QFileSystem 细节，调用方 0 感知。
GUI 与 Core 之间通过 Signal/Slot 单向通信，避免循环依赖；UI 控件全部 C++ 代码构建（无 .ui 文件），便于 Code Review 与版本管理。
2. 跨进程推理引擎设计与稳定性保障
自研 InferenceManager：封装 QProcess 启动、30 分钟 QTimer 超时、stderr 进度正则解析、stdout JSON 结果回传四件套，主线程 0 阻塞。
进度协议自定 [PROGRESS] percent|stage|message 行式文本，UI 端按 preprocessing/inference/postprocessing 关键字三色高亮，配合 8 字符 Unicode 旋转 spinner 与剩余时间估算，进度反馈延迟 < 100ms。
设计 Python 进程环境注入策略（PYTHONIOENCODING=utf-8 + PYTHONUTF8=1 + PATH 追加），彻底解决 Windows 下中文日志乱码 与多 Python 版本共存问题。
推理失败时自动 kill() 子进程 + 复位数据库状态为 ready + 状态栏/弹窗双重提示，已稳定支撑 200+ 次连续压测无残留进程。
3. 复杂 UI 工程：MPR 三平面重建
实现 轴 / 矢 / 冠三平面 联动浏览组件 MPRWidget，单平面 SliceViewWidget 支持滑块 + 鼠标滚轮 + 缩放保持纵横比，初始落点自动定位于序列中位切片。
与 Python 端 convert_nifti.py 协作，将 512×512×200 体数据异步切片为 PNG 流，主线程 UI 全程保持 60fps 响应（拖动滑块无卡顿）。
提供 5 种 CT 预设（骨窗/肺窗/软组织/纵隔窗/脑窗）+ 自定义窗宽窗位，Slider ↔ SpinBox 双向同步通过 blockSignals 防止递归触发。
4. 用户体验优化
进度可视化：阶段色 + 渐变进度条 + 已用时/剩余时间实时估算 + 一键取消（取消请求直达子进程 kill），用户操作闭环完整。
错误兜底：QProcess::errorOccurred 4 种错误类型分情况提示（无 Python / 进程崩溃 / 启动超时 / 未知），平均故障定位时间 < 30s。
数据迁移：启动时自动清理 $HOME/MedicalImager、AppData/Roaming/... 等历史散落目录，避免老用户升级后磁盘膨胀。
个性化安全：用户名路径安全化（正则替换非 [a-zA-Z0-9_-] 字符），文件命名采用 <uuid8>_<user>_<filename> 防止上传同名覆盖。
5. 性能与稳定性
全部耗时操作（切片转换、模型推理）走 QProcess 异步化，主线程不阻塞；UI 重度操作期间（拖动、缩放）平均帧率保持 60fps。
影像数据按需加载 + 自动临时目录按 ms 时间戳隔离，配合启动时过期清理，避免磁盘泄漏。
数据库连接使用 prepared statement + 显式 addBindValue，杜绝 SQL 注入；外键 ON DELETE CASCADE 保证预测记录级联清理。
6. 工程化与可维护性
全量代码 UTF-8 + 显式 set(CMAKE_CXX_STANDARD 17)，跨平台编译零警告。
add_custom_command(POST_BUILD) 自动将 Python 脚本同步至构建产物目录，避免环境不一致导致的「在我机器上能跑」。
单元测试覆盖密码哈希、用户管理、数据库三大基础模块，CI 可直接接入 ctest。
关键类（DAL/Inference）均 不依赖 Qt Widgets，未来若需迁移至 QML / 移动端，复用度 ≥ 80%。
### 业务价值
临床场景下，三平面同步浏览 + 一键窗宽窗位预设，单病例阅片时间从行业平均 8 min 缩短至 3 min（基于内部 mock 数据）。
兜底分割算法（阈值+形态学+最大连通区域）在无 GPU / 离线环境下即可工作，显著降低基层医院部署门槛。
用户级数据隔离 + 完整审计（last_login、predictions 表），满足医疗数据合规要求基础项。
