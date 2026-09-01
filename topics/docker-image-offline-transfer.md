# Docker 镜像离线打包与导入

用于没有镜像仓库或需要通过文件在机器之间迁移 Docker 镜像的场景。

## 打包为 tar.gz

推荐直接通过管道压缩：

```bash
docker save myapp:v1.0 | gzip > myapp-v1.0.tar.gz
```

如果安装了 `pv`，可以观察传输/压缩进度：

```bash
docker save myapp:v1.0 | pv | gzip > myapp-v1.0.tar.gz
```

也可以分两步执行：

```bash
docker save -o myapp-v1.0.tar myapp:v1.0
gzip myapp-v1.0.tar
```

## 目标机器导入

兼容性较好的方式：

```bash
gzip -dc myapp-v1.0.tar.gz | docker load
```

等价写法：

```bash
gunzip -c myapp-v1.0.tar.gz | docker load
```

较新的 Docker 通常也支持直接读取压缩包：

```bash
docker load -i myapp-v1.0.tar.gz
```

导入后检查镜像：

```bash
docker images
```

## 一次打包多个镜像

`docker save` 可以一次保存多个镜像及其标签：

```bash
docker save image1:v1 image2:v2 image3:v3 | gzip > images.tar.gz
```

目标端统一导入：

```bash
gzip -dc images.tar.gz | docker load
```

## 关键区别

镜像迁移应使用：

```text
docker save  <-> docker load
```

不要混用：

```text
docker export <-> docker import
```

`save/load` 面向 **镜像**，会保留镜像层、标签和镜像元数据，适合完整迁移镜像。

`export/import` 面向 **容器文件系统**，导出的主要是容器当前文件系统快照，不适合用来完整备份和迁移 Docker 镜像。
