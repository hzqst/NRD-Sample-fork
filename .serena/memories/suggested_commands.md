# 常用命令

- 部署/生成工程：`./1-Deploy.bat` 或 `cmake -S . -B _Build`
- 构建：`./2-Build.bat` 或 `cmake --build _Build --config Release -j %NUMBER_OF_PROCESSORS%`
- 运行：`./3-Run.bat`，脚本会选择 API、分辨率、DLSS 模式和场景
- 清理：`./4-Clean.bat`
- 快速定位文件：`rg --files`
- 快速全文搜索：`rg "pattern"`

注意：用户规则要求除非明确请求，不要在完成代码或修复后主动运行 test/build 命令。