# 为什么必须是"混合加密"：单用 RSA 或单用 AES 的短板

> RSA 和 AES 各有所长，也各有短板。理解了短板，就理解了为什么现实中的文件加密都是"RSA 保护密钥、AES 干活"的混合体系。

## 一、只用 AES：快，但钥匙送不出去

AES 是对称加密：加密和解密用同一把钥匙。它的优点是**快**，几个 GB 的文件也能流畅处理；缺点是**钥匙分发难**——你想把加密文件发给朋友，钥匙本身怎么安全地交到他手里？如果钥匙在传输中被截获，加密就白做了。

## 二、只用 RSA：钥匙好送，但干不了重活

RSA 是非对称加密：公钥加密、私钥解密，钥匙分发问题天然解决（公钥本来就是公开的）。但它有两个硬伤：

1. **慢**：RSA 运算比 AES 慢几个数量级；
2. **一次能加密的数据极小**：2048 位 RSA 用 PKCS#1 v1.5 填充，一次最多加密 214 字节（256 - 11）。想加密一个文件？得分成几百字节的小块逐块加密，又慢又麻烦。

## 三、混合体系：各干各擅长的事

标准做法是分工：

- **AES**：负责加密真正的文件数据（快、能处理大数据）；
- **RSA**：只加密那把小小的 AES 密钥（数据小，正好在 RSA 的能力范围内，钥匙分发问题也解决了）。

接收方反向操作：先用 RSA 私钥解出 AES 密钥，再用 AES 密钥解密文件。两边短板互相补上。

## 四、动手验证：RSA 的"块头限制"真实存在

下面这段程序用实测证明上面的结论：RSA-2048 加密 300 字节的数据直接报错"message too long"；而 AES-GCM 加密 1MB 数据毫秒级完成。

```go
package main

import (
	"bytes"
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"crypto/rsa"
	"fmt"
	"time"
)

func main() {
	// 1. RSA-2048 尝试加密 300 字节：超出单次上限（约 214 字节），必然失败
	rsaKey, err := rsa.GenerateKey(rand.Reader, 2048)
	if err != nil {
		panic(err)
	}
	bigData := bytes.Repeat([]byte("A"), 300)
	_, err = rsa.EncryptPKCS1v15(rand.Reader, &rsaKey.PublicKey, bigData)
	fmt.Printf("RSA-2048 加密 300 字节：err = %v（预期失败，单次上限约 214 字节）\n", err)

	// 2. AES-GCM 加密 1MB：轻松快速
	aesKey := make([]byte, 32) // AES-256
	if _, err := rand.Read(aesKey); err != nil {
		panic(err)
	}
	block, err := aes.NewCipher(aesKey)
	if err != nil {
		panic(err)
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		panic(err)
	}
	nonce := make([]byte, gcm.NonceSize())
	if _, err := rand.Read(nonce); err != nil {
		panic(err)
	}
	oneMB := bytes.Repeat([]byte("B"), 1024*1024)
	start := time.Now()
	ciphertext := gcm.Seal(nil, nonce, oneMB, nil)
	elapsed := time.Since(start)
	fmt.Printf("AES-256-GCM 加密 1MB：%v，密文 %d 字节\n", elapsed, len(ciphertext))

	// 3. 解密验证，确认数据无损
	plain, err := gcm.Open(nil, nonce, ciphertext, nil)
	if err != nil {
		panic(err)
	}
	fmt.Printf("解密还原一致：%v\n", bytes.Equal(plain, oneMB))
}
```

运行结果（示例）：

```
RSA-2048 加密 300 字节：err = crypto/rsa: message too long for RSA key size（预期失败，单次上限约 214 字节）
AES-256-GCM 加密 1MB：1.2ms，密文 1048592 字节
解密还原一致：true
```

## 五、本讲小结

- AES 快但钥匙难送，RSA 钥匙好送但慢且一次只能加密约 214 字节；
- 混合加密是标准答案：RSA 只保护小小的 AES 密钥，AES 负责加密文件本体；
- 上面的实测证明了 RSA 的块头限制是真实存在的物理约束，不是理论说法。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 标准库文档与公开的密码学常识。
