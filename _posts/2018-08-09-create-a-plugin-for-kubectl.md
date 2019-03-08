---
layout: post
categories: Kubernetes
title: 编写 kubectl 插件
date: 2018-08-09 22:19:35 +0800
description: 为 kubectl 编写一个插件
keywords: kubectl,plugin,插件
catalog: true
multilingual: false
tags: Kubernetes
---

> 最近忙的晕头转向，博客停更了 1 个月，感觉对不起党、对不起人民、对不起 ~~CCAV~~...不过在忙的时候操作 Kubernetes 集群要频繁的使用 `kubectl` 命令，而在多个 NameSpace 下来回切换每次都得加个 `-n` 简直让我想打人；索性翻了下 `kubectl` 的插件机制，顺便写了一个快速切换 NameSpace 的小插件，以下记录一下插件编写过程

## 一、插件介绍

`kubectl` 命令从 `v1.8.0` 版本开始引入了 alpha feature 的插件机制；在此机制下我们可以对 `kubectl` 命令进行扩展，从而编写一些自己的插件集成进 `kubectl` 命令中；**`kubectl` 插件机制是与语言无关的，也就是说你可以用任何语言编写插件，可以是 `bash`、`python` 脚本，也可以是 `go`、`java` 等编译型语言；所以选择你熟悉的语言即可**，以下是一个用 `go` 编写的用于快速切换 NameSpace 的小插件，运行截图如下:

**所谓: 开局一张图，功能全靠编 😂**
![swns.gif](https://mritd.oss.link/markdown/6t89g.gif)

当前插件代码放在 [mritd/swns](https://github.com/mritd/swns) 这个项目下面

## 二、插件加载

`kubectl` 插件机制目前并不提供包管理器一样的功能，比如你想执行 `kuebctl plugin install xxx` 这种操作目前还没有实现(个人感觉差个规范)；所以一旦我们编写或者下载一个插件后，我们只有正确放在特定目录才会生效；

**目前插件根据文档描述只有两部分内容: `plugin.yaml` 和其依赖的二进制/脚本等可执行文件**；根据文档说明，`kubectl` 会尝试在如下位置查找并加载插件，所以我们只需要将 `plugin.yaml` 和相关二进制放在在对应位置即可:

- `${KUBECTL_PLUGINS_PATH}`: 如果这个环境变量定义了，那么 `kubectl` **只会**从这里查找；**注意: 这个变量可以是多个目录，类似 PATH 变量一样，做好分割即可**
- `${XDG_DATA_DIRS}/kubectl/plugins`: 关于这个变量具体请看 [XDG System Directory Structure](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)，我了解也不多；**如果这个变量没定义则默认为 `/usr/local/share:/usr/share`**
- `~/.kube/plugins`: 这个没啥可说的，我推荐还是将插件放在这个位置比较友好一点

所以最终插件目录结构类似这样:

``` sh
➜  ~ tree .kube
.kube
├── config
└── plugins
    └── swns
        ├── plugin.yaml
        └── swns
```

## 三、Plugin.yaml

`plugin.yaml` 这个文件实际上才是插件的核心，在这个文件里声明了插件如何使用、调用的二进制/脚本等重要配置；**一个插件可以没有任何脚本/二进制可执行文件，但至少应当有一个 `plugin.yaml` 描述文件**；目前 `plugin.yaml` 的结构如下:

``` yaml
name: "targaryen"                 # 必填项: 用于 kuebctl 调用的插件名称
shortDesc: "Dragonized plugin"    # 必填项: 用于 help 该插件时的简短描述
longDesc: ""                      # 非必填: 插件的长描述
example: ""                       # 非必填: 插件的使用样例
command: "./dracarys"             # 必填项: 插件实际执行的文件位置，可以相对路径 or 绝对路径，或者在 PATH 里也行
flags:                            # 非必填: 插件支持的 flag
  - name: "heat"                  # 必填项: 如果你写了支持的 flag，那么此项必填
    shorthand: "h"                # 非必填: 该选项的缩短形式
    desc: "Fire heat"             # 必填项: 同样每个 flag 都必须书写描述
    defValue: "extreme"           # 非必填: 默认值
tree:                             # 允许定义一些子命令
  - ...                           # 子命令支持同样的设置属性(我想知道子命令的子命令的子命令支不支持...我还没去试过)
```

## 四、插件环境变量

在编写插件时，**有时插件运行时需要获取到一些参数，比如 `kubectl` 执行时的全局 flag 等，**为了方便插件开发者，`kuebctl` 的插件机制提供一些预置的环境变量方便我们读取；即如果你用 `bash` 写插件，那么这些变量你只需要 `${xxxx}` 即可拿到，然后做一些你想做的事情；这些变量目前支持如下:

- `KUBECTL_PLUGINS_CALLER`: `kubectl` 二进制文件所在位置；**作为插件编写者，我们无需关系 api server 是否能联通，因为配置是否正确应当由使用者决定；在需要时我们只需要直接调用 `kubectl` 即可；**比如在 `bash` 脚本中执行 `get pod` 等
- `KUBECTL_PLUGINS_CURRENT_NAMESPACE`: 当前 `kuebctl` 命令所对应的 NameSpace，**插件机制确保了该值一定正确；即这是经过解析了 `--namespace` 选项或者 `kubeconfig` 配置后的最终结果；作为插件编写者，我们无需关心处理过程**；想详细了解的的可以去看源码，以及 `Cobra` 库(Kubernetes 用这个库解析命令行参数和配置)
- `KUBECTL_PLUGINS_DESCRIPTOR_*`: 插件自己本身位于 `plugin.yaml` 中的描述信息，比如 `KUBECTL_PLUGINS_DESCRIPTOR_NAME` 输出 `plugin.yaml` 下的 `name` 属性；一般可以用作插件输出自己的帮助文档等
- `KUBECTL_PLUGINS_GLOBAL_FLAG_*`: 获取 `kubectl` 所有全局 flag 值的变量，比如 `KUBECTL_PLUGINS_GLOBAL_FLAG_NAMESPACE` 能拿到 `--namespace` 选项的值
- `KUBECTL_PLUGINS_LOCAL_FLAG_*`: 同上面类似，只不过这个是获取插件自己本身 flag 的值，个人认为在脚本语言中，比如 `bash` 等处理选项不怎么好用时，可以考虑直接从变量拿

以上变量我并未都测试，具体以测试为准，**删库跑路等情况本人概不负责**

## 五、写一个切换 NameSpace 的插件

前面墨迹一大堆只是为了描述清楚 **要写一个插件应该怎么干** 的问题，下面开始 **这么干**

### 5.1、编写配置

上面已经介绍好了 `plugin.yaml` 怎么写，那么根据我自己的需求，我写的这个切换 NameSpace 插件的名字暂且叫做 `swns`；我希望 `swns` 执行后接受一个 NameSpace 的字符串，然后调用 `kuebctl config` 去设置当前默认的 NameSpace，这样在后续命令中我就不用再一直加个 `-n xxx` 参数了；同时我希望使用更方便点，当执行 `swns` 命令时，如果不提供 NameSpace 的字符串，那我就弹出下拉列表供用户选择；综上需求自己想明白后，就写一个 `plugin.yaml`，如下:

``` yaml
name: "swns"
shortDesc: "Switch NameSpace"
longDesc: "Switch Kubernetes current context namespace."
example: "kubectl plugin swns [NAMESPACE]"
command: "./swns"
```

### 5.2、编写插件

上面 `plugin.yaml` 已经定义好了，那么接下来就简单了，撸代码实现了就好；代码如下:

``` go
// 注意: 下面的模板语法大括号中间没有空格，此处空格是为了防止博客渲染出错

package main

import (
	"fmt"
	"os"
	"os/exec"
	"strings"

	"github.com/mritd/promptx"
)

func main() {

	// 先拿到当前的 context
	cmd := exec.Command("kubectl", "config", "current-context")
	cmd.Stdin = os.Stdin
	cmd.Stderr = os.Stderr

	b, err := cmd.Output()
	checkAndExit(err)
	currentContext := strings.TrimSpace(string(b))

	// 如果提供了 NameSpace 字符串，我直接改就行了
	if len(os.Args) > 1 {
		cmd = exec.Command("kubectl", "config", "set-context", currentContext, "--namespace="+os.Args[1])
		cmd.Stdout = os.Stdout
		checkAndExit(cmd.Run())
		fmt.Printf("Kubernetes namespace switch to %s.\n", os.Args[1])
	} else {
		// 没提供我就得先把所有的 NameSpace 弄出来
		cmd = exec.Command("kubectl", "get", "ns", "-o", "template", "--template", "{ { range .items } }{ { .metadata.name } } { { end } }")
		b, err = cmd.Output()
		checkAndExit(err)
		allNameSpace := strings.Fields(string(b))

		// 弄到所有的 NameSpace 后，我在弄一个下拉列表(这是我自己造的一个下拉列表库)
		cfg := &promptx.SelectConfig{
			ActiveTpl:    "»  { { . | cyan } }",
			InactiveTpl:  "  { { . | white } }",
			SelectPrompt: "NameSpace",
			SelectedTpl:  "{ { \"» \" | green } }{ {\"NameSpace:\" | cyan } } { { . } }",
			DisPlaySize:  9,
			DetailsTpl:   ` `,
		}
		s := &promptx.Select{
			Items:  allNameSpace,
			Config: cfg,
		}

		// 用户选中一个 NameSpace 后我就拿到了想要设置的 NameSpace 字符串
		selectNameSpace := allNameSpace[s.Run()]

		// 跟上面套路一样，写进去就行了
		cmd = exec.Command("kubectl", "config", "set-context", currentContext, "--namespace="+selectNameSpace)
		cmd.Stdout = os.Stdout
		checkAndExit(cmd.Run())
		fmt.Printf("Kubernetes namespace switch to %s.\n", selectNameSpace)
	}
}

func checkErr(err error) bool {
	if err != nil {
		fmt.Println(err)
		return false
	}
	return true
}

func checkAndExit(err error) {
	if !checkErr(err) {
		os.Exit(1)
	}
}
```

最后编译后放到上面所说的插件加载目录即可

到此，**"全局一张图，功能全靠编"** 图上面也有了，编的的也差不多 😂

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
