# Babycrypto

## 题目描述
一个签名预言机服务，可以用固定的 `session_secret`（即 ECDSA 中的 k 值）为任意消息签名。目标是利用 k 值重用漏洞恢复私钥，然后对挑战消息生成有效签名。

## 漏洞类型
ECDSA k 值重用（Nonce Reuse Attack）

## 漏洞原理
ECDSA 签名公式：
- r = (k * G).x mod n
- s = k^(-1) * (z + r * d) mod n

当同一个 k 用于签名两个不同消息时：  
k = (z1 - z2) * (s1 - s2)^(-1) mod n  
d = (s1 * k - z1) * r^(-1) mod n

## 攻击原理
`chal.py` 中使用固定的 `session_secret` 作为 ECDSA 签名的 k 值，导致可以从两个不同消息的签名中恢复私钥。

## 攻击步骤

1. 获取签名预言机返回的 4 组签名数据
2. 使用前两个签名 (AAAA, BBBB) 恢复 k 值和私钥
3. 计算挑战消息的签名
4. 提交签名获取 flag

## 解题步骤

### 1. 运行服务
```bash
cd ~/paradigm-ctf-2021/babycrypto/public
source venv/bin/activate
python deploy/chal.py
```

 ### 2. 获取签名数据
 依次输入：  
 1  
secret  
AAAA  
BBBB  
CCCC  
DDDD  

记录输出中的：

AAAA 的 (r, s)

BBBB 的 (r, s)

test= 的值

 ### 3. 计算私钥和签名

```python
from ecdsa import SECP256k1
from ecdsa.numbertheory import inverse_mod
import hashlib

def hash_message(msg):
    k = hashlib.sha3_256()
    k.update(msg.encode())
    d = k.digest()
    n = int.from_bytes(d, 'big')
    olen = SECP256k1.order.bit_length()
    return n >> max(0, len(d)*8 - olen)

# 填入数据
r1, s1 = 0x..., 0x...  # AAAA
r2, s2 = 0x..., 0x...  # BBBB
z3 = 0x...             # test

z1 = hash_message("AAAA")
z2 = hash_message("BBBB")
n = SECP256k1.order

k = ((z1 - z2) % n) * inverse_mod((s1 - s2) % n, n) % n
d = (((s1 * k) % n - z1) % n) * inverse_mod(r1, n) % n

g = SECP256k1.generator
r = (k * g).x() % n
s = inverse_mod(k, n) * ((z3 + d * r) % n) % n

print(f"signature r = {hex(r)}")
print(f"signature s = {hex(s)}")
```
 ### 4. 提交签名
在 r? 后输入计算出的 r（去掉 0x 前缀）  
在 s? 后输入计算出的 s  

 ### 5. 获取 flag
输出 PCTF{placeholder}

## 关键代码

```python
# 恢复 k
k = ((z1 - z2) % n) * inverse_mod((s1 - s2) % n, n) % n

# 恢复私钥
d = (((s1 * k) % n - z1) % n) * inverse_mod(r1, n) % n

# 计算挑战签名
r = (k * G).x() % n
s = inverse_mod(k, n) * ((z3 + d * r) % n) % n
```

## Lessons Learned

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `chal.py` 运行时报 `No module named 'sha3'` | Python 3.12 不兼容 `pysha3` | 改用 `hashlib.sha3_256()` |
| 用 `socat` 转发后 `solve.py` 卡住 | 时序问题，缓冲区没对齐 | 放弃自动化，改为手动获取数据 + 脚本计算 |
| 在 `r?` 后面输入了 `CCCC` | 误解了提示 | `r?` 需要输入计算出的签名值，不是消息内容 |
| 每次运行数据都不一样 | 每次运行生成新的随机密钥 | 每次都要重新计算，不能复用上次的结果 |
| `solve.py` 只输出了 AAAA 和 BBBB 就卡住 | `recvuntil(b'message? ')` 没有匹配到 | 服务器在发送完签名后不立即发送 `message?` 提示 |

### 关键理解

1. **k 值重用的特征**：所有签名的 r 值相同，因为 `r = (k * G).x`
2. **为什么用 AAAA/BBBB**：任意消息都可以，只要知道原文就能计算哈希 z
3. **手动操作比自动化更可靠**：对于简单交互，手动 `nc` + Python 脚本计算比 pwntools 全自动更稳定
4. **每次运行 `chal.py` 都会生成新的随机密钥**：所以不能复用之前的数据

### 学到的知识点

- ECDSA 签名原理和 k 值重用漏洞
- `inverse_mod` 计算模逆元
- Python 3.12 中 `hashlib.sha3_256()` 替代 `pysha3`
- `socat` 将脚本暴露为网络服务
- 虚拟环境避免 `externally-managed-environment` 错误

## Flag
PCTF{placeholder}

### 关于 Flag 占位符

在 `chal.py` 中，flag 通过 `os.getenv("FLAG", "PCTF{placeholder}")` 获取。
- 本地运行时：没有 `FLAG` 环境变量 → 输出 `PCTF{placeholder}`
- 比赛环境：会设置真正的 `FLAG` 环境变量 → 输出真实 flag

