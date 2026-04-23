# Babyrev

## 题目结构

rev = reverse engineering，这是逆向题。**不提供 `Challenge.sol` 源码**。只能从字节码入手。

## 漏洞类型

**信息暴露 (Information Disclosure)** / **硬编码密钥 (Hardcoded Credentials)**

它的“漏洞”在于，合约的字节码中直接包含了**明文形式的 Flag**。

## 漏洞原理

在无法获取源码的情况下，只能通过逆向分析合约的 EVM 字节码来寻找解题线索。

这个“漏洞”的本质是**敏感信息明文存储**。由于区块链上所有合约的字节码都是公开的，任何人都能直接提取并读取其中的字符串常量。

## 目标

查看 `Setup.sol`：

```solidity
contract Setup {
    Challenge public challenge;

    constructor() {
        challenge = new Challenge();
    }

    function isSolved() public view returns (bool) {
        return challenge.solved();
    }
}
```
目标：让 challenge.solved() 返回 true。

## 1. 获取 Challenge 字节码

Challenge 合约的字节码在 `public/deploy/compiled.bin` 中：

```bash
cd ~/paradigm-ctf-2021/babyrev/public/deploy
ls -la
# 看到 compiled.bin
```
compiled.bin 是 JSON 文件，包含所有编译后的合约信息。

## 2. 提取 Challenge 字节码

```bash
# 安装 jq（JSON 解析工具）
sudo apt install jq -y

# 提取 Challenge 合约的字节码
cat compiled.bin | jq -r '.contracts["/private//Challenge.sol:Challenge"].bin' > challenge.hex

# 查看前 100 个字符
head -c 100 challenge.hex
```
## 3. 反汇编字节码
使用 pyevmasm 进行反汇编：

```bash
pip install pyevmasm
```
创建反汇编脚本
```python
import sys
import pyevmasm

with open('challenge.hex', 'r') as f:
    code = f.read().strip()

if code.startswith('0x'):
    code = code[2:]
code = code.replace('\n', '').replace(' ', '')

if len(code) % 2 == 1:
    code += '0'

for insn in pyevmasm.disassemble_all(bytes.fromhex(code)):
    print(f"{insn.pc:04x}: {insn}")
```
运行反汇编
```bash
python disasm.py > challenge.opcodes
```
查看前 50 行
```bash
head -50 challenge.opcodes
```
输出：
```text
0000: PUSH1 0x80
0002: PUSH1 0x40
0004: MSTORE
0005: CALLVALUE
0006: DUP1
0007: ISZERO
0008: PUSH2 0x10
000b: JUMPI
...
```

## 4. 在字节码中寻找字符串
在反汇编过程中，我发现字节码中包含可读的字符串。用 xxd 将十六进制转换后查看：
```bash
cat challenge.hex | xxd -r -p | strings | grep -i "PCTF"
```
输出：  
PCTF{v32y_53cu23_3nc2yp710n_4190217hm}

## FLAG
PCTF{v32y_53cu23_3nc2yp710n_4190217hm}

## Lessons Learned


| 误区 | 正确做法 |
|------|----------|
| 以为必须深入逆向整个字节码 | 先试试提取可读字符串 |
| 以为需要理解复杂算法 | flag 可能直接明文存储 |
| 以为必须写攻击脚本 | 简单的命令行工具往往最有效 |


| 问题 | 解决方案 |
|------|----------|
| 没有 `Challenge.sol` 源码 | 字节码在 `compiled.bin` 中 |
| 不知道如何提取字节码 | 用 `jq` 解析 JSON |
| 字节码太长无法直接阅读 | 用 `pyevmasm` 反汇编 |
| 不知道如何找 flag | 用 `xxd -r -p` 转二进制后用 `strings` 提取 |

## 核心教训

> **在区块链上，任何存储在合约字节码中的数据都是公开的。** 永远不要将密码、私钥或 Flag 等敏感信息以明文形式放入合约代码中，即使它经过了看似复杂的算法处理。
