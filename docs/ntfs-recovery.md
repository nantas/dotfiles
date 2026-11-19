# Linux 数据分区异常关机后的恢复

适用于使用 ntfs3 挂载的 NTFS 数据卷。先确认设备、保存证据与备份，再决定修复；
不能把清除 dirty 标记或挂载成功当成数据完整性证明。本流程不由 chezmoi 自动执行。

**本机 `~/data` 已于同日迁移为 ext4，恢复与比较通过，重启验收待完成。**
下述 NTFS 处置步骤只用于仍为 NTFS 的卷；不要对现在的 ext4 数据卷运行 ntfsfix。

## 2026-11-09：证据与原因判断

依据当日[调查报告](reports/handoff-linux-work-2026-11-09.md)及本机只读复核：

| 证据 | 含义与边界 |
| --- | --- |
| 数据卷是 Kingston NV2 上约 586 GiB 的 `nvme0n1p5`，挂到 `~/data` | Linux 根目录是 p3/ext4；约 7.3 TiB 的另一块 NTFS 磁盘不是本次目标 |
| 本次 09:13、10:00 的内核日志均报 `volume is dirty and "force" flag is not set!` | ntfs3 拒绝挂载有待检查标记的卷，不是目录或代理服务故障 |
| 报告记录 `$MFTMirr does not match $MFT (record 3)` | 文件系统元数据确有不一致，不能把问题缩减为一个多余标记 |
| 报告记录实际执行 ntfsfix 后修正镜像，再执行 `ntfsfix -d` | 后一步完整输出未保留；systemd 独立证实 10:02 挂载成功 |
| 迁移前复核卷为 ntfs3/rw，挂载单元 active、Result=success，无 force 参数 | 当时访问已恢复；未重新在已挂载卷上运行 ntfsfix，也未逐项验证文件 |
| EFI/FAT 提示未正常卸载，journald 替换未正常关闭或损坏的日志文件 | 多处留下异常关机迹象；日志文件被替换不等于应用数据已修复 |
| 上一轮启动日志止于 11 月 6 日 18:34，未找到最终卸载/关机完成记录，wtmp 标记 crash | 支持关机未正常完成；日志可能不完整，不能单凭缺失断定具体断电时刻 |
| 报告中的 SMART 无介质错误、累计 Unsafe Shutdowns=7 | 无直接硬件损坏证据；累计值不能归因到这一次。本次 sudo 需密码，未重新读取 SMART |

最符合这些证据的解释是：写回或卸载尚未完成，机器就失去供电或被强制停止，留下
NTFS 元数据不一致和待检查状态。具体是供电中断、长按电源、复位还是卡死后的断电，
现有证据无法区分；也没有证据定位到某个 ntfs3 驱动缺陷。

用户已确认机器上无 Windows 系统。遗留 Windows Boot Manager 启动项不能证明仍能启动
Windows，也没有依据将本次归因于 Windows 快速启动或休眠。若未来恢复双系统，再检查
这两项，并避免 Linux 写入处于 Windows 休眠状态的卷。

本机 fstab 使用 `nofail`，所以数据卷失败时系统仍能启动；项目目录依然不可用。
`x-systemd.device-timeout=10s` 只限制等待设备时间，不能修复文件系统。
清掉失败单元状态也不代表卷已经恢复，必须检查实际挂载来源。

## 为什么第一次修复后仍然挂不上

NTFS 有日志恢复能力，但不能保证每一种异常关机都能自动恢复。内核文档说明 ntfs3
支持日志重放，也明确不推荐用 `force` 绕过 dirty 检查。

`ntfsfix` 只处理部分基本不一致、重置 NTFS 日志并请求 Windows 后续检查，不等同于
`chkdsk`。不带 `-d` 时它会设置待检查标记；因此“修复命令成功”之后 ntfs3 仍因 dirty
拒绝挂载，并不矛盾。`-d` 在工具能够修复并挂载卷时清除标记，但没有补做完整检查。

最初的 ntfsfix 处理只算“应急恢复访问”，当时完整备份、文件校验与完整 NTFS 检查尚未完成。
后续已完成独立备份及 ext4 迁移，见下文；文件比较证明副本一致，不证明异常关机前的数据未受损。
报告中 Codex 在线账号路由超时是另一项未解决问题，不能推断由磁盘故障直接引起。

## 预防：按优先级处理

1. **先做独立备份。** 覆盖项目、未提交/未跟踪文件、Git LFS 和重要应用数据，验证可恢复。
   Git 远端不包含所有本地工作；同一 NVMe 上的另一个分区不算独立备份。
2. **保证正常关机与供电。** 用桌面关机或 `systemctl poweroff`，等机器真正关机后再断插座。
   固定工作站可配置 UPS，并验证低电量自动关机。`sync` 只帮助写回，不替代卸载和正常关机。
3. **只运行 Linux 的工作数据优先迁到 ext4。** 这样可以使用本机原生权限、链接及离线检查工具，
   减少对 Windows NTFS 修复环境的依赖；ext4 仍需要备份和断电保护。
   迁移流程是：独立备份并验证 → 停写 → 重建文件系统 → 恢复 → 更新本机 UUID/挂载配置
   → 验证权限、项目和正常重启。不能直接格式化当前卷，也不承诺无损原地转换。
4. **保留失败可见性。** 暂不更改 nofail；依赖数据卷的后台任务应先确认挂载来源，
   需要强依赖时再单独设计 systemd 的 `RequiresMountsFor=`。卷缺失时不要在裸挂载目录里
   重建项目，否则后续挂载会遮住这些文件。

不将 `force` 加到 fstab，不配置开机自动 `ntfsfix -d`，也不靠换成 ntfs-3g 来掩盖错误。
本机随后经用户授权完成下述迁移；未配置自动修复、供电设置或新的服务依赖。

## 本机迁移结果（2026-11-09）

- 将整个数据卷归档到独立机械盘，源清单约 162 万项，`tar + zstd` 归档约 281.48 GiB。
  已完整读回、计算 SHA-256 并与源文件比较，备份前后清单一致。
- 冻结源卷后再次核对清单，重新校验归档 SHA-256，再只对原数据分区创建 ext4。
  未重建分区表，Linux 根分区、EFI 和机械盘原有数据保持原样。
- 数据恢复完成，恢复后的 tar 完整比较退出码为 0，比较日志为空；普通用户匿名临时文件
  的写入、同步和读回实测通过，`~/data/projects` 已恢复。
- `/etc/fstab` 仅替换数据卷一行：新 UUID、ext4、保留 nofail 与设备等待策略，fsck pass 改为 2；
  不再使用 NTFS 的 uid/gid、dmask/fmask。fstab 检查无错误，挂载单元为 active/mounted。
- ext4 数据卷约 576 GiB，已用约 472 GiB、可用约 99 GiB；格式化时预留 1% 块。
  文件系统元数据与预留块影响可用容量，不能仅凭 df 数字判断是否漏恢复。
- 保留机械盘归档及校验文件。尚未重启，正常重启后的自动挂载验收仍待完成。

2026-11-10：用户确认数据卷上的日常工作正常。复核 fstab UUID 与实际 ext4 卷一致，
fstab 检查无错误；systemd 已生成挂载单元及 `local-fs.target.wants` 依赖，开机会自动尝试挂载。
保留的 nofail 不会禁用自动挂载，只允许挂载失败时继续启动。此次复核仍在迁移前启动的同一轮
系统会话中，未将配置检查当作重启实测。

机器特有的一次性脚本、fstab 原始副本与日志在本机
`~/.local/state/data-ext4-migration-2026-11-09/`，不由 chezmoi 部署。
`status.json` 是首次尝试的停止记录，最终结果看 `resume-status.json`：`complete`、`verified=true`。

### 格式化后的身份检查不能触发再次格式化

首次脚本创建 ext4 并挂载后，身份检查报 `New mount identity mismatch` 而停止，尚未恢复文件。
后续 findmnt、lsblk、UUID 链接及内核日志一致证明挂载正确。可能是刚格式化后的设备元数据
尚未刷新，但首次检查没有保留实际返回值，原因不能完全确定。

续跑只执行恢复、比较及 fstab 更新，不含格式化步骤；使用 udev settle 和直接读取超级块的
blkid 探测确认新卷。遇到同类问题先记录实际 SOURCE/FSTYPE/UUID 与分区标识，确认已经发生
哪些写盘操作，不能把检查失败当作格式化失败而重跑格式化。

### NTFS/WSL 扩展属性的兼容性例外

本次恢复产生 1,780,689 条告警，全部是 `$LXUID`、`$LXGID`、`$LXMOD` 三种旧 WSL 属性
无法写入 ext4，没有其他恢复诊断。它们不属于 ext4 接受的扩展属性命名空间；原值仍留在
备份归档中，未写入新卷。Linux 下的所有者、权限按归档常规头部恢复并通过比较。

本机最小实测证实：这种告警不使 tar 提取返回失败，文件内容及有效 `user.*` 属性仍能恢复；
缺少这类属性时，tar 比较也可能返回成功。因此不能用 `tar --compare` 的退出码声称所有扩展
属性完全等价，必须审阅恢复日志，区分受支持的属性与跨文件系统不兼容项。

未来从这份归档恢复到 ext4 时，可在保留原始归档的前提下，添加精确的
`--xattrs-exclude='$LXUID'`、`--xattrs-exclude='$LXGID'`、`--xattrs-exclude='$LXMOD'`。
该组合已用本机 tar 验证：无上述告警，内容及有效属性保留。不要因此忽略其他属性或文件错误。

## 标准流程

### 1. 确认目标与保存证据（只读）

以下命令先在本机执行。不要沿用历史设备号；NVMe 编号、盘符可能改变。
日志可能包含私人路径，分享或写进仓库前脱敏。

```sh
data_mount="$HOME/data"
mount_unit=$(systemd-escape --path --suffix=mount "$data_mount")
lsblk -o NAME,MODEL,SIZE,FSTYPE,UUID,MOUNTPOINTS
findmnt --fstab --target "$data_mount"
findmnt --mountpoint "$data_mount"
systemctl status "$mount_unit" --no-pager
journalctl -b -u "$mount_unit" --no-pager
journalctl -b -k --no-pager | rg -i 'ntfs3|I/O error|nvme.*(error|timeout|reset)'
journalctl -b -1 -u "$mount_unit" --no-pager
```

`findmnt --target` 在目标未挂载时可能返回承载该目录的根文件系统，验收要用
`--mountpoint` 精确匹配。即使 systemctl 无失败项，也必须检查来源、类型和 rw/ro。

将 fstab UUID 与 lsblk 的型号、容量、分区对应后，手动设置修复目标：

```sh
data_device=/dev/disk/by-uuid/REPLACE_WITH_VERIFIED_UUID
readlink -f "$data_device"
```

**未替换并核对 UUID 就停止。** SMART 读取需管理员权限，目标是对应整盘，不是分区：
`sudo smartctl -H -A /dev/已核对的整盘`。若出现介质错误、反复掉盘或 I/O 错误，停止写入，
优先到独立健康介质做镜像/专业恢复，不反复运行修复程序。

### 2. 停写、备份、确认卷已卸载

当前能访问时，先备份到独立设备并验证；不能访问时，重要数据应先做块级镜像。
修复会修改原卷，尚无可恢复副本时先讨论风险与恢复方案，不直接进入写修复。
只读挂载失败也不要改用 force 强上。

安排维护窗口，保存工作、关闭占用卷的编辑器、终端、agent 和后台任务，再卸载。
执行本流程的终端先离开该卷：

```sh
cd "$HOME"
sudo fuser -vm "$data_mount"
sudo systemctl stop "$mount_unit"
findmnt --source "$data_device"
lsblk -o NAME,FSTYPE,MOUNTPOINTS
```

必须确认该设备在所有已知挂载点均已卸载；有容器/独立挂载命名空间时也需核实。
卸载失败先找占用，不使用 `umount -l`、强制卸载或批量杀进程绕过。
**以下 ntfsfix 命令绝不在已挂载卷上执行。**

### 3. 只读诊断，选择修复分支

先确认工具来源：`command -v ntfsfix`、`pacman -Qo /usr/bin/ntfsfix`。
本次 Arch 的工具属于 `ntfsprogs`，不要假定只装 ntfs-3g 就一定有它。

```sh
sudo ntfsfix --no-action "$data_device"
```

保留输出和退出码。`--no-action` 不写盘，其输出即使描述修复动作也不表示已经修复。

| 情况 | 下一步 |
| --- | --- |
| 重要数据无备份、严重元数据损坏、工具失败或重复出现不一致 | 停止写修复，先做镜像与恢复评估 |
| 有 Windows/恢复环境可用 | 核对卷容量、标签和实际盘符后，在管理员终端运行 `chkdsk X: /f`，完成后正常关机；X 不是历史盘符 |
| 暂无 Windows 环境、已备份且接受应急恢复的边界 | 可由用户授权后采用下述 ntfsfix 分支 |

`chkdsk /f` 需要锁定卷；在恢复环境中盘符可能变化。不默认使用额外扫描坏扇区的 `/r`。
机器没有已安装 Windows 不妨碍另行准备可运行 chkdsk 的恢复环境，但本流程不自动创建它。

### 4. Linux 应急分支（写盘，必须单独确认）

仅在目标核对、备份、卸载均完成后执行：

```sh
sudo ntfsfix "$data_device"
```

若失败或仍有未解决不一致，停止。若基本修复成功，但后续挂载仅因 dirty 被拒绝，
且已明确接受尚未完整检查的风险，可以在确认仍未挂载后执行：

```sh
sudo ntfsfix -d "$data_device"
```

这不是通用第二步：若已通过 chkdsk、已正常挂载或有新错误，不重复清标记。
清除标记后仍失败，收集新日志并回到诊断分支，不循环修复、不添加 force。

### 5. 验收与后续闭环

```sh
sudo systemctl start "$mount_unit"
findmnt --mountpoint "$data_mount" -o SOURCE,FSTYPE,OPTIONS
systemctl show "$mount_unit" -p ActiveState -p Result
journalctl -b -u "$mount_unit" --no-pager -n 20
```

验收不止查看目录存在：

- 来源必须对应已核对 UUID，文件系统为预期类型、挂载选项符合预期。
- 检查重要文件可读，与已有备份/校验和比对；对每个重要仓库检查 `git status`、
  `git fsck --full`。Git 检查不覆盖未提交文件内容、LFS 实体或普通文档，需另行验证。
- 在已完成备份后，可创建独立临时文件验证写入、同步和读回，仅清理该测试文件。
- 优先恢复数据备份，再按需恢复受影响的应用入口；不为刷新目录而重启整个工作会话。
- 维护窗口内正常卸载/重挂或正常重启后复验。再次 dirty 应升级调查，检查关机日志、
  SMART 增量、供电与内核版本，不能把重复清标记当维护手段。

关闭事件时记录：备份位置（留在本机私人记录）、修复输出、文件校验范围、重启验收结果，
以及完整检查/迁移是否仍待办。EFI 的 FAT 告警另行安排卸载后的检查，不能用 ntfsfix 处理。

## 资料

- [Linux 内核 NTFS3 文档](https://www.kernel.org/doc/html/latest/filesystems/ntfs3.html)：日志重放、force 的边界。
- [ntfsfix 手册（Arch 随包发布）](https://man.archlinux.org/man/ntfsfix.8.en)：有限修复、dirty 与 no-action 选项。
- [Microsoft chkdsk 文档](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/chkdsk)：完整检查入口、卷锁定与 /f、/r 区别。
