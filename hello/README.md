
# Hello

## 目标
调用 `Hello.sol` 中的 `solve()` 函数，将 `solved` 状态变量改为 `true`。

## 核心代码
```solidity
contract Hello {
    bool public solved = false;
    function solve() public { solved = true; }
}
```

## 解题思路
1. 获取 `Hello` 合约地址
2. 调用 `solve()` 函数
3. 验证 `solved == true`

## 攻击脚本（Hardhat）
```javascript
const hello = await ethers.getContractAt("Hello", helloAddress);
await hello.solve();
```
