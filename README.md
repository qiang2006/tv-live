# live/live.m3u

给 CM311-1a-YST 盒子用的直播源列表（央视 + 卫视），每一条都用 `ffprobe` 验过能解出视频轨。

## 盒子怎么用它

开机时 `/system/bin/tv2-maint.sh` 会按顺序试这几个地址（`REMOTES`）：

```
https://cdn.jsdelivr.net/gh/Gathin23/cleanyourtv@main/live/live.m3u
```

校验（首行 `#EXTM3U`、`#EXTINF` 不少于 50 条）通过才**原子替换**盒子上的本地列表；
第一个源失败就试下一个，**全失败就保留旧列表**，电视照常能看，不会弹任何提示。

> 仓库原来叫 `tv-vibehack`，已改名为 `cleanyourtv`。旧名字还能用（GitHub 做重定向），
> 但那是重定向，一旦有人抢注 `tv-vibehack`，jsDelivr 就会拉到别人的东西 —— 所以盒子里用的是新名字。

### 加一个 Gitee 镜像（可选，国内更快）

GitHub/jsDelivr 在国内偶尔被 DNS 污染或限速，加一条 Gitee 源就是免费的冗余：

```powershell
git remote add gitee https://gitee.com/<你的用户名>/<仓库名>.git
git push gitee main
```

然后在盒子上把地址填进去（空格分隔，按优先级从高到低）：

```sh
adb shell "sed -i 's|^REMOTES_DEFAULT=.*|REMOTES_DEFAULT=\"https://gitee.com/<用户名>/<仓库名>/raw/main/live/live.m3u https://cdn.jsdelivr.net/gh/Gathin23/cleanyourtv@main/live/live.m3u\"|' /system/bin/tv2-maint.sh"
```

要求：**Gitee 仓库必须是公开的**（盒子不带任何凭据，私有仓库的 raw 要登录）。
地址格式是 `/raw/<分支>/<路径>`，和 GitHub 的 raw 不一样，别写错。

## 更新方法

```powershell
cd F:\TV2

# 1) 重新合并公开源 + 逐条验源 + 推给盒子（几分钟）
.\tools\refresh_live.ps1 -Box 192.168.2.134

# 2) 把盒子上"正在用"的那份抓回仓库（自动校验、不会拉错文件）
.\tools\publish_live.ps1 -Box 192.168.2.134 -Commit

# 3) 推上去，下次开机所有盒子自动生效
git push
```

`publish_live.ps1` 从播放器的 `shared_prefs` 里读 `IPTV_SOURCE_CURRENT` 的真实路径，
所以不会像 `adb pull .../iptv-*.txt` 那样把一堆历史列表一起拉下来。

推送后 jsDelivr 对 `@main` 有缓存（约 12 小时），盒子下次开机拿到的可能是缓存版本，
最迟一天后生效。要立刻生效，可以在盒子上手动跑一次：

```powershell
adb shell 'TV2_ONCE=1 sh /system/bin/tv2-maint.sh'
```

## 注意

- 这个文件是 **LF** 结尾。用会改成 CRLF 的编辑器保存会让 `#EXTM3U` 那行带上 `\r`，校验会失败。
- 列表里的源都是公开源，不涉及任何隐私；但公共源会陆续失效，失效了就重跑第 1 步。
