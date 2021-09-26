---
title: RuleX-构建编译
support: false
---

# 构建编译
首先需要克隆代码到本地:
```sh
git clone https://github.com/wwhai/rulex.git
```

## 编译
```sh
make build
```
## Docker打包
```sh
make docker
```

## 统计代码
```sh
make clocs
```

## 单元测试
```sh
make cover
```

> 本地需要有 Golang 的环境和 Make工具链