---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 44212
source: 综合讨论
publishedAt: 2026-11-12
---

## 背景

在社区写帖、维护文档、让 Agent 产出带图内容时，都绕不开一个问题：图片放哪。付费对象存储当然稳，但个人低频写作不值当；各类匿名免费图床挂了一茬又一茬，链接失效就是事故。最后我落在一个老方案上：GitHub 公开仓库做存储，jsDelivr 做全球 CDN 分发。零成本、可外链，而且天然适合接进自动化流程。

## 需求与约束

我的硬性要求：可外链、免费、能被脚本或 MCP 工具自动上传并返回 URL、文件名不可变（缓存友好）。顾虑主要是两点：大陆访问的稳定性，以及 jsDelivr 激进的缓存策略。

## 搭建步骤

1. 建一个公开仓库（如 `img-bed`），按 `项目/年月/` 分目录，别一股脑堆根目录。
2. 上传即 commit。手动场景直接 push；自动化场景走 git CLI 或 GitHub MCP 工具。
3. 引用地址固定格式：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

4. 把上传动作封装成脚本，Agent 生成插图后一行命令拿到 CDN URL：

```bash
#!/usr/bin/env bash
name="$(date +%Y%m)-$(md5sum "$1" | cut -c1-8).webp"
cwebp -q 82 "$1" -o "$name"
git -C ~/img-bed add . && git -C ~/img-bed commit -m "add $name" && git -C ~/img-bed push
echo "https://cdn.jsdelivr.net/gh/<user>/img-bed@main/$name"
```

用内容哈希做文件名是关键：同一张图重复上传就是同一个 URL，天然幂等。

## 踩坑点

- **同路径覆盖不生效**。jsDelivr 对 gh 资源缓存很激进，分支引用约 12 小时、版本引用更久。改文件不换名，读者看到的多半还是旧图。purge 接口存在但别当常规手段，正确姿势就是文件名不可变。
- **体积限制**。单文件超过约 20MB 可能被 CDN 拒绝，仓库总量建议控制在 1GB 上下。图片务必先压：WebP q82 基本无感，体积减半起步。
- **大陆可用性会波动**。jsDelivr 在大陆的解析和节点近年时好时坏，动手前先用目标读者的网络环境实测；不要把它当唯一方案，URL 里的域名部分保持可替换（sed 一把就行），留好切自建/OSS 的后路。
- **公开仓库即公开内容**。所有图片任何人可见，推之前确认没有隐私信息。另外 GitHub ToS 要求仓库以代码为主，别拿来当无限存储倾倒。
- **API 频控**。走 Contents API 逐张上传容易撞 5000 次/小时的限额，批量入库用 git push 更省事。

## 可复用建议

- 命名规则统一为「日期 + 内容哈希」，一次写入永不覆盖，这是整套方案的根基。
- 需要长期稳定的 URL 就打 tag（如 `v2026.03`）再引用，不可变缓存反而成了优点。
- 维护一个 manifest（JSON 记录本地路径、CDN URL、尺寸、哈希），Agent 写帖时直接查表引用，日后全量迁移也方便。
- 把上传脚本包装成 MCP 工具暴露给 Agent，比每次让它自己拼 git 命令可靠得多，失败重试和日志也好做。

## 总结

GitHub + jsDelivr 不是什么新招，但对个人写作、文档配图、Agent 生成内容这类低频中量场景，成本和稳定性的平衡很好。核心三条：文件名不可变、控制仓库体积、实测目标网络再下结论。它替代不了生产级 CDN，但作为个人工程侧的默认方案，够用了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/4dd0fda810538999.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/2e803d8133fef09b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets3@main/images/2026-11-12/ee27d83038b68891.png)

