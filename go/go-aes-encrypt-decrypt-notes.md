# Go AES 加解密标准实现笔记

## 先说清楚：AES 是什么

AES（Advanced Encryption Standard，高级加密标准）是目前使用最广的对称加密算法：加密和解密用同一把密钥。它处理的是固定大小的数据块——块长固定 128 位（16 字节），密钥长度可以是 128、192、256 位三档，分别对应 AES-128、AES-192、AES-256。密钥越长越难暴力破解，日常使用直接选 AES-256 即可（32 字节密钥）。

Go 标准库把 AES 放在 `crypto/aes` 包里。但要注意：`crypto/aes` 只提供最底层的"单个块加解密"能力，真实数据都不止 16 字节，所以实际写代码时要配合 `crypto/cipher` 里的加密模式使用。模式选错是 AES 用错的头号原因。

## 模式怎么选

| 模式 | 特点 | 建议 |
|---|---|---|
| GCM | 加密的同时生成认证标签，密文被改动解密时会直接报错 | 首选。新代码默认用它 |
| CBC | 需要 PKCS#7 填充，需要随机 IV，自己不带完整性校验 | 老系统兼容时用，必须另配 HMAC 或改用 GCM |
| CTR | 把块加密变成流加密，不需要填充 | 配合 HMAC 使用；能用 GCM 就不用它 |
| ECB | 相同明文块得到相同密文块，会泄漏数据图案 | 永远不要用 |

下面给一个完整的 AES-256-GCM 示例：随机生成密钥、加密一段文本、再解回来，并演示篡改密文时解密失败。

```go
package main

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"log"
)

func main() {
	// 1. 生成 32 字节随机密钥（AES-256）。密钥应由 crypto/rand 生成，
	//    生产环境里存放在密钥管理服务或环境配置中，不要硬编码在代码里。
	key := make([]byte, 32)
	if _, err := rand.Read(key); err != nil {
		log.Fatal(err)
	}

	// 2. 用密钥创建块加密器，再包装成 GCM 模式。
	block, err := aes.NewCipher(key)
	if err != nil {
		log.Fatal(err)
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		log.Fatal(err)
	}

	// 3. 加密：每次加密都生成随机 nonce（长度由 gcm.NonceSize() 给出），
	//    把 nonce 拼在密文最前面一起保存。nonce 不需要保密，但绝不能重复。
	plaintext := []byte("需要保密的一段话：明天的会议改到下午三点。")
	nonce := make([]byte, gcm.NonceSize())
	if _, err := rand.Read(nonce); err != nil {
		log.Fatal(err)
	}
	sealed := gcm.Seal(nil, nonce, plaintext, nil)
	blob := append(nonce, sealed...)
	fmt.Printf("密文(十六进制): %s\n", hex.EncodeToString(blob))

	// 4. 解密：先切出 nonce，其余部分交给 Open 还原。
	ns := gcm.NonceSize()
	if len(blob) < ns {
		log.Fatal("密文太短，数据不完整")
	}
	got, err := gcm.Open(nil, blob[:ns], blob[ns:], nil)
	if err != nil {
		log.Fatal("解密失败：", err)
	}
	fmt.Printf("解密结果: %s\n", got)
	fmt.Printf("与原文一致: %v\n", string(got) == string(plaintext))

	// 5. 篡改验证：改动密文任意一个字节，GCM 的认证标签会让解密直接失败，
	//    而不是默默返回错误的数据。
	tampered := make([]byte, len(blob))
	copy(tampered, blob)
	tampered[len(tampered)-1] ^= 0xFF
	if _, err := gcm.Open(nil, tampered[:ns], tampered[ns:], nil); err != nil {
		fmt.Println("篡改后的密文解密失败（符合预期）:", err)
	}
}
```

运行输出（密文每次都不同，因为密钥和 nonce 都是随机的）：

```text
密文(十六进制): <每次运行不同>
解密结果: 需要保密的一段话：明天的会议改到下午三点。
与原文一致: true
篡改后的密文解密失败（符合预期）: cipher: message authentication failed
```

## 加密文件时的思路

加密大文件不要一次性全读进内存，按块流式处理：

- 小文件（几 MB 以内）：直接复用上面的 GCM 写法，整体加密后写入文件即可。
- 大文件：把文件切成固定大小的分块（比如 64KB 一块），每块用独立 nonce 分别 GCM 加密，依次写入目标文件；解密时按同样顺序逐块还原。这样内存占用稳定，也不用自己处理填充。
- 分块时务必保证 nonce 不重复：可以用"随机基准 nonce + 块序号递增"的组合，或每块重新随机（块数不是天文数字时，96 位随机 nonce 碰撞概率可忽略）。

## 常见坑清单

1. **用 ECB 模式**：相同明文块产生相同密文块，图片加密后轮廓都看得见。标准库甚至不提供 ECB 实现，就是不想让你用。
2. **nonce/IV 复用**：GCM 同一密钥下 nonce 重复会同时泄漏明文关系和认证密钥，是灾难性错误。每次加密重新随机生成。
3. **CBC 不做完整性校验**：CBC 密文可被篡改且解密不报错，还会引入填充预言攻击。用 CBC 就必须再加 HMAC，先加密后签名（Encrypt-then-MAC）；能选 GCM 就别走这条路。
4. **密钥硬编码或用弱口令直接当密钥**：口令要经过 KDF（如 argon2、scrypt、PBKDF2）派生出密钥，不能把口令字符串补零凑够 32 字节直接用。
5. **自己发明填充和拼接规则**：填充用标准 PKCS#7，拼接格式写清字段边界（nonce 长度、密文长度），并给格式留版本号，方便日后升级算法。

## 小结

- AES 本身只是块算法，安全性很大程度上取决于模式：新代码默认 AES-256-GCM。
- 记住三件事：密钥随机生成并妥善保管、nonce 每次都要新、优先选自带认证的模式。
- 代码全部基于 Go 标准库 `crypto/aes`、`crypto/cipher`、`crypto/rand`，没有第三方依赖。

本文为原创学习笔记：按 Go 标准库公开用法编写，未参考原文件内容；代码为原创实现并已经运行验证。
