# openpilot `.git`目录巨大问题

openpilot 仓库年份久、历史提交多、早期提交过不少二进制资源；完整 clone 下来，`.git`可达**几百 MB ~ 1GB+**，工作树源码本身并不大，体积几乎全部来自 git 历史对象GitHub。
## 最佳实践：重新浅克隆，一劳永逸
openpilot 开发调试绝大多数场景**不需要完整历史**

```sh
# --depth 1：只拉取最新1个提交；--single‑branch只拉主分支；子模块同时浅克隆
git clone --depth 1 --single-branch --recurse-submodules --shallow-submodules \
  https://github.com/commaai/openpilot.git

  # https://github.com/autowarefoundation/autoware.git
```
