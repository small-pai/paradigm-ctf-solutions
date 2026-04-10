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
## Flag
PCTF{placeholder}
