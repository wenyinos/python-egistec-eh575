# EH575 指纹驱动部署实施与回滚方案

> 适用机型：Acer Swift 3x（主机名 swift3x.ha）
> 驱动来源：[SandroRoque/python-egistec-eh575](https://github.com/SandroRoque/python-egistec-eh575)（fork 自 AbbhinavJayaraman/python-egistec-eh575）
> 部署日期：2026-08-24
> 部署方式：`install-stable.sh`（事务式，失败自动回滚）

---

## 一、背景

本机指纹模块 EgisTec EH575（USB ID `1c7a:0575`）不在 libfprint 官方支持列表中
（官方 EgisTec 支持：0570/0571/0582/0583/0584/0586/0587/0588/05a1/05ae/0603/9201），
系统 fprintd 无法枚举设备。本方案通过 python-egistec-eh575 的 D-Bus 桥接驱动
（open-fprintd + egis-bridge）提供指纹功能，替代系统 fprintd 的 `net.reactivated.Fprint`
D-Bus 服务。

## 二、架构

```
fprint 客户端 (fprintd-enroll / fprintd-verify / GDM 锁屏)
        │ D-Bus 系统总线 net.reactivated.Fprint
        ▼
open-fprintd（Manager，注册 net.reactivated.Fprint）
        ▲  RegisterDevice / io.github.uunicorn.Fprint.Device.Egis
        │
egis-bridge（USB 设备桥接，持有 1c7a:0575）
        │
        ▼
EH575 传感器（滑动式，图像 103×52）
```

- **open-fprintd**：D-Bus 管理器，提供标准 fprintd Manager 接口（GetDefaultDevice 等），
  与系统 fprintd 客户端完全兼容。
- **egis-bridge**：USB 驱动层，初始化传感器（自检 47 条命令全成功），
  向 manager 注册设备路径 `/net/reactivated/Fprint/Device/0`。
- 模板存储：`/var/lib/open-fprintd/egis`；校准数据：`/var/lib/open-fprintd/egis-calibration`。

## 三、实施步骤（已执行，2026-08-24）

### 1. 安装 Python 依赖（4 个包，实际 116 个依赖）

```bash
sudo dnf install -y python3-opencv python3-numpy python3-scikit-image python3-pyusb
```

### 2. 停用系统 fprintd（避免 D-Bus 名字竞争）

```bash
sudo systemctl stop fprintd
```

> 注：fprintd.service 为 static 单元（不可 disable），且为 D-Bus activatable；
> open-fprintd 开机自启（multi-user.target）会先占用 `net.reactivated.Fprint` 名字。

### 3. 运行安装脚本

```bash
cd ~/MyDev/python-egistec-eh575
./install-stable.sh --check      # 只读预检
sudo ./install-stable.sh         # 安装（自动备份 + 验证 + 失败回滚）
```

### 4. 安装产物清单

| 路径 | 内容 |
|---|---|
| `/opt/egis-driver/` | 驱动主体（open-fprintd、egis-bridge、egis-calibrate、egis-doctor、openfprintd/、egis_driver/） |
| `/etc/systemd/system/open-fprintd.service` | Manager 服务单元 |
| `/etc/systemd/system/egis-bridge.service` | 桥接服务单元 |
| `/etc/systemd/system/multi-user.target.wants/{open-fprintd,egis-bridge}.service` | 开机自启符号链接 |
| `/etc/udev/rules.d/70-egis-eh575.rules` | USB 设备权限规则（MODE=0660 GROUP=input） |
| `/usr/share/dbus-1/system.d/io.github.uunicorn.Fprint.Device.Egis.conf` | D-Bus 名字授权策略 |
| `/usr/share/polkit-1/actions/net.reactivated.fprint.policy` | 新增文件（**与官方 `net.reactivated.fprint.device.policy` 独立共存，未覆盖**；含 register action；与官方文件重叠定义 enroll/verify action id，按文件名排序本项目文件后加载生效） |
| `/var/lib/open-fprintd/{egis,egis-calibration}/` | 登记模板与校准数据（0700） |

### 5. 验证结果

```bash
systemctl is-active open-fprintd egis-bridge   # → active active
busctl --system call net.reactivated.Fprint \
  /net/reactivated/Fprint/Manager \
  net.reactivated.Fprint.Manager GetDefaultDevice
# → o "/net/reactivated/Fprint/Device/0"
sudo /opt/egis-driver/egis-doctor
# → EH575 compatibility: PASS / USB: compatible / Services: ready
fprintd-list "$USER"
# → found 1 devices / Device at /net/reactivated/Fprint/Device/0
```

## 四、部署中遇到的问题（已解决）

### 问题 1：首次安装验证失败 → 自动回滚

- **现象**：安装后 20 秒内 D-Bus 设备未就绪，脚本自动回滚。
- **根因**：egis-bridge 启动时请求 D-Bus 名
  `io.github.uunicorn.Fprint.Device.Egis` 被策略拒绝（AccessDenied），
  以及 RegisterDevice 的 polkit 检查失败（策略文件中 `net.reactivated.fprint.manager.register`
  action 未注册）——D-Bus/polkit 策略文件加载时序问题。
- **处理**：手动装回 `io.github.uunicorn.Fprint.Device.Egis.conf` 与
  `net.reactivated.fprint.policy` 后重跑安装脚本，成功。
- **经验**：若安装失败回滚后，应先确认 `/usr/share/dbus-1/system.d/` 与
  `/usr/share/polkit-1/actions/` 下策略文件完整，再重试。

### 问题 2：安装脚本两个小缺陷（不影响部署）

1. **回滚不清理 enable 符号链接**：回滚后 `/etc/systemd/system/multi-user.target.wants/`
   下的两个 service 链接残留，指向不存在的单元，会导致 systemd 启动报错。
   **若再遇回滚，需手动清理**：
   ```bash
   sudo rm -f /etc/systemd/system/multi-user.target.wants/open-fprintd.service \
              /etc/systemd/system/multi-user.target.wants/egis-bridge.service
   sudo systemctl daemon-reload
   ```
2. **退出码污染**：首次安装（无旧备份目录）时脚本以退出码 1 结束，
   因为末尾 `[ -d "$BACKUP_DIR" ]` 测试返回 1。功能正常，仅退出码不准。

## 五、回滚方案（完整卸载，恢复到部署前状态）

> 前提：以 root 执行；模板/校准数据如需保留先备份 `/var/lib/open-fprintd`。

### 步骤 1：停止并禁用服务

```bash
sudo systemctl stop open-fprintd egis-bridge
sudo systemctl disable open-fprintd egis-bridge
```

### 步骤 2：删除服务单元与自启链接

```bash
sudo rm -f /etc/systemd/system/open-fprintd.service \
           /etc/systemd/system/egis-bridge.service \
           /etc/systemd/system/multi-user.target.wants/open-fprintd.service \
           /etc/systemd/system/multi-user.target.wants/egis-bridge.service
sudo systemctl daemon-reload
```

### 步骤 3：删除 udev 规则

```bash
sudo rm -f /etc/udev/rules.d/70-egis-eh575.rules
sudo udevadm control --reload-rules && sudo udevadm trigger --subsystem-match=usb
```

### 步骤 4：删除 D-Bus 策略

```bash
sudo rm -f /usr/share/dbus-1/system.d/io.github.uunicorn.Fprint.Device.Egis.conf
```

### 步骤 5：删除项目 polkit 策略（官方文件无需恢复）

项目文件 `net.reactivated.fprint.policy` 与官方 `net.reactivated.fprint.device.policy`
**文件名不同、独立共存，未覆盖任何官方文件**（`rpm -V fprintd` 已验证官方包完好）。
直接删除项目文件即可：

```bash
sudo rm -f /usr/share/polkit-1/actions/net.reactivated.fprint.policy
```

### 步骤 6：删除驱动目录与数据

```bash
sudo rm -rf /opt/egis-driver            # 驱动主体
sudo rm -rf /var/lib/open-fprintd       # 模板与校准数据（确认无需保留时）
```

> 若安装脚本留有备份目录（`/opt/egis-driver.backup.*`），删除驱动前可确认其内容。

### 步骤 7：恢复系统 fprintd 行为

```bash
sudo systemctl start fprintd    # 系统 fprintd 按需激活（D-Bus activatable）
```

### 步骤 8：终止手动调试残留进程（如有）

部署/调试期间若以 `--debug` 方式手动运行过 `open-fprintd`/`egis-bridge`
（进程挂在用户会话 systemd 下，非服务单元管理，`systemctl stop` 无法终止），
即使服务已卸载，进程仍会持有 `net.reactivated.Fprint` D-Bus 名字，导致
`fprintd-list` 仍报告设备（`Client must claim device first`）。需手动清理：

```bash
ps aux | grep -E 'bin/open-fprintd|bin/egis-bridge' | grep -v grep
sudo kill <PID>
```

> 经验：回滚验证若发现 `net.reactivated.Fprint` 的持有单元是 `session-*.scope`
> 而非 service 单元，即为手动残留进程，按本步骤清理。

### 步骤 9：卸载 Python 依赖（可选）

仅当确认这些包无其他用途：

```bash
sudo dnf remove -y python3-opencv python3-numpy python3-scikit-image python3-pyusb
```

> 注意：这些是通用科学计算包，系统其他程序（如 image 处理脚本）可能依赖，
> 建议保留或先 `dnf remove --setopt=clean_requirements_on_remove=1` 评估影响。

### 回滚验证清单

```bash
systemctl status open-fprintd egis-bridge   # → not-found / inactive
busctl --system list | grep -i fprint       # → 仅 net.reactivated.Fprint (activatable)
fprintd-list "$USER"                        # → No devices available（官方驱动不支持该设备，预期行为）
rpm -V fprintd                              # → 无输出（polkit 文件已恢复）
ls /opt/egis-driver                         # → 不存在
```

## 六、风险与注意事项

1. **实验性驱动**：驱动声称仅验证于部分 ASUS/Lenovo 机型；本机为 Acer Swift 3x，
   USB 参数与 known-good 基线一致（bcdDevice 1072、端点 01/82、512B 包）。
2. **小图匹配限制**：EH575 为滑动式传感器（103×52 像素），本驱动用 SIFT/FLANN
   特征匹配绕开 libfprint 小图缺陷；若验证失败率高，需执行校准流程。
3. **验证 fail-closed**：未校准前验证一律失败（日志：`verification will fail closed`），
   必须完成 `egis-calibrate` 校准（genuine 样本 + impostor 样本 + 阈值分析）后验证才生效。
4. **fprintd-delete 可能失败**：本项目 polkit 文件无 `net.reactivated.fprint.device.delete`
   action（官方 fprintd 有），`fprintd-delete` 可能报 NoSuchAction。
5. **与系统 fprintd 的竞争**：若系统 fprintd 被 D-Bus 激活先于 open-fprintd 注册
   `net.reactivated.Fprint`，open-fprintd 注册会失败；当前 open-fprintd 开机自启可避免。
6. **系统升级影响（重要）**：本项目全部文件均非 rpm 包文件（服务单元、udev 规则、
   dbus 策略、polkit 策略均独立于 fprintd/libfprint 包），**常规 `dnf update` 不会覆盖**，
   升级后无需重新部署；跨版本升级（system-upgrade）后建议跑一次 `egis-doctor` 验证。
   万一出现 bridge 注册失败（如 Python 大版本导致依赖不兼容），重跑
   `sudo ./install-stable.sh` 即可，已登记指纹数据保留。
7. **密码认证不受影响**：回滚前/后 GDM 密码登录始终可用。

## 七、日常维护

```bash
# 查看服务
systemctl status open-fprintd egis-bridge

# 查看驱动日志
journalctl -u open-fprintd -u egis-bridge -n 50

# 诊断
sudo /opt/egis-driver/egis-doctor --json --output doctor.json

# 更新驱动（源码更新后）
cd ~/MyDev/python-egistec-eh575
git pull && ./tools/check && sudo ./install-stable.sh

# 重新登记（模板 schema 变更后需重录）
fprintd-enroll -f right-index-finger
```
