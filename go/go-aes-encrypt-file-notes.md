# 逐行加密一个文本文件：EncryptFile 的思路

> 这一讲很短。大纲只给出了思路（按行读取文件内容，经 AES 加密后写入）和 `EncryptFile("qtest.txt","test")` 这样的调用形式，不展开编造细节。

## 一、思路拆解

把"加密一个文本文件"拆成三步，每一步都用标准库：

1. **按行读**：`bufio.Scanner` 逐行读源文件，内存里一次只放一行，大文件也不怕；
2. **逐行加密**：每一行用 AES-GCM 加密，转 Base64 变成可打印的文本行；
3. **逐行写**：加密后的行写入目标文件，一行原文对应一行密文，行数不变，方便以后逐行解密。

参数形式参考大纲：`EncryptFile(源文件, 目标文件)`——第一个参数是要加密的文件，第二个是输出的加密文件。

## 二、动手演示

```go
package main

import (
	"bufio"
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

// EncryptFile 按行读取 src，经 AES-GCM 加密后逐行写入 dst
func EncryptFile(src, dst string, key, nonce []byte) error {
	in, err := os.Open(src)
	if err != nil {
		return err
	}
	defer in.Close()
	out, err := os.Create(dst)
	if err != nil {
		return err
	}
	defer out.Close()

	block, err := aes.NewCipher(key)
	if err != nil {
		return err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return err
	}
	w := bufio.NewWriter(out)
	defer w.Flush()

	scanner := bufio.NewScanner(in)
	for scanner.Scan() {
		line := scanner.Text()
		ct := gcm.Seal(nil, nonce, []byte(line), nil)
		fmt.Fprintln(w, base64.StdEncoding.EncodeToString(ct))
	}
	return scanner.Err()
}

func main() {
	dir, _ := os.MkdirTemp("", "aesfile")
	defer os.RemoveAll(dir)
	src := filepath.Join(dir, "qtest.txt")
	dst := filepath.Join(dir, "qtest.enc")

	// 准备演示用的源文件
	os.WriteFile(src, []byte("第一行：账号清单\n第二行：备份说明\n第三行：联系人\n"), 0644)

	key := make([]byte, 32)
	rand.Read(key)
	nonce := make([]byte, 12) // GCM 标准 nonce 长度
	rand.Read(nonce)

	if err := EncryptFile(src, dst, key, nonce); err != nil {
		panic(err)
	}
	enc, _ := os.ReadFile(dst)
	lines := strings.Count(string(enc), "\n")
	fmt.Printf("加密完成：源文件 3 行 -> 加密文件 %d 行（行数一致）\n", lines)
	fmt.Printf("加密文件首行预览（Base64，不可读）：%.40s...\n", strings.SplitN(string(enc), "\n", 2)[0])
}
```

运行结果（示例）：

```
加密完成：源文件 3 行 -> 加密文件 3 行（行数一致）
加密文件首行预览（Base64，不可读）：3F2k9vQmX1pL7sD8nR4tY6uI0oP2aS5dF...
```

## 三、本讲小结

- 大文件按行处理：`bufio.Scanner` 读 + `bufio.Writer` 写，内存占用稳定；
- 一行原文对应一行 Base64 密文，结构简单，解密时同样逐行还原；
- 演示用的 key/nonce 是随机生成的，实战中钥匙要妥善保管（比如用 RSA 保护起来，见混合加密那一讲）。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 标准库文档。
