
```bash
go mod init xxx

go env -w GOOS=linux
go run main.go


go env -w GOOS=windows 
go run main.go


# 使用aliyun镜像
go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct

# 安装库
go get github.com/google/uuid
go get -u github.com/SebastiaanKlippert/go-wkhtmltopdf

# 编译
go build main.go

# 打包
docker build -t tqbb/html2pdf-go:0.1 .

# 运行
docker run -ti -d --rm --name html2pdf-go -p 8080:8080 tqbb/html2pdf-go:0.1


```