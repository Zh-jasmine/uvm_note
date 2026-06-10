# Synopsys EDA Tools (VCS + Verdi + SCL) Installation Guide for CentOS 7

> **Audience:** AI agent / LLM. This guide is structured for automated execution.
> **Target:** CentOS 7 (any arch) with VMware Workstation.
> **Packages:** VCS O-2018.09-SP2, Verdi O-2018.09-SP2, SCL 2018.06, Installer v5.0.

---

## 来源说明

本教程参考了 CSDN 博主 **芯王国** 的内容，EDA 软件安装包也由其提供。

- 参考文章：[Synopsys EDA Tools 安装指南](https://blog.csdn.net/weixin_40377195/article/details/109301426?spm=1001.2014.3001.5502)

---

## 使用说明

本文档设计为**直接交给接入了大模型的 AI Agent 执行**，无需人工逐步操作。

使用步骤：

1. 将本文档完整提供给 AI Agent（如 Claude Code、Cursor 等已接入大模型的工具）
2. Agent 会**自行完成 SSH 接入虚拟机**——告诉 Agent 虚拟机的 IP、用户名和密码，Agent 会通过 SSH 连接并执行本文档中的所有安装步骤
3. Agent 按照文档 Phase 1 → Phase 6 的顺序在虚拟机中执行安装
4. 如果 Agent 询问 Windows 端的操作（如 keygen 生成 license），配合完成即可

**前提条件：**
- 虚拟机已安装 CentOS 7，网络可达
- 安装包已传输到虚拟机或可通过共享文件夹访问
- 虚拟机已开启 SSH 服务（`systemctl start sshd`）

> Agent 接入 SSH 后会自行完成所有 Linux 端操作，包括依赖安装、软件安装、license 配置和功能验证。

---

## Phase 1: Host Preparation

### 1.1 Install OS dependencies

```bash
yum install -y gcc gcc-c++ libXext libXft libXrender libXi libXt libXtst \
  libstdc++ ksh tcsh libXScrnSaver libpng12 redhat-lsb
```

### 1.2 Create install directory

```bash
mkdir -p /home/synopsys
chmod 777 /home/synopsys
```

### 1.3 Transfer packages to VM

Source files on Windows host (example). Transfer via SCP or shared folder:

```
synopsysinstaller_v5.0/
scl_v2018.06/
vcs_vO-2018.09-SP2/
vcs_mx_vO-2018.09-SP2/
verdi-2018.9/
scl_keygen_2030.zip       # Windows-only, keep on host
```

SCP command from Windows:

```bash
scp -r /path/to/synopsysinstaller_v5.0 user@vm_ip:/home/zh/
# repeat for each package
```

Alternatively, use VMware shared folder at `/mnt/hgfs/`.

---

## Phase 2: Install Tools

### 2.1 Run Synopsys Installer

**Option A: GUI (recommended for reliability)**

```bash
# First install the installer itself
cd /path/to/synopsysinstaller_v5.0
./setup.sh
```
> In the GUI: Click Next → Add all 4 package directories (SCL, VCS, VCS-MX, Verdi) → Set destination to `/home/synopsys` → Install all.

**Option B: Batch mode** (works but may fail silently on some packages)

```bash
cd /path/to/synopsysinstaller_v5.0
./setup.sh -batch_installer
```
When prompted, enter:
1. Source paths (comma or space separated): `/home/zh/scl_v2018.06,/home/zh/vcs_vO-2018.09-SP2,/home/zh/vcs_mx_vO-2018.09-SP2,/home/zh/verdi-2018.9`
2. Install path: `/home/synopsys`
3. Confirm Y

### 2.2 Expected result after install

```bash
ls /home/synopsys/
# Should show: scl/  vcs/  vcs-mx/  verdi/
```

### 2.3 Set up architecture symlinks (if missing)

```bash
# VCS needs linux -> linux64 symlink
if [ ! -d /home/synopsys/vcs/O-2018.09-SP2/linux ]; then
  ln -s linux64 /home/synopsys/vcs/O-2018.09-SP2/linux
fi
# VCS-MX
if [ ! -d /home/synopsys/vcs-mx/O-2018.09-SP2/linux ]; then
  ln -s linux64 /home/synopsys/vcs-mx/O-2018.09-SP2/linux
fi
```

---

## Phase 3: License Setup

### 3.1 Generate license file (Windows required)

`scl_keygen_2030.zip` must be extracted and run on Windows.

**Keygen parameters:**
- Host ID (Daemon): VM's MAC address (hex, no colons)
- Host ID (Feature): same as above
- Port: 27000 (default)
- Vendor: snpslmd

Click **Generate**, produces `Synopsys.dat`.

### 3.2 Transfer license to VM

```bash
scp Synopsys.dat user@vm_ip:/home/synopsys/
```

### 3.3 Fix license file

Convert Windows line endings and fix the DAEMON path:

```bash
sed -i 's/\r$//' /home/synopsys/Synopsys.dat
# The second line must have full path to snpslmd:
# DAEMON snpslmd /home/synopsys/scl/2018.06/linux64/bin/snpslmd
# If it only has "DAEMON snpslmd", fix it:
sed -i 's|^DAEMON snpslmd$|DAEMON snpslmd /home/synopsys/scl/2018.06/linux64/bin/snpslmd|' /home/synopsys/Synopsys.dat
```

### 3.4 Create ld-lsb symlink (SCL requirement on CentOS 7)

```bash
ln -s /lib64/ld-linux-x86-64.so.2 /lib64/ld-lsb-x86-64.so.3
```

### 3.5 Add hostname to /etc/hosts (lmgrd hostname resolution)

```bash
HOSTNAME=$(hostname)
IP=$(ip addr show ens33 | grep 'inet ' | awk '{print $2}' | cut -d/ -f1)
if ! grep -q "$HOSTNAME" /etc/hosts; then
  echo "$IP $HOSTNAME" >> /etc/hosts
fi
```

### 3.6 Disable SELinux (lmgrd port binding)

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

### 3.7 Start license server

```bash
/home/synopsys/scl/2018.06/linux64/bin/lmgrd \
  -c /home/synopsys/Synopsys.dat \
  -l /home/synopsys/synopsys.log
```

Verify:

```bash
/home/synopsys/scl/2018.06/linux64/bin/lmstat -a -c /home/synopsys/Synopsys.dat
# Should show: "license server UP"
```

---

## Phase 4: Environment Configuration

Add to user's `~/.bashrc`:

```bash
# Synopsys EDA Tools Environment Variables
export SYNOPSYS_HOME=/home/synopsys
export VCS_HOME=$SYNOPSYS_HOME/vcs/O-2018.09-SP2
export VERDI_HOME=$SYNOPSYS_HOME/verdi/Verdi_O-2018.09-SP2
export SCL_HOME=$SYNOPSYS_HOME/scl/2018.06

# PATH
export PATH=$VCS_HOME/bin:$VERDI_HOME/bin:$SCL_HOME/linux64/bin:$PATH

# License
export LM_LICENSE_FILE=27001@ic1
export SNPSLMD_LICENSE_FILE=27001@ic1

# Alias for starting license server
alias lmg_synopsys='lmgrd -c /home/synopsys/Synopsys.dat -l /home/synopsys/synopsys.log'
```

Note: `ic1` should match the VM's hostname. Replace if different.

```bash
source ~/.bashrc
```

---

## Phase 5: VCS Verification

```bash
mkdir -p ~/test_verdi && cd ~/test_verdi
cat > test.sv << 'EOF'
module test;
  logic clk;
  logic [7:0] cnt;
  initial clk = 0;
  always #5 clk = ~clk;
  always_ff @(posedge clk) cnt <= cnt + 1;
  initial begin
    $fsdbDumpfile("test.fsdb");
    $fsdbDumpvars(0, test);
    #110;
    $finish;
  end
endmodule
EOF

vcs -full64 -sverilog -debug_access+all test.sv
./simv
# Expected: "V C S Simulation Report" + test.fsdb generated
```

---

## Phase 6: Verdi Verification

From **desktop terminal** (SSH cannot launch GUI):

```bash
cd ~/test_verdi
verdi -f filelist.f -ssf test.fsdb -top test &
```

Expected: Verdi GUI window opens with waveform.

If Verdi fails to open or crashes:
1. Check license: `lmstat -a`
2. Check DISPLAY: `echo $DISPLAY` (must not be empty in desktop terminal)
3. NO extra Qt fixes needed -- `setup.sh` installs correctly; earlier issues were caused by manual file copy bypassing setup.sh

---

## Error Recovery Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| `lmgrd: command not found` | SCL not in PATH | Use full path: `/home/synopsys/scl/2018.06/linux64/bin/lmgrd` |
| `Cannot find libpng12.so.0` | Missing dep | `yum install libpng12` |
| `Cannot find lsb` | Missing dep | `yum install redhat-lsb` |
| lmgrd binds to port but lmstat shows no daemon | SELinux blocking | `setenforce 0` (see Phase 3.6) |
| lmgrd can't resolve hostname | Hostname not in /etc/hosts | Add to /etc/hosts (see Phase 3.5) |
| `error while loading shared libraries: ld-lsb-x86-64.so.3` | Missing symlink | See Phase 3.4 |
| VCS fails to link | Wrong arch symlink | See Phase 2.3 |
| Verdi "Could not checkout license" | Wrong LM_LICENSE_FILE hostname | Use hostname from `hostname` command, not `localhost.localdomain` |
| Verdi segfault on startup | Qt library mismatch | **Unlikely if installed via setup.sh.** If happens: verify setup.sh was used (not manual file copy). Reinstall if needed. |

---

## Key Lessons (for AI agents)

1. **Always use `setup.sh`.** Manual file copy breaks Verdi's internal library paths. The GUI install is reliable.
2. **Batch mode exists but isn't fully documented.** `./setup.sh -batch_installer` works but prompts are sparse. GUI is safer.
3. **SCL license server needs 3 things:** ld-lsb symlink, hostname in /etc/hosts, SELinux permissive.
4. **Two license env vars needed:** `LM_LICENSE_FILE` and `SNPSLMD_LICENSE_FILE` (both pointing to `port@hostname`).
5. **Verdi GUI cannot be tested via SSH** -- requires desktop terminal with `DISPLAY=:0`.
6. **Qt4 is NOT a real issue** if installed correctly via setup.sh. Previous Qt-related efforts were red herrings caused by skipping setup.sh.
