# wcdb-key-tool

Linux 微信数据库密钥提取工具。通过 ELF 静态分析自动适配新版本，无需每次更新手动逆向。

Extract WeChat (WCDB/SQLCipher4) database encryption keys on Linux. Auto-adapts to new versions via ELF static analysis — no manual reverse engineering needed per update.

## 背景 / Background

微信 4.1+ 版本不再在进程内存中缓存明文数据库密钥（raw key），导致所有基于内存模式扫描（`x'<hex>'`）的开源工具全部失效。

本工具通过以下创新方法解决这一问题：

1. **ELF 静态分析**：自动分析微信二进制文件，定位密钥写入函数的断点地址（无需手动逆向每个版本）
2. **GDB 断点捕获**：在微信登录时设置断点，从 CPU 寄存器读取 32 字节 passphrase
3. **PBKDF2 派生**：将 passphrase 通过 PBKDF2-SHA512（256,000 次迭代）派生为每个数据库的加密密钥

WeChat 4.1+ no longer caches raw database encryption keys in process memory, breaking all existing open-source tools that rely on `x'<hex>'` pattern scanning.

This tool solves it with:
1. **ELF static analysis** — auto-locates the key-writing function's breakpoint address (no per-version reverse engineering)
2. **GDB breakpoint capture** — reads the 32-byte passphrase from CPU registers during WeChat login
3. **PBKDF2 derivation** — derives per-database encryption keys via PBKDF2-SHA512 (256K iterations)

## 安装 / Install

```bash
# 无需安装，单文件即可运行
# No installation needed — single file tool
sudo apt install gdb  # 唯一依赖
```

## 使用 / Usage

```bash
# 提取密钥（首次需要重新登录微信）
sudo python3 wcdb_key_tool.py extract

# 提取 + 解密一步完成
sudo python3 wcdb_key_tool.py extract --decrypt

# 仅解密（已有密钥文件）
sudo python3 wcdb_key_tool.py decrypt
```

### 首次使用流程 / First-time Setup

1. 确保微信 Linux 已启动并登录
2. 运行 `sudo python3 wcdb_key_tool.py extract`
3. 工具会提示你在微信中**退出登录并重新登录**
4. 登录完成后密钥自动捕获并保存
5. 后续运行无需重复此步骤（密钥已缓存）

## 技术原理 / How It Works

### 为什么现有工具失效？

微信 4.0.x 的 WCDB 在进程内存中以 `x'<64hex_key><32hex_salt>'` 格式缓存 raw key。所有开源工具（wechat-decrypt、wx-cli 等）通过扫描进程内存匹配这个模式来提取密钥。

微信 4.1+ 改变了机制：**内存中存储的不再是 raw key，而是 passphrase**。passphrase 需要经过 PBKDF2-SHA512（256,000 次迭代）才能派生出实际的加密密钥。现有工具找到的 hex 模式对应的 salt 与数据库文件的 salt 不匹配，因此全部失败。

### 本工具的解决方案

1. **ELF 静态分析**：在微信二进制的 `.rodata` 节中搜索 `com.Tencent.WCDB.Config.Cipher` 字符串，通过 LEA 指令的交叉引用追踪到密钥处理函数的入口地址
2. **GDB 断点**：在该函数入口设置断点，等待微信登录时触发，从 `$rsi` 寄存器读取 passphrase
3. **PBKDF2 派生**：对每个数据库文件，使用其 16 字节 salt + passphrase 执行 PBKDF2-SHA512（256K 迭代），得到 32 字节 AES-256 密钥
4. **HMAC 验证**：用派生出的密钥验证数据库第一页的 HMAC-SHA512，确认密钥正确

### 安全说明 / Security Notes

- 此工具仅用于提取用户**自己设备上自己微信账号**的数据库密钥
- GDB 在登录时短暂附加（<2 秒），读取一个寄存器值后立即 detach
- 不修改微信的任何行为，不接触网络协议
- 不会触发封号（详见 FAQ）

## 兼容性 / Compatibility

| 平台 | 微信版本 | 状态 |
|------|---------|------|
| Linux x86_64 | 4.1.1.4 | ✅ 已验证 |
| Linux x86_64 | 4.1.0.x | ✅ 应兼容（ELF 分析自动适配） |
| Linux x86_64 | 4.0.x | ✅ 兼容（内存扫描 fallback） |
| macOS | - | ❌ 暂不支持 |
| Windows | - | ❌ 暂不支持 |

## FAQ

**Q: 会不会封号？**  
A: 不会。工具只在登录瞬间读取内存寄存器值，整个过程 <2 秒，不修改任何程序行为，不接触微信服务器通信。

**Q: passphrase 存在哪里？**  
A: 存储在 `~/.wcdb-key-tool/wechat-passphrase.json`（权限 600），仅当前用户可读。

**Q: 为什么需要 sudo？**  
A: GDB 需要 `ptrace` 权限来附加到其他进程的内存空间。你也可以用 `echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope` 临时放开权限（重启后恢复）。

**Q: 微信更新后还能用吗？**  
A: 大概率可以。ELF 静态分析通过字符串交叉引用定位函数，只要微信继续使用 WCDB 的 `com.Tencent.WCDB.Config.Cipher` 字符串，就能自动适配。

## 致谢 / Credits

- [kkocdko](https://kkocdko.site/post/202510212134) — GDB 断点法的原始思路
- [wxchat-export](https://github.com/lopleec/wxchat-export) — ELF 静态分析方法
- [ylytdeng/wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt) — 内存扫描基础代码
- [CloudDreamAI](https://github.com/TANGandXUE) — PBKDF2 派生发现 + 完整集成

## License

MIT — see [LICENSE](LICENSE)
