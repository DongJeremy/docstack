# Go配置

## Linux

```bash
mkdir -p /opt/goworks/{bin,src,pkg}

cat << EOF >> /etc/bashrc
export GOPATH=/usr/share/gocode
export GOPROXY=https://goproxy.io
EOF

source /etc/bashrc
mkdir -p /opt/goworks/src/github.com/davidddw/gopl

go env
go version
export GOPATH=/opt/goworks
```

## windows

```bash
GOROOT D:\\Program\\go
GOPATH D:\\Program\\GoWorks
PATH: $PATH:$GOROOT\\bin
```

## 编译

```bash
go build -ldflags "-s -w" xxx.go
upx --brute binaryfile
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags "-s -w"　//编译为linux 64位系统下的程序
CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build
CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build main.go
```

## go mod

```bash
go mod init project
go mod tidy
```

## VSCode环境配置

````json
{
    "git.path": "D:\\Program\\PortableGit\\bin\\git.exe",
    "git.autofetch": true,
    "git.enableSmartCommit": true,
    "terminal.integrated.shell.windows": "C:\\Windows\\System32\\cmd.exe",
    "terminal.integrated.rendererType": "dom",
    "go.buildOnSave": "workspace",
    "go.lintOnSave": "package",
    "go.vetOnSave": "package",
    "go.buildTags": "",
    "go.buildFlags": [],
    "go.lintFlags": [],
    "go.vetFlags": [],
    "go.coverOnSave": false,
    "go.useCodeSnippetsOnFunctionSuggest": false,
    "go.formatTool": "goimports",
    "go.goroot": "E:\\d05660\\GoLang\\go",
    "go.gopath": "E:\\d05660\\GoLang\\gopath",
    "editor.rulers": [
        150
    ],
    "go.useLanguageServer": true,
    "workbench.startupEditor": "newUntitledFile"
}
````

```powershell
GOROOT E:\\d05660\\GoLang\\go
GOPATH E:\\d05660\\GoLang\\gopath
GOPROXY https://goproxy.io/
PATH: $PATH:$GOROOT\\bin
```

创建文件夹

```
E:\\d05660\\GoLang\\gopath\bin
E:\\d05660\\GoLang\\gopath\src
E:\\d05660\\GoLang\\gopath\pkg
```

如果不想把插件安装在C盘的话，可以自己新建一个文件来存储插件，然后在快捷方式的目标中修改路径

```--extensions-dir "D:\Program Files\Microsoft VS Code\extensions"```