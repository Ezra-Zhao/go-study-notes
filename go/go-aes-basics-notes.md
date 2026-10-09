# AES 对称加密第一课：分组、密钥与 GCM 模式

> 这一讲很短，只讲 AES 的三个基本概念，外加一段能跑的加解密演示。大纲细节有限，不展开编造。

## 一、三个基本概念

1. **对称加密**：加密和解密用同一把钥匙。钥匙就是一串随机字节，AES 支持 16 / 24 / 32 字节三种长度，分别对应 AES-128 / AES-192 / AES-256，越长越安全；
2. **分组加密**：AES 每次处理固定 16 字节的一"块"，长数据按块切分处理。Go 的 `crypto/aes` 包负责这层；
3. **分组模式**：光有分组不够，还要决定"块与块之间怎么衔接"。新手直接用 **GCM 模式**（`cipher.NewGCM`）：它自带完整性校验，密文被篡改时解密会直接报错，省心。

## 二、动手演示：AES-256-GCM 一来一回

```go
package main

import (
	"bytes"
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"fmt"
)

func main() {
	key := make([]byte, 32) // AES-256
	if _, err := rand.Read(key); err != nil {
		panic(err)
	}
	block, err := aes.NewCipher(key)
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

	plain := []byte("这是我的秘密备忘")
	ciphertext := gcm.Seal(nil, nonce, plain, nil)
	fmt.Printf("明文 %d 字节 -> 密文 %d 字节\n", len(plain), len(ciphertext))

	back, err := gcm.Open(nil, nonce, ciphertext, nil)
	if err != nil {
		panic(err)
	}
	fmt.Printf("解密还原一致：%v\n", bytes.Equal(back, plain))

	// 篡改一个字节，解密必须失败：GCM 的完整性校验在起作用
	ciphertext[10] ^= 0x01
	if _, err := gcm.Open(nil, nonce, ciphertext, nil); err != nil {
		fmt.Printf("篡改后解密失败（符合预期）：%v\n", err)
	}
}
```

运行结果（示例）：

```
明文 24 字节 -> 密文 40 字节
解密还原一致：true
篡改后解密失败（符合预期）：cipher: message authentication failed
```

## 三、本讲小结

- AES 是对称加密，钥匙 16/24/32 字节对应三种强度；
- 新手用 GCM 模式，自带防篡改校验；
- `nonce`（随机数）每次加密都要换，不能重复使用同一把钥匙配同一个 nonce。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 标准库文档。
