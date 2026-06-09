# Verdi 常用操作（tmp）

## 启动

```bash
# 有 -kdb 编译（带 Instance 树）
cd /tmp/AXI-LITE2SPI/UVMTB
verdi -dbdir simv.daidir -ssf tb_top.fsdb &

# 只有波形（无 Instance 树）
verdi -ssf tb_top.fsdb &
```

## 窗口

| 操作 | 菜单 | 快捷键 |
|------|------|--------|
| 打开 Instance 树 | View → Instance | Ctrl+1 |
| 打开信号/波形 | View → Waveform | Ctrl+4 |
| 打开源码 | View → Source Code | Ctrl+5 |
| 打开 Schematic | View → Schematic | Ctrl+3 |

## 波形操作

| 操作 | 说明 | 快捷键 |
|------|------|--------|
| 放大 | 水平放大波形 | `i` |
| 缩小 | 水平缩小波形 | `o` |
| 缩放到全屏 | 显示全部波形 | `f` |
| 放大选中区域 | 拖选区域后按 | `z` |
| 添加Marker | 添加时间标记 | `m` |
| 跳转到下一个跳变沿 | 移到下一个值变化 | `n` |
| 跳转到上一个跳变沿 | 移到上一个值变化 | `p` |

## 信号操作

| 操作 | 说明 |
|------|------|
| 把信号拖到波形 | 从 Instance/Source 窗口选中信号拖到 Waveform 窗口 |
| 添加信号到波形 | 选中信号 → 右键 → Add to Waveform |
| 查看信号值 | 鼠标悬停在信号上 |
| 信号搜索 | Waveform 窗口按 `/` 输入信号名 |
| 改变信号显示格式 | 选中信号 → 右键 → Radix → Hex/Dec/Bin/ASCII |
| 分组信号 | 选中多个信号 → 右键 → Group |
| 取消分组 | 右键组名 → Ungroup |

## 源码窗口

| 操作 | 说明 |
|------|------|
| 跳转到信号定义 | 双击信号（Source/Instance 窗口） |
| 查找 | Ctrl+F |
| 跳转到行 | Ctrl+G |
| 增大字体 | Ctrl+Shift+. |
| 减小字体 | Ctrl+Shift+, |

## 常用调试流程

1. 打开波形，看信号整体变化（`f` 缩放全屏）
2. 找到感兴趣的时间区域，放大（拖选后 `z`）
3. 点信号，按 `n`/`p` 跳变沿
4. 从 Instance 树找信号拖到波形
5. 双击信号跳回源码定位

## 保存/恢复波形设置

```
# 保存 .rc 文件：File → Save Session
# 下次打开：File → Restore Session
```

## 其他

| 操作 | 说明 |
|------|------|
| `nWave` → `Signal` → `Value Radix` | 批量改信号显示格式 |
| `Tools` → `Preferences` → `Font` | 改字体大小 |
| `Tools` → `Preferences` → `Color` | 改波形颜色 |
| `Tools` → `Waveform Calculate` | 做简单的计算 |

---

*2026-05-28 临时笔记，待整理进正式文档*
