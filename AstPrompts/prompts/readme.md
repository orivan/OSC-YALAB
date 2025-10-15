## 如何增加远程库(为了便于使用github agent)
- 在 github 创建一个库(新建的库不能初始化)并复制库地址
- 增加远程地址
```bash
git remote -v
git remote add github git@github.com:xxxx.git
git remote -v
```
- 把所有内容及标签推送到新的远程地址
```bash
git push --all github
git push --tags github
```
- 设置默认的推送及拉取库
```bash
# 切换到主分支 master
git push -u github master/main
# 如果有其它分支也执行此操作
```
