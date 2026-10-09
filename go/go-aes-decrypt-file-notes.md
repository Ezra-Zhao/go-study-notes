# 把加密文件逐行还原：DecryptFile 的思路

> 上一讲把文本文件逐行加密了，这一讲反向操作：逐行读加密文件 → 解密 → 写回原文。同样很短，大纲只给了 `DecryptFile("encryptFile_test","qtest.txt")` 的调用形式和实现要点，不编造细节。

## 一、思路拆解

解密是加密的镜像，每一步反着来：

1. **按行读加密文件**：`bufio.Scanner` 逐行读，每行是一段 Base64；
2. **Base64 解码 + AES 解密**：`DecryptByAes` 负责把一行密文还原成原文行；
3. **写回文件**：`bufio.NewWriter` 把原文行逐行写入目标文件；
4. **打印结果**：完成后输出"文件解密成功，生成文件名为…，文件大小为…Byte"这样的确认信息。

参数形式参考大纲：`DecryptFile(加密文件, 还原目标文件)`。

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
)

// DecryptByAes 解密单行 Base64 密文
func DecryptByAes(line string, key, nonce []byte) (string, error) {
	ct, err := base64.StdEncoding.DecodeString(line)
	if err != nil {
		return "", err
	}
	block, err := aes.NewCipher(key)
	if err != nil {
		return "", err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}
	plain, err := gcm.Open(nil, nonce, ct, nil)
	if err != nil {
		return "", err
	}
	return string(plain), nil
}

// DecryptFile 逐行读取加密文件 src，解密后写入 dst
func DecryptFile(src, dst string, key, nonce []byte) error {
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

	w := bufio.NewWriter(out)
	defer w.Flush()
	scanner := bufio.NewScanner(in)
	for scanner.Scan() {
		plain, err := DecryptByAes(scanner.Text(), key, nonce)
		if err != nil {
			return err
		}
		fmt.Fprintln(w, plain)
	}
	if err := scanner.Err(); err != nil {
		return err
	}
	w.Flush()
	info, err := out.Stat()
	if err != nil {
		return err
	}
	fmt.Printf("文件解密成功，生成文件名为%s，文件大小为%d Byte\n", dst, info.Size())
	return nil
}

func main() {
	dir, _ := os.MkdirTemp("", "aesdec")
	defer os.RemoveAll(dir)
	encFile := filepath.Join(dir, "encryptFile_test")
	decFile := filepath.Join(dir, "qtest.txt")

	key := make([]byte, 32)
	rand.Read(key)
	nonce := make([]byte, 12)
	rand.Read(nonce)

	// 先造一个加密文件（加密逻辑见上一讲，这里内联最小实现）
	block, _ := aes.NewCipher(key)
	gcm, _ := cipher.NewGCM(block)
	lines := []string{"第一行：账号清单", "第二行：备份说明", "第三行：联系人"}
	var sb string
	for _, l := range lines {
		sb += base64.StdEncoding.EncodeToString(gcm.Seal(nil, nonce, []byte(l), nil)) + "\n"
	}
	os.WriteFile(encFile, []byte(sb), 0644)

	// 解密并验证
	if err := DecryptFile(encFile, decFile, key, nonce); err != nil {
		panic(err)
	}
	back, _ := os.ReadFile(decFile)
	fmt.Printf("还原内容一致：%v\n", string(back) == "第一行：账号清单\n第二行：备份说明\n第三行：联系人\n")
}
```

运行结果（示例）：

```
文件解密成功，生成文件名为/tmp/aesdec1234567890/qtest.txt，文件大小为 72 Byte
还原内容一致：true
```

## 三、本讲小结

- 解密流程：读加密行 → Base64 解码 → `DecryptByAes` → `bufio.Writer` 写回；
- 完成后打印文件名和大小，确认信息是好习惯，出问题第一时间能发现；
- 加密（上一讲）和解密（这一讲）用的 key/nonce 必须同一套，钥匙对不上就解不开。

本文为学习笔记（编纂），用自己的话重写；知识点源自 Go 标准库文档。
