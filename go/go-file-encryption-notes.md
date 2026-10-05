# Go 文件加密与解密实战：RSA + AES 混合加密

> 学这一章，是为了保护数据：备份加密、敏感文档本地存放、传输过程中的保护。加密是盾，不是矛。

## 一、为什么需要文件加密

先想三个正面场景：

1. **云备份加密**：重要资料上传网盘前先加密，就算网盘泄露，别人拿到也只是一堆乱码；
2. **敏感文档本地存放**：笔记本电脑里放着合同、证件扫描件，设备丢了也不怕；
3. **传输保护**：给客户发资料，先加密再发，中间人截获了也打不开。

结论：加密解决的是"东西丢了也不怕"的问题，是每个人的数据保护基本功。

## 二、原理：RSA + AES 混合加密

### 1. 对称加密（AES）：快，但钥匙不好送

对称加密用同一把钥匙加密和解密。AES 是目前的主流，速度快，几 GB 的文件也能流畅处理。但有个经典难题：**这把钥匙怎么安全地交到对方手里？** 钥匙本身在传输中一旦被截获，加密就白做了。

### 2. 非对称加密（RSA）：钥匙好管，但慢

RSA 用一对钥匙：公钥加密、私钥解密。公钥可以公开给任何人，私钥自己藏好。对方用你的公钥加密，只有你能用私钥解开——钥匙分发问题解决了。但 RSA 运算慢，不适合直接加密大文件。

### 3. 混合加密：业界的标准答案

把两者拼起来，各取所长（TLS、PGP 等协议都是这个思路）：

1. 随机生成一把一次性的 AES 会话密钥；
2. 用 AES 会话密钥加密文件内容（快）；
3. 用接收方的 RSA 公钥加密这把会话密钥（解决钥匙分发）；
4. 把"加密后的会话密钥 + 加密后的文件"一起发出去；
5. 接收方先用 RSA 私钥解出会话密钥，再用它解密文件。

一句话：**文件体走 AES（求快），钥匙走 RSA（求安全）**。

## 三、Go 实现

Go 标准库把整套工具都备齐了：`crypto/rsa`、`crypto/aes`、`crypto/x509`、`encoding/pem`。下面代码按功能拆成几段，合在一起即为完整程序。

### 1. RSA 密钥生成与 PEM 存取

```go
package main

import (
    "crypto/rand"
    "crypto/rsa"
    "crypto/x509"
    "encoding/pem"
    "fmt"
    "os"
)

// 生成 RSA 密钥对，bits 一般取 2048
func generateRSAKey(bits int) (*rsa.PrivateKey, *rsa.PublicKey, error) {
    priv, err := rsa.GenerateKey(rand.Reader, bits)
    if err != nil {
        return nil, nil, err
    }
    return priv, &priv.PublicKey, nil
}

// 私钥存成 PEM 文件，权限 0600（仅自己可读写）
func savePrivateKey(path string, key *rsa.PrivateKey) error {
    der := x509.MarshalPKCS1PrivateKey(key)
    block := &pem.Block{Type: "RSA PRIVATE KEY", Bytes: der}
    return os.WriteFile(path, pem.EncodeToMemory(block), 0600)
}

// 公钥存成 PEM 文件，可以公开
func savePublicKey(path string, key *rsa.PublicKey) error {
    der, err := x509.MarshalPKIXPublicKey(key)
    if err != nil {
        return err
    }
    block := &pem.Block{Type: "PUBLIC KEY", Bytes: der}
    return os.WriteFile(path, pem.EncodeToMemory(block), 0644)
}

func loadPrivateKey(path string) (*rsa.PrivateKey, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    block, _ := pem.Decode(data)
    if block == nil {
        return nil, fmt.Errorf("no PEM block found in %s", path)
    }
    return x509.ParsePKCS1PrivateKey(block.Bytes)
}

func loadPublicKey(path string) (*rsa.PublicKey, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    block, _ := pem.Decode(data)
    if block == nil {
        return nil, fmt.Errorf("no PEM block found in %s", path)
    }
    pub, err := x509.ParsePKIXPublicKey(block.Bytes)
    if err != nil {
        return nil, err
    }
    rsaPub, ok := pub.(*rsa.PublicKey)
    if !ok {
        return nil, fmt.Errorf("not an RSA public key")
    }
    return rsaPub, nil
}
```

要点：私钥文件权限设 `0600`；公钥用 PKIX 格式、私钥用 PKCS1 格式，这是 Go 里的常规配对。

### 2. RSA 加解密（只用来加密小数据，比如会话密钥）

```go
package main

import (
    "crypto/rand"
    "crypto/rsa"
)

func rsaEncrypt(pub *rsa.PublicKey, plain []byte) ([]byte, error) {
    return rsa.EncryptPKCS1v15(rand.Reader, pub, plain)
}

func rsaDecrypt(priv *rsa.PrivateKey, cipherText []byte) ([]byte, error) {
    return rsa.DecryptPKCS1v15(rand.Reader, priv, cipherText)
}
```

注意：RSA 一次能加密的数据长度受密钥长度限制（2048 位密钥约 245 字节），所以它只负责加密 32 字节的会话密钥，文件内容交给 AES。

### 3. AES-CBC 加解密与 PKCS7 填充

AES 是按 16 字节一块加密的，文件长度不一定是 16 的倍数，所以需要填充。PKCS7 的规则很直观：缺几个字节就补几个"几"。

```go
package main

import (
    "bytes"
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "fmt"
)

func pkcs7Pad(data []byte, blockSize int) []byte {
    pad := blockSize - len(data)%blockSize
    return append(data, bytes.Repeat([]byte{byte(pad)}, pad)...)
}

func pkcs7Unpad(data []byte) ([]byte, error) {
    if len(data) == 0 {
        return nil, fmt.Errorf("empty data")
    }
    pad := int(data[len(data)-1])
    if pad == 0 || pad > len(data) {
        return nil, fmt.Errorf("invalid padding")
    }
    for _, b := range data[len(data)-pad:] {
        if int(b) != pad {
            return nil, fmt.Errorf("invalid padding")
        }
    }
    return data[:len(data)-pad], nil
}

// CBC 模式加密：随机 IV 写在密文最前面，解密时先读出来
func aesEncryptCBC(key, plain []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }
    plain = pkcs7Pad(plain, block.BlockSize())
    out := make([]byte, block.BlockSize()+len(plain))
    iv := out[:block.BlockSize()]
    if _, err := rand.Read(iv); err != nil {
        return nil, err
    }
    cipher.NewCBCEncrypter(block, iv).CryptBlocks(out[block.BlockSize():], plain)
    return out, nil
}

func aesDecryptCBC(key, cipherText []byte) ([]byte, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, err
    }
    bs := block.BlockSize()
    if len(cipherText) < bs*2 || len(cipherText)%bs != 0 {
        return nil, fmt.Errorf("invalid ciphertext length")
    }
    iv := cipherText[:bs]
    data := cipherText[bs:]
    plain := make([]byte, len(data))
    cipher.NewCBCDecrypter(block, iv).CryptBlocks(plain, data)
    return pkcs7Unpad(plain)
}
```

要点：CBC 模式下每个块的加密都混入前一块的结果，所以 IV（初始向量）每次必须随机，且不需要保密，直接拼在密文前面即可；解密时严格校验填充，防篡改。

### 4. 混合加密：文件分块加解密

大文件不能一次读进内存，用流式分块处理。文件格式约定为：`[4 字节密钥长度][RSA 加密的会话密钥][16 字节 IV][AES 加密的文件内容]`。

```go
package main

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "crypto/rsa"
    "encoding/binary"
    "fmt"
    "io"
    "os"
)

func hybridEncryptFile(pub *rsa.PublicKey, srcPath, dstPath string) error {
    // 1. 随机生成 32 字节 AES 会话密钥（AES-256）
    sessionKey := make([]byte, 32)
    if _, err := rand.Read(sessionKey); err != nil {
        return err
    }
    // 2. 用 RSA 公钥加密会话密钥
    encKey, err := rsaEncrypt(pub, sessionKey)
    if err != nil {
        return err
    }

    in, err := os.Open(srcPath)
    if err != nil {
        return err
    }
    defer in.Close()
    out, err := os.Create(dstPath)
    if err != nil {
        return err
    }
    defer out.Close()

    // 3. 写文件头：密钥长度 + 加密后的会话密钥
    if err := binary.Write(out, binary.BigEndian, uint32(len(encKey))); err != nil {
        return err
    }
    if _, err := out.Write(encKey); err != nil {
        return err
    }

    // 4. 写随机 IV
    block, err := aes.NewCipher(sessionKey)
    if err != nil {
        return err
    }
    iv := make([]byte, block.BlockSize())
    if _, err := rand.Read(iv); err != nil {
        return err
    }
    if _, err := out.Write(iv); err != nil {
        return err
    }

    // 5. 分块读取、加密、写入；尾部留到最后做 PKCS7 填充
    enc := cipher.NewCBCEncrypter(block, iv)
    buf := make([]byte, 4096)
    var carry []byte
    for {
        n, rerr := in.Read(buf)
        if n > 0 {
            chunk := append(carry, buf[:n]...)
            whole := len(chunk) / block.BlockSize() * block.BlockSize()
            if whole > 0 {
                enc.CryptBlocks(chunk[:whole], chunk[:whole])
                if _, err := out.Write(chunk[:whole]); err != nil {
                    return err
                }
            }
            carry = chunk[whole:]
        }
        if rerr != nil {
            break
        }
    }
    tail := pkcs7Pad(carry, block.BlockSize())
    enc.CryptBlocks(tail, tail)
    _, err = out.Write(tail)
    return err
}

func hybridDecryptFile(priv *rsa.PrivateKey, srcPath, dstPath string) error {
    in, err := os.Open(srcPath)
    if err != nil {
        return err
    }
    defer in.Close()

    // 1. 读文件头，解出会话密钥
    var keyLen uint32
    if err := binary.Read(in, binary.BigEndian, &keyLen); err != nil {
        return err
    }
    encKey := make([]byte, keyLen)
    if _, err := io.ReadFull(in, encKey); err != nil {
        return err
    }
    sessionKey, err := rsaDecrypt(priv, encKey)
    if err != nil {
        return err
    }

    // 2. 读 IV，解密剩余内容，去填充后写文件
    block, err := aes.NewCipher(sessionKey)
    if err != nil {
        return err
    }
    iv := make([]byte, block.BlockSize())
    if _, err := io.ReadFull(in, iv); err != nil {
        return err
    }
    rest, err := io.ReadAll(in)
    if err != nil {
        return err
    }
    if len(rest) == 0 || len(rest)%block.BlockSize() != 0 {
        return fmt.Errorf("invalid ciphertext")
    }
    plain := make([]byte, len(rest))
    cipher.NewCBCDecrypter(block, iv).CryptBlocks(plain, rest)
    plain, err = pkcs7Unpad(plain)
    if err != nil {
        return err
    }
    return os.WriteFile(dstPath, plain, 0644)
}
```

分块逻辑的关键：每次只加密"凑满整块"的部分，不足一块的字节留到下一轮（`carry`），最后统一做 PKCS7 填充。这样无论文件多大，内存占用都恒定在几 KB。

小贴士：密文是二进制，如果要在 JSON 或文本协议里传输，先用 `base64.StdEncoding.EncodeToString()` 转成可打印字符，接收方再解回来。

### 5. 使用示例

```go
func main() {
    // 第一次：生成密钥对并保存
    priv, pub, err := generateRSAKey(2048)
    if err != nil {
        panic(err)
    }
    if err := savePrivateKey("private.pem", priv); err != nil {
        panic(err)
    }
    if err := savePublicKey("public.pem", pub); err != nil {
        panic(err)
    }

    // 加密：只需要对方的公钥
    pub2, _ := loadPublicKey("public.pem")
    if err := hybridEncryptFile(pub2, "合同.pdf", "合同.pdf.enc"); err != nil {
        panic(err)
    }

    // 解密：只需要自己的私钥
    priv2, _ := loadPrivateKey("private.pem")
    if err := hybridDecryptFile(priv2, "合同.pdf.enc", "合同-解密.pdf"); err != nil {
        panic(err)
    }
}
```

## 四、安全启示：从 WannaCry 事件学到的防御课

2017 年的 WannaCry 勒索事件给全行业上了一课。抛开攻击细节，只看防御方应该做什么：

1. **及时打补丁**：事件利用的是早已发布补丁的已知漏洞。系统和软件的更新提醒出来就装，别拖；
2. **重要数据离线备份**：备份遵循"三份拷贝、两种介质、一份离线"，备份文件本身再用上面的方法加密；
3. **不乱开附件和不明链接**：来路不明的文档、压缩包，先确认再打开；
4. **收紧内网**：不必要的端口和共享服务关掉，机器密码用高强度且不复用；
5. **加密是日常习惯**：备份先加密再上云，敏感文件存本地也加密——等出事再加密就晚了。

技术是中立的：同一套 RSA+AES，既可以是保护数据的盾，也可以被坏人滥用。我们学它、写它，是为了当好数据的守门人。

本文为学习笔记（编纂），知识点源自公开的 Go 标准库文档与网络安全常识；
