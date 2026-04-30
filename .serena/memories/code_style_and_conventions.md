# 代码风格与约定

- 语言标准：C++17 / C99。
- 格式化：仓库包含 `.clang-format`，基于 Google 风格，4 空格缩进，指针左结合，`ColumnLimit: 0`，大括号 Attach。
- C++ 命名风格从现有代码看以 PascalCase 类型/函数、`m_` 成员变量、枚举值和常量混合使用为主。
- 项目大量使用 NRI/NRD 结构体、descriptor、pipeline、barrier 与 command buffer API，应优先沿用现有封装和资源生命周期模式。