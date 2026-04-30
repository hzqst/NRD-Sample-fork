# 项目概览

NRD Sample 是 NVIDIA NRD/NRI 相关的实时路径追踪示例与参考实现，重点展示 NRD denoising、DLSS-RR/上采样、SHARC 缓存、透明/玻璃渲染等实时游戏路径追踪实践。

技术栈：C++17、C99、CMake 3.30+、NRI（D3D12/Vulkan 抽象）、NRD、NRIFramework、HLSL shaders、Packman 资源下载。主要目录包括 `Source/`、`Shaders/`、`External/`、`Tests/`、`_Data/`、`_Shaders/`、`_Bin/`、`_Build/`。