# 个人防病毒 checklist：补丁、附件、口令、备份

> 这一讲是纯防御清单：把"病毒防范"落成普通人和小团队能直接执行的动作，外加一段文件完整性校验的演示代码。

## 一、补丁：第一优先级

- 系统和软件的更新提示出来就装，安全补丁不要拖；
- 2017 年的 MS17-010 补丁就是例子：补丁早就发布，打了的机器安然无恙；
- 很久没开机的电脑：先确认口令已改、补丁已装，再联网。

## 二、附件：不乱点

- 不轻易打开来路不明的 doc、rtf 等后缀附件；
- 压缩包先确认发送人再解压；
- 链接先看域名再点开，短链接尤其小心。

## 三、口令：内网不复用

- 内网多台机器用同一账号同一密码的，尽快改掉——一台失守等于全军覆没；
- 密码用高强度且每处不同，记不住就用密码管理器。

## 四、备份：最后的保险

- 重要数据遵循"三份拷贝、两种介质、一份离线"；
- 备份文件本身用加密保护（见本系列的文件加密几讲），上传网盘前先加密；
- 定期做恢复演练：备份能不能还原，演练一次才知道。

## 五、动手演示：给重要文件做"完整性指纹"

备份有了，还要能发现"备份被人动过"。下面这段程序用 SHA-256 给文件算指纹：先记录正常指纹，之后随时校验，一旦文件被改动，指纹立刻对不上。

```go
package main

import (
	"crypto/sha256"
	"fmt"
	"os"
	"path/filepath"
)

func fingerprint(path string) (string, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%x", sha256.Sum256(data)), nil
}

func main() {
	dir, _ := os.MkdirTemp("", "integrity")
	defer os.RemoveAll(dir)
	f := filepath.Join(dir, "重要资料.txt")
	os.WriteFile(f, []byte("2026 年度备份清单 v1"), 0644)

	// 1. 记录正常指纹
	good, err := fingerprint(f)
	if err != nil {
		panic(err)
	}
	fmt.Printf("记录指纹：%.16s...\n", good)

	// 2. 文件未被改动：校验通过
	now, _ := fingerprint(f)
	fmt.Printf("未改动时校验：%v\n", now == good)

	// 3. 模拟文件被改动一个字节：校验告警
	data, _ := os.ReadFile(f)
	data[0] ^= 0x01
	os.WriteFile(f, data, 0644)
	tampered, _ := fingerprint(f)
	fmt.Printf("改动后校验：%v（指纹变化，发出告警）\n", tampered == good)
}
```

运行结果（示例）：

```
记录指纹：9f2c41d8e5b07a3c...
未改动时校验：true
改动后校验：false（指纹变化，发出告警）
```

## 六、本讲小结

- 防御四件套：**及时打补丁、不乱开附件、口令不复用、备份三二一**；
- 指纹校验是备份的搭档：备份保证"有"，指纹保证"没被改"；
- 这些动作都不需要高深技术，难的是坚持执行。

本文为学习笔记（编纂），用自己的话重写；知识点源自公开的网络安全常识。
