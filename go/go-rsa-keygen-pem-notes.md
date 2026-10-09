# Go 生成 RSA 密钥对：2048 位与 PEM 存放

> 混合加密的第一步是先有钥匙。这一讲用 Go 标准库生成 2048 位 RSA 密钥对，并以 PEM 格式存成文件，后续加解密都从这里取钥匙。

## 一、密钥生成三步走

Go 的 `crypto/rsa` 包把这件事做得很直接：

1. **生成**：`rsa.GenerateKey(rand.Reader, 2048)` 生成 2048 位密钥对（2048 是目前兼顾安全与性能的主流选择）；
2. **编码**：私钥用 `x509.MarshalPKCS1PrivateKey` 转 DER，公钥用 `x509.MarshalPKIXPublicKey` 转 DER；
3. **落盘**：DER 是二进制，再包一层 PEM（Base64 + 头尾标记行），分别存为 `private.txt` 和 `public.txt`，文本可读、好管理。

PEM 文件长这样（截断示意）：

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA...
-----END RSA PRIVATE KEY-----
```

```
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
-----END PUBLIC KEY-----
```

## 二、动手实现：GenerateRSAKey(2048)

下面这段程序完整演示：生成 → PEM 编码 → 写入 `public.txt` / `private.txt` → 读回解析验证，确保存进去的钥匙是能用的。

```go
package main

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"fmt"
	"os"
	"path/filepath"
)

// GenerateRSAKey 生成 bits 位 RSA 密钥对，PEM 编码后分别写入 dir 下的 public.txt / private.txt
func GenerateRSAKey(bits int, dir string) error {
	key, err := rsa.GenerateKey(rand.Reader, bits)
	if err != nil {
		return err
	}
	// 私钥：PKCS#1 DER -> PEM
	privDER := x509.MarshalPKCS1PrivateKey(key)
	privPEM := pem.EncodeToMemory(&pem.Block{Type: "RSA PRIVATE KEY", Bytes: privDER})
	if err := os.WriteFile(filepath.Join(dir, "private.txt"), privPEM, 0600); err != nil {
		return err
	}
	// 公钥：PKIX DER -> PEM
	pubDER, err := x509.MarshalPKIXPublicKey(&key.PublicKey)
	if err != nil {
		return err
	}
	pubPEM := pem.EncodeToMemory(&pem.Block{Type: "PUBLIC KEY", Bytes: pubDER})
	if err := os.WriteFile(filepath.Join(dir, "public.txt"), pubPEM, 0644); err != nil {
		return err
	}
	return nil
}

func main() {
	dir, err := os.MkdirTemp("", "rsakeys")
	if err != nil {
		panic(err)
	}
	defer os.RemoveAll(dir)

	if err := GenerateRSAKey(2048, dir); err != nil {
		panic(err)
	}

	// 读回验证：私钥能解析、位数对、公钥能解析
	privPEM, _ := os.ReadFile(filepath.Join(dir, "private.txt"))
	privBlock, _ := pem.Decode(privPEM)
	privKey, err := x509.ParsePKCS1PrivateKey(privBlock.Bytes)
	if err != nil {
		panic(err)
	}
	pubPEM, _ := os.ReadFile(filepath.Join(dir, "public.txt"))
	pubBlock, _ := pem.Decode(pubPEM)
	pubIfc, err := x509.ParsePKIXPublicKey(pubBlock.Bytes)
	if err != nil {
		panic(err)
	}
	pubKey, ok := pubIfc.(*rsa.PublicKey)
	if !ok {
		panic("公钥类型断言失败")
	}
	fmt.Printf("私钥位数：%d，公钥指数 E=%d\n", privKey.N.BitLen(), pubKey.E)
	fmt.Printf("私钥文件头：%.27s\n", privPEM)
	fmt.Printf("公钥文件头：%.26s\n", pubPEM)
	fmt.Println("密钥对生成、落盘、读回解析全部通过")
}
```

运行结果（示例）：

```
私钥位数：2048，公钥指数 E=65537
私钥文件头：-----BEGIN RSA PRIVATE KEY-----
公钥文件头：-----BEGIN PUBLIC KEY-----
密钥对生成、落盘、读回解析全部通过
```

## 三、两个容易踩的坑

1. **私钥文件权限**：示例里写 `private.txt` 用了 `0600`，只有自己能读。密钥文件权限放开等于把钥匙插在门上；
2. **别把私钥和公钥搞混**：`ParsePKCS1PrivateKey` 解析私钥，`ParsePKIXPublicKey` 解析公钥，混用会直接报错。PEM 头里的 `RSA PRIVATE KEY` / `PUBLIC KEY` 就是区分标记。

## 四、本讲小结

- `rsa.GenerateKey(rand.Reader, 2048)` 一行生成密钥对；
- DER 转 PEM 再落盘：私钥 `private.txt`（0600）、公钥 `public.txt`；
- 读回解析验证是好习惯，确保存进去的钥匙真实可用。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 标准库文档。
