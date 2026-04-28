# Proxmox VE 8.4.0显卡直通完整指南：NVIDIA Tesla T4 实战

## 前言
PCIe Passthrough 技术允许虚拟机直接访问物理GPU 设备，绕过宿主机系统，从而获得接近原生性能的图形处理能力。

目前我已在服务器完成了proxmox8.4.0的安装，并且安装了带有NVIDIA Tesla T4的显卡。现在我需要将显卡直接直通到一台vm实例上面。

## 1. 确认硬件和BIOS支持

### 检查显卡兼容性
Tesla T4是数据中心显卡，完全支持PCIe直通，非常适合虚拟化环境。

### 启用IOMMU
进入服务器BIOS，启用Intel VT-d（如果是Intel CPU）或AMD-Vi（如果是AMD CPU）。

### 验证系统信息
```bash
# 输出确认当前proxmox server 是基于debian 12 版本的 8.4.0 proxmox 操作系统
cat /etc/os-release 
pveversion -v
````

<img width="725" height="404" alt="Image" src="https://github.com/user-attachments/assets/f5524463-0794-436f-bdc6-f26b5f405b98" />

````bash
# 显卡信息
lspci -nnk|grep "NVIDIA"
````

<img width="999" height="74" alt="Image" src="https://github.com/user-attachments/assets/1733103d-3ca8-48b0-bdbc-ef63076d54da" />

## 2. 配置Proxmox主机

### 修改GRUB参数，检查和启用IOMMU支持
```bash
# 备份原始配置
cp /etc/default/grub{,.bak} 

# 编辑GRUB配置
vi /etc/default/grub
```
在`GRUB_CMDLINE_LINUX_DEFAULT`中添加（使用默认，pve版本为8.4.0）：
```bash
# 修改 GRUB_CMDLINE_LINUX_DEFAULT 配置为
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt initcall_blacklist=sysfb_init pcie_acs_override=downstream"

# 注意：pve  7.2 以前版本使用
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt video=efifb:off,vesafb:off pcie_acs_override=downstream"
````

<img width="1315" height="255" alt="Image" src="https://github.com/user-attachments/assets/b6b1fb7a-8b9c-4920-9fd1-f724fb595c6a" />

**说明**：
- `video=efifb:off`用于禁用主机对GPU的显示输出，确保虚拟机独占GPU。
- `intel_iommu=on` 启用Intel平台的IOMMU支持。
- `iommu=pt`仅对直通设备启用IOMMU，减少性能开销。
- `initcall_blacklist=sysfb_init`防止宿主机占用显卡帧缓冲区。
- `pcie_acs_override=downstream`解决某些PCIe设备的ACS限制问题。

**注意**：
对于AMD平台，需将`intel_iommu=on`替换为`amd_iommu=on`
更新GRUB：
```bash
update-grub
```

<img width="631" height="165" alt="Image" src="https://github.com/user-attachments/assets/7c8e2a62-270d-4e74-8237-99050416b2c7" />

### 加载VFIO模块
```bash
vi /etc/modules
```
添加以下内容：
```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

<img width="934" height="171" alt="Image" src="https://github.com/user-attachments/assets/d8aad77a-fe7c-4d4c-a5ca-46d441370596" />

reboot重启proxmox主机


### 验证IOMMU是否启用：
IOMMU（Input-Output Memory Management Unit）是硬件辅助的虚拟化技术，为PCI设备直通提供必要的内存管理和隔离功能。
```bash
dmesg | grep -E "DMAR|IOMMU"
```

<img width="1164" height="778" alt="Image" src="https://github.com/user-attachments/assets/88a68188-3296-4475-8511-324f02f69c83" />

### 验证并配置VFIO绑定Tesla T4
验证VFIO模块：
```bash
dmesg | grep -i vfio
```

<img width="675" height="52" alt="Image" src="https://github.com/user-attachments/assets/cbf3f04a-9ce4-442b-b4c0-0df8c150af4f" />

```bash
vi /etc/modprobe.d/vfio.conf
```
添加（使用你的显卡PCI ID）：
```
options vfio-pci ids=10de:1eb8 disable_vga=1
```

**获取ID方法**：
```bash
lspci -nn | grep -i tesla

输出示例：
c3:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
```

<img width="641" height="76" alt="Image" src="https://github.com/user-attachments/assets/f6d25a07-62b9-4f03-ba6e-999aa3ea7ec7" />

验证是否支持，中断重映射：
```bash
dmesg | grep 'remapping'
```

<img width="1098" height="98" alt="Image" src="https://github.com/user-attachments/assets/0eb9bf78-1774-4c79-8ecb-02f151c7dafe" />

屏蔽显卡驱动：
在proxmox主机屏蔽掉显卡的驱动：
```bash
echo "# NVIDIA" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist nvidiafb" >> /etc/modprobe.d/blacklist.conf 
echo "blacklist nvidia_drm" >> /etc/modprobe.d/blacklist.conf 
echo "" >> /etc/modprobe.d/blacklist.conf
```

**其他有用的配置**：
```bash
# 允许不安全的中断
echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf 

# 为 NVIDIA 卡添加稳定性修复和优化
echo "options kvm ignore_msrs=1 report_ignored_msrs=0" > /etc/modprobe.d/kvm.conf
```

### 更新initramfs并重启
```bash
# 更新内核引导文件并重启宿主机：
update-initramfs -k all -u 
reboot
```

## 3. 验证IOMMU配置
重启后检查IOMMU组：
```bash
for iommu_group in $(find /sys/kernel/iommu_groups/ -maxdepth 1 -mindepth 1 -type d); do 
    echo "IOMMU Group $(basename "$iommu_group"):" 
    lspci -nns "$(cat "$iommu_group/devices/*/uevent" | grep -oP 'PCI_SLOT_NAME=\K.*')" 
done | grep -i tesla
```
确保Tesla T4的所有设备（3D控制器和音频设备）在同一IOMMU组中。

## 4. VM实例添加显卡

### 添加Tesla T4显卡
这里使用了登录web控制台增加pci设备的方式，当然也可以shell登录宿主机使用命令添加的方式！

以VM实例104为例：点击vm实例-硬件-添加-PCI设备-原始设备，找到tesla设备选中，添加：

<img width="1764" height="873" alt="Image" src="https://github.com/user-attachments/assets/ce48789c-2a0b-4a82-a8ac-116e83f36edd" />

### 验证vm实例挂载显卡成功
登录vm实例，使用如下命令确认vm实例成功挂载了显卡设备：
```bash
lspci | grep -i vga 
lspci -nn | grep NVIDIA
```

<img width="726" height="106" alt="Image" src="https://github.com/user-attachments/assets/c207a9db-5065-44ef-9bf2-f8afb38bd081" />

ok到这里 显卡就完成了 显卡直通的相关操作！

## 5. 安装Ubuntu驱动

### 安装NVIDIA驱动
```bash
# 添加NVIDIA仓库
sudo apt update 
sudo apt install software-properties-common 
sudo add-apt-repository ppa:graphics-drivers/ppa 
sudo apt update 

# 安装CUDA Toolkit（包含驱动）
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin 
sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600 
wget https://developer.download.nvidia.com/compute/cuda/12.1.1/local_installers/cuda-repo-ubuntu2204-12-1-local_12.1.1-530.30.02-1_amd64.deb 
sudo dpkg -i cuda-repo-ubuntu2204-12-1-local_12.1.1-530.30.02-1_amd64.deb 
sudo cp /var/cuda-repo-ubuntu2204-12-1-local/cuda-*-keyring.gpg /usr/share/keyrings/ 
sudo apt update 
sudo apt install cuda-drivers 

# 重启虚拟机
sudo reboot
```

### 验证GPU识别
```bash
nvidia-smi
```
输出应显示Tesla T4信息：
```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.148.08             Driver Version: 570.148.08     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla T4                       Off |   00000000:01:00.0 Off |                    0 |
| N/A   57C    P0             26W /   70W |     209MiB /  15360MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|    0   N/A  N/A          169410      C   ./bin/wf_ai_server                      206MiB |
+-----------------------------------------------------------------------------------------+
```

## 6. 常见问题解决

### 虚拟机无法启动
检查Proxmox日志：
```bash
journalctl -xe | grep qemu
```
尝试取消"主GPU"勾选。

### 驱动安装失败
确保禁用了Ubuntu的nouveau驱动：
```bash
sudo bash -c "echo 'blacklist nouveau' > /etc/modprobe.d/blacklist-nvidia-nouveau.conf"
sudo bash -c "echo 'options nouveau modeset=0' >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf" 
sudo update-initramfs -u
```

### NVIDIA-SMI显示错误
检查内核模块是否正确加载：
```bash
lsmod | grep nvidia
```
重新安装驱动：
```bash
sudo apt purge cuda-drivers && sudo apt install cuda-drivers
```

## 优化建议
- **启用MIG（多实例GPU）**：Tesla T4支持将GPU分割为多个独立实例，可通过`nvidia-smi mig`命令配置。
- **安装cuDNN**：用于深度学习加速，需从NVIDIA官网下载并安装。
- **监控GPU使用**：在虚拟机内使用`nvidia-smi`或`nvtop`监控GPU状态。

完成上述步骤后，Ubuntu虚拟机将能够完全利用Tesla T4的计算能力，适用于AI推理、高性能计算等工作负载。