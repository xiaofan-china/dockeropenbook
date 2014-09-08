## 构建属于自己的分支版本开发环境

本文主要针对 dockercn 在构建自身分支版本时所遇到的问题和总结，所有命令都在 Ubuntu14.04 64bit 系统下执行。**为了保证下载操作的成功执行，请务必使用速度较快的 VPN。**

### 准备工作

想要构建自己的分支版本，那么必然需要先进行 Fork 官方或任意一个上游 Docker 分支，本文全部以 Fork 官方库为例。

由于官方分支为自包容的，因此需要 Fork 的库只有一个，即 `github.com/docker/docker` 一个。当然，如果您希望对 `github.com/docker/libcontainer` 包也作出自定义修改，那么也可以克隆该库。这里之所以说我们不用克隆是因为官方分支吧所有第三方依赖都放置在 `github.com/docker/docker/vendor` 中。

鉴于 Go 语言开发环境的特殊性，我们所克隆的库需要放置在自身的目录，即 `$GOPATH/src/github.com/dockercn/docker`。这个时候问题就出现了，如果您尝试使用 `go install` 构建二进制，将会出现一长串的导入路径未找到的错误，或者其实你是在构建官方库，任何对 `github.com/dockercn/docker` 下的修改都是无效的。

### 替换导入路径

文件系统的目录结构对于 Go 语言来说就是包导入路径的唯一判断依据，因此就算您将文件拷贝到自身的目录下，但引用的实际上还是官方分支的代码。因此，我们需要将所有 `.go` 代码文件和 `Dockerfile` 文件中的相应路径替换为我们自己的导入路径。

假如您打算通过手动替换的话，那么是非常不明智的，这里有一个我们已经写好的通用脚本可以帮助您（[cpath.go](https://github.com/dockercn/docker/tree/master/hack/cpath)）： 

```go
package main

import (
	"bytes"
	"flag"
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"path"
	"path/filepath"
	"strings"

	"github.com/Unknwon/com"
)

var (
	from = flag.String("from", "", "old import path")
	to   = flag.String("to", "", "new import path")
)

func main() {
	flag.Parse()

	if len(*to) == 0 {
		log.Fatalln("Option '-to' cannot be empty")
	} else if len(*from) == 0 {
		log.Fatalln("Option '-from' cannot be empty")
	}

	if (*to)[len(*to)-1] != '/' {
		*to += "/"
	}
	if (*from)[len(*from)-1] != '/' {
		*from += "/"
	}

	gopath := com.GetGOPATHs()[0]
	rootPath := path.Join(gopath, "src", *to)
	if !com.IsDir(rootPath) {
		log.Fatalf("Given path(%s) does not exist or not a directory", rootPath)
	}

	if err := filepath.Walk(rootPath, func(path string, info os.FileInfo, err error) error {
		if info.IsDir() || (!strings.HasSuffix(path, ".go") && !strings.HasSuffix(path, "Dockerfile")) {
			return nil
		}
		fmt.Println(path)

		data, err := ioutil.ReadFile(path)
		if err != nil {
			return err
		}
		data = bytes.Replace(data, []byte(*from), []byte(*to), -1)
		if err = ioutil.WriteFile(path, data, info.Mode()); err != nil {
			return err
		}
		return nil
	}); err != nil {
		log.Fatalf("Fail to walk: %v", err)
	}
}
```

比如我们已经将复制的代码放置于 `$GOPATH/src/github.com/dockercn/` 目录下，则我们实际上要进行的操作就是将所有 `github.com/docker/` 前缀替换为 `github.com/dockercn/`。

- 拉取该脚本需要的依赖库，在同个目录下执行 `go get -v`。
- 执行脚本：`go run cpath.go -from github.com/docker/ -to github.com/dockercn/`

如此，就完成了替换导入路径的工作。

### 开始构建

现在，我们就要开始构建最基本的开发环境了。

进入到 `$GOPATH/src/github.com/dockercn/docker` 目录下，执行 `sudo make build`。这里的过程比较长，主要就是拉取官方的 Ubuntu 镜像然后开始一系列的初始化操作。

环境构建完成后，需要进行测试操作（如果是未经任何修改的代码此步其实没有必要）：`sudo make test`。

接着，就是需要生成我们需要带有 `deamon` 命令的 server 版二进制了（Docker 分 client 和 server 两个版本）：`sudo make binary`。该命令会在结束时将二进制存放至 `./bundles/<version>-dev/binary/`。

最后，需要使用新构建的二进制替换掉系统原有的（该步操作可能需要重启机器才能生效）：

	sudo service docker stop ; sudo cp $(which docker) $(which docker)_ ; sudo cp ./bundles/<version>-dev/binary/docker-<version>-dev $(which docker);sudo service docker start
	
恭喜您！开始享受 Docker Hack 之旅吧！