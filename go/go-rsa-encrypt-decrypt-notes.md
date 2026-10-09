# Go 实现 RSA 加解密：从 PEM 公钥到密文再回来

> 密钥对有了，下一讲是"用起来"：读 PEM 公钥文件加密一小段数据（比如 AES 密钥），再用私钥解回来，全流程跑通。

## 一、加密流程四步

对照 `crypto/rsa` + `crypto/x509` 的标准做法：

1. **读文件**：`os.ReadFile` 读出 `public.txt` 的 PEM 文本；
2. **PEM 解码**：`pem.Decode` 剥掉头尾标记，拿到 DER 字节；
3. **解析公钥**：`x509.ParsePKIXPublicKey` 把 DER 还原成 `*rsa.PublicKey`；
4. **加密**：`rsa.EncryptPKCS1v15(rand.Reader, pubKey, data)` 得到密文。

解密是镜像流程：读 `private.txt` → `pem.Decode` → `x509.ParsePKCS1PrivateKey` → `rsa.DecryptPKCS1v15`。注意 PKCS#1 v1.5 填充决定了单次加密上限（2048 位密钥约 214 字节），所以这里只加密小数据——实战中加密的是 AES 密钥，正好落在这个范围内。

## 二、动手实现：RsaEncrypt / RsaDecrypt 全流程

下面这段程序先在内存里生成密钥对并写成 PEM 文件（复用上一讲的思路），然后严格按"读文件 → PEM 解码 → x509 解析 → 加解密"的流程走一遍，最后验证解密结果与原文一致。

```go
package main

import (
	"bytes"
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"fmt"
	"os"
	"path/filepath"
)

// RsaEncrypt 读取 PEM 公钥文件，加密 data（长度须在 RSA 单次上限内）
func RsaEncrypt(data []byte, pubKeyPath string) ([]byte, error) {
	buf, err := os.ReadFile(pubKeyPath)
	if err != nil {
		return nil, err
	}
	// PEM 解码
	block, _ := pem.Decode(buf)
	if block == nil {
		return nil, fmt.Errorf("PEM 解码失败")
	}
	pubIfc, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	pubKey, ok := pubIfc.(*rsa.PublicKey)
	if !ok {
		return nil, fmt.Errorf("不是 RSA 公钥")
	}
	return rsa.EncryptPKCS1v15(rand.Reader, pubKey, data)
}

// RsaDecrypt 读取 PEM 私钥文件，还原密文
func RsaDecrypt(ciphertext []byte, privKeyPath string) ([]byte, error) {
	buf, err := os.ReadFile(privKeyPath)
	if err != nil {
		return nil, err
	}
	block, _ := pem.Decode(buf)
	if block == nil {
		return nil, fmt.Errorf("PEM 解码失败")
	}
	privKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	return rsa.DecryptPKCS1v15(rand.Reader, privKey, ciphertext)
}

func main() {
	dir, err := os.MkdirTemp("", "rsademo")
	if err != nil {
		panic(err)
	}
	defer os.RemoveAll(dir)

	// 准备密钥文件（演示用，实际使用时由上一讲的 GenerateRSAKey 生成）
	key, _ := rsa.GenerateKey(rand.Reader, 2048)
	privDER := x509.MarshalPKCS1PrivateKey(key)
	os.WriteFile(filepath.Join(dir, "private.txt"),
		pem.EncodeToMemory(&pem.Block{Type: "RSA PRIVATE KEY", Bytes: privDER}), 0600)
	pubDER, _ := x509.MarshalPKIXPublicKey(&key.PublicKey)
	os.WriteFile(filepath.Join(dir, "public.txt"),
		pem.EncodeToMemory(&pem.Block{Type: "PUBLIC KEY", Bytes: pubDER}), 0644)

	// 模拟"被 RSA 保护的小数据"：一把 32 字节的 AES 密钥
	aesKey := make([]byte, 32)
	if _, err := rand.Read(aesKey); err != nil {
		panic(err)
	}
	ciphertext, err := RsaEncrypt(aesKey, filepath.Join(dir, "public.txt"))
	if err != nil {
		panic(err)
	}
	fmt.Printf("原文 %d 字节 -> 密文 %d 字节\n", len(aesKey), len(ciphertext))

	plain, err := RsaDecrypt(ciphertext, filepath.Join(dir, "private.txt"))
	if err != nil {
		panic(err)
	}
	fmt.Printf("解密还原一致：%v\n", bytes.Equal(plain, aesKey))
}
```

运行结果（示例）：

```
原文 32 字节 -> 密文 256 字节
解密还原一致：true
```

## 三、两个细节

1. **密文长度固定**：2048 位密钥加密出来的密文恒为 256 字节，和原文多长没关系——这是非对称加密的典型特征；
2. **错误处理别吞掉**：`pem.Decode` 返回 nil（文件不是 PEM 格式）时要明确报错，示例里用 `fmt.Errorf` 而不是静默忽略，调试时能省很多时间。

## 四、本讲小结

- 加密：读 PEM → `pem.Decode` → `x509.ParsePKIXPublicKey` → `rsa.EncryptPKCS1v15`；
- 解密：读 PEM → `pem.Decode` → `x509.ParsePKCS1PrivateKey` → `rsa.DecryptPKCS1v15`；
- 实战定位：RSA 只加密 AES 密钥这类小数据，文件本体交给 AES。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 标准库文档。
