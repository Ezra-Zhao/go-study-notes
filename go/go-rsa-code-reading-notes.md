# Go 代码精读：用 RSA 给 AES 密钥套个信封

这篇笔记拆一份很短的配套代码文件（约 3 KB）。它做的事一句话说得清：把一把 AES 密钥用 RSA 公钥加密，再用私钥解回来。文件不大，但正好拿来练读代码的基本功——先看签名，再看流程，最后看错误处理。

## 一、先看骨架：文件被注释切成三段

- 生成密钥一段：调用 GenerateRSAKey(2048) 生成 2048 位密钥对，落盘成 public.txt 和 private.txt（承接上一篇的密钥生成）；
- RSA 加密一段：本篇主角，把 AES 密钥加密；
- RSA 解密一段：用私钥把 AES 密钥还原回来，是加密的镜像流程；
- main() 把三段串起来，跑一遍完整演示。

读别人的代码，先摸骨架再钻细节，顺序反了很容易在枝节里迷路。

## 二、签名会说话：RsaEncrypt(AesKey string, path string) []byte

加密入口的签名是 `func RsaEncrypt(AesKey string, path string) []byte`，短短一行藏了三个信息：

1. 第一个参数叫 AesKey——被加密的根本不是文件，而是 AES 密钥。混合加密的分工在这里坐实：大数据交给 AES，RSA 只负责护送密钥；
2. 第二个参数 path 是公钥文件的路径，函数自己负责开文件、读内容，调用方不用先把公钥读出来再传进去；
3. 只返回一个 []byte，没有 error。出错怎么办？函数内部用 log.Fatal 就地终止进程。教学演示里图省事很常见，但生产代码别学这一招：库函数不该替调用方决定"活不下去"，老老实实把 error 返回出去才是正道。下面的实现保留了这个签名，是为了对着原文件的读法讲；改进的方向放在文末。

## 三、读公钥文件：Stat 定尺码的老式三步

原文件读公钥不用现成的一行函数，而是手动走三步：os.Open 打开文件，info, _ := fp.Stat() 问出文件大小，make([]byte, info.Size()) 按尺寸备好缓冲，最后 fp.Read(buf) 把内容读进来。

为什么要先问大小？老式写法讲究"先量尺寸再裁布"：不知道文件多大，缓冲就备不准。

这个写法的坑也得认清：Read 并不保证一次读满，返回的字节数可以小于缓冲长度（短读是合法行为）；文件若在 Stat 之后被人改大，还会读不全。真要稳妥，要么用 io.ReadFull 循环读满，要么干脆换成 os.ReadFile——今天写新代码基本都是一行了事。但存量项目里这种三步写法到处都是，读老代码时必须认得。

顺带一提，这一步出错同样走 log.Fatal，和"只返回 []byte"的签名正好配套：一错到底，就地终止。

## 四、从文本到密文：解码三连

公钥文本读进来之后，还要连过三关才能用：

1. pem.Decode(buf)：PEM 是带头尾标记的文本信封，先拆信封拿到里面的 DER 字节。文件不是 PEM 格式时它返回 nil，所以拆完必须先判空，别拿着 nil 往下走；
2. x509.ParsePKIXPublicKey(block.Bytes)：把 DER 字节解析成公钥对象。返回值是通用接口类型，还要再断言成 *rsa.PublicKey，才能确认手里拿的确实是 RSA 公钥；
3. rsa.EncryptPKCS1v15(rand.Reader, pubKey, 明文)：真正加密的一步。随机源参与填充运算，所以同一份明文每次加密出的密文都不一样，这是正常现象。

长度上有一条硬红线（已实测）：PKCS#1 v1.5 填充至少占 11 个字节，2048 位密钥一次最多加密 245 字节，超了直接报 message too long for RSA key size。AES 密钥最长才 32 字节，绰绰有余；想拿 RSA 加密文件本体，门都没有——那是 AES 的活。另外，密文长度恒等于密钥长度：2048 位密钥加密出的密文永远是 256 字节，和明文多长无关。

## 五、完整实现（按上面读到的流程手写，已实跑验证）

下面这份实现按原文件的结构与风格手写：加密函数保留 RsaEncrypt(AesKey string, path string) []byte 的签名与三步读法。解密函数的完整签名在原文件里没有观测到，这里按与加密相同的风格补齐，目的是把全流程跑通。

```go
package main

import (
	"crypto/rand"
	"crypto/rsa"
	"crypto/x509"
	"encoding/pem"
	"fmt"
	"log"
	"os"
	"path/filepath"
)

// GenerateRSAKey 生成 bits 位 RSA 密钥对，落盘为 public.txt / private.txt
func GenerateRSAKey(bits int, dir string) {
	key, err := rsa.GenerateKey(rand.Reader, bits)
	if err != nil {
		log.Fatal(err)
	}
	privPEM := pem.EncodeToMemory(&pem.Block{
		Type:  "RSA PRIVATE KEY",
		Bytes: x509.MarshalPKCS1PrivateKey(key),
	})
	if err := os.WriteFile(filepath.Join(dir, "private.txt"), privPEM, 0600); err != nil {
		log.Fatal(err)
	}
	pubDER, err := x509.MarshalPKIXPublicKey(&key.PublicKey)
	if err != nil {
		log.Fatal(err)
	}
	pubPEM := pem.EncodeToMemory(&pem.Block{Type: "PUBLIC KEY", Bytes: pubDER})
	if err := os.WriteFile(filepath.Join(dir, "public.txt"), pubPEM, 0644); err != nil {
		log.Fatal(err)
	}
}

// RsaEncrypt 读取 PEM 公钥文件，加密 AES 密钥字符串，返回密文
func RsaEncrypt(aesKey string, path string) []byte {
	fp, err := os.Open(path)
	if err != nil {
		log.Fatal(err)
	}
	defer fp.Close()
	info, err := fp.Stat()
	if err != nil {
		log.Fatal(err)
	}
	buf := make([]byte, info.Size())
	if _, err := fp.Read(buf); err != nil {
		log.Fatal(err)
	}
	block, _ := pem.Decode(buf)
	if block == nil {
		log.Fatal("公钥文件不是 PEM 格式")
	}
	pubIfc, err := x509.ParsePKIXPublicKey(block.Bytes)
	if err != nil {
		log.Fatal(err)
	}
	pubKey, ok := pubIfc.(*rsa.PublicKey)
	if !ok {
		log.Fatal("公钥文件里装的不是 RSA 公钥")
	}
	ciphertext, err := rsa.EncryptPKCS1v15(rand.Reader, pubKey, []byte(aesKey))
	if err != nil {
		log.Fatal(err)
	}
	return ciphertext
}

// RsaDecrypt 读取 PEM 私钥文件，把密文还原成 AES 密钥字符串
func RsaDecrypt(ciphertext []byte, path string) string {
	buf, err := os.ReadFile(path)
	if err != nil {
		log.Fatal(err)
	}
	block, _ := pem.Decode(buf)
	if block == nil {
		log.Fatal("私钥文件不是 PEM 格式")
	}
	privKey, err := x509.ParsePKCS1PrivateKey(block.Bytes)
	if err != nil {
		log.Fatal(err)
	}
	plain, err := rsa.DecryptPKCS1v15(rand.Reader, privKey, ciphertext)
	if err != nil {
		log.Fatal(err)
	}
	return string(plain)
}

func main() {
	dir, err := os.MkdirTemp("", "rsa-code-reading")
	if err != nil {
		log.Fatal(err)
	}
	defer os.RemoveAll(dir)

	GenerateRSAKey(2048, dir)

	aesKey := "0123456789abcdef0123456789abcdef" // 演示用 AES-256 密钥字符串（32 字节）
	ciphertext := RsaEncrypt(aesKey, filepath.Join(dir, "public.txt"))
	fmt.Printf("AES 密钥 %d 字节 -> RSA 密文 %d 字节\n", len(aesKey), len(ciphertext))

	plain := RsaDecrypt(ciphertext, filepath.Join(dir, "private.txt"))
	fmt.Printf("解密还原一致：%v\n", plain == aesKey)
}
```

运行结果（Go 1.24 实跑）：

```
AES 密钥 32 字节 -> RSA 密文 256 字节
解密还原一致：true
```

## 六、两个使用边界

1. 私钥文件就是命门：private.txt 的落盘权限要收紧（示例里给的是 0600），更不能提交进任何代码仓库。演示代码把密钥写进临时目录、跑完即删，学的就是这个习惯；
2. 技术本身是中性的：用公钥护送对称密钥，是备份加密、密钥分发里的常规工程写法。本文只讲技术本身与防御视角，不涉及任何攻击用途。

## 七、小结

- 读代码先看骨架：生成、加密、解密三段，加一个 main 串联全程；
- 签名定分工：RSA 只加密 AES 密钥，2048 位密钥 245 字节的单次上限，决定了它当不了文件搬运工；
- 老式三步读法要认得：Open、Stat、make、Read；新代码用 os.ReadFile；
- 解码三连：pem.Decode、x509.ParsePKIXPublicKey、rsa.EncryptPKCS1v15。

改进方向：把 log.Fatal 换成返回 error、把手动三步读法换成 os.ReadFile，这份教学代码就接近生产写法了。

本文为学习笔记（编纂），用自己的话重写；
