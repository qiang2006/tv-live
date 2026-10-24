# live/live.m3u

给 CM311-1a-YST 盒子用的直播源列表（央视 + 卫视），每一条都用 `ffprobe` 验过能解出视频轨。

## 盒子怎么用它

开机时 `/system/bin/tv2-maint.sh` 会按顺序试这两个地址（`REMOTES`，空格分隔）：

```
1. https://gitee.com/wodegitee33/tv-live/raw/main/live.m3u            ← 主源（国内直连）
2. https://cdn.jsdelivr.net/gh/qiang2006/tv-live@main/live.m3u        ← 兜底（GitHub → jsDelivr）
```

校验（首行 `#EXTM3U`、`#EXTINF` 不少于 50 条）通过才**原子替换**盒子上的本地列表；
第一个源失败就试下一个，**全失败就保留旧列表**，电视照常能看，不会弹任何提示。

实测（2026-10-24，真机 `192.168.2.172`）：

```
playlist updated: 177 channels (https://gitee.com/wodegitee33/tv-live/raw/main/live.m3u, OK 34401)
```

> 上一版是 172 条。2026-10-24 在 `.172` 上重新合并 + 逐条 ffprobe 验源后变成 **177 条**
> （CCTV 51 / 卫视 109 / 地方 11 / 其余 6），旧列表里的频道**没有丢失**，
> 新增 海南卫视、CCTV+1、CCTV+2。

> **2026-10-24 修正**：兜底源原来指向 `Gathin23/cleanyourtv@main/live/live.m3u`，
> 但那条路**一直是死的** —— 本机的 GitHub 凭据没有那个仓库的写权限（403），
> 所以 `live/` 从未推上去，jsDelivr 返回 **HTTP 404**。现已改为独立的
> `qiang2006/tv-live`（结构同 Gitee 那个小仓库，`live.m3u` 在**仓库根**）。
> 盒子上实测：`OK 34401`，177 条。

## 两个源分别在哪

| | 地址 | 内容 | 谁维护 |
|---|---|---|---|
| 主源 | `gitee.com/wodegitee33/tv-live` | **只有** `live.m3u` + `README.md`（都在根） | 本仓库 `live/` 目录的 subtree 推送 |
| 兜底 | `github.com/qiang2006/tv-live` | 同上（都是公开小仓库） | 同上（另一个账号） |

> 两个都是 `git subtree push --prefix=live <url> main` 推出来的，所以远端看到的是
> **子树根**：`live.m3u` 直接在仓库根目录，没有 `live/` 这一层。
> 兜底地址是 `.../gh/qiang2006/tv-live@main/live.m3u`，**不要**写成 `@main/live/live.m3u`（那是 404）。


Gitee 那个仓库是**独立的小仓库**，只放列表，和 `suishoubao` / `hazard-collab` 那些私有项目完全隔离。
它必须是**公开仓库** —— 盒子不带任何凭据，私有仓库的 raw 会返回登录页。

## 更新方法

```powershell
cd F:\TV2

# 1) 重新合并公开源 + 逐条验源 + 推给盒子（几分钟）
.\tools\refresh_live.ps1 -Box 192.168.2.134

# 2) 把盒子上"正在用"的那份抓回仓库（自动校验、不会拉错文件）
.\tools\publish_live.ps1 -Box 192.168.2.134 -Commit

# 3) 推主仓库（注意：这条对 Gathin23/cleanyourtv 会 403，见下方说明）
git push

# 4) 推两个"只放列表"的小仓库（subtree，不会带上代码和文档）
git subtree push --prefix=live gitee main                                   # Gitee 主源
git subtree push --prefix=live https://github.com/qiang2006/tv-live.git main # GitHub 兜底
```

`publish_live.ps1` 从播放器的 `shared_prefs` 里读 `IPTV_SOURCE_CURRENT` 的真实路径，
所以不会像 `adb pull .../iptv-*.txt` 那样把一堆历史列表一起拉下来。

**Gitee 没有 CDN 缓存，推完立刻生效**；GitHub 那条走 jsDelivr，`@main` 有约 12 小时缓存，
所以正常情况下盒子拿到的是 Gitee 的新版本，根本用不到缓存。要立刻验证，可以在盒子上跑：

```powershell
adb shell 'TV2_ONCE=1 sh /system/bin/tv2-maint.sh'
adb logcat -d -s TV2_MAINT | tail -3
```

**推完 GitHub 后要 purge 一次 jsDelivr 缓存**（浏览器打开一次即可，或在盒子上跑也行）：

```
https://purge.jsdelivr.net/gh/qiang2006/tv-live@main/live.m3u
```

### 为什么不再推 `Gathin23/cleanyourtv`

那个仓库是**主项目**（260+ 机型、2000+ 提交、还在被别人活跃开发），而本机这个工作副本
`git rev-list --count` 显示 **落后 2079 个提交**。往里推 main 会被拒（非快进），强推会毁掉整个项目。
所以列表改推到**独立小仓库** `qiang2006/tv-live`，和 Gitee 那个完全对称。
`git push`（第 3 步）对那个仓库仍会 403，属于预期 —— 列表发布不依赖它。

## 注意

- 这个文件是 **LF** 结尾。用会改成 CRLF 的编辑器保存会让 `#EXTM3U` 那行带上 `\r`。
  （校验其实能容忍 `\r`，但保持 LF 更省事；仓库根目录的 `.gitattributes` 已经把
  `*.m3u` / `*.sh` 钉成 `eol=lf`，本机 `core.autocrlf=true` 也不会再把它检出成 CRLF。）
- 列表里的源都是公开源，不涉及任何隐私；但公共源会陆续失效，失效了就重跑第 1 步。
- Gitee 的 raw 地址格式是 `/raw/<分支>/<路径>`，和 GitHub 的 `raw.githubusercontent.com` 不一样。
- **两个小仓库都必须是公开的**：盒子上不存任何凭据，私有仓库的 raw 会返回登录页，
  校验会失败（首行不是 `#EXTM3U`），盒子就静默保留旧列表。
