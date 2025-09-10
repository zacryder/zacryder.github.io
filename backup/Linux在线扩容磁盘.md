**场景1：LVM逻辑卷管理（推荐）**
1.**检查当前分区与卷组**
启动虚拟机后，执行以下命令查看磁盘布局：
````
lsblk
sudo vgdisplay  # 确认卷组名称（如ubuntu-vg）
sudo lvs        # 查看逻辑卷信息
````
2.**创建新分区并加入LVM**

* 使用`fdisk`或`parted`创建新分区（假设新分区为`/dev/sda4`）：

  ```bash
  sudo fdisk /dev/sda# 
  输入以下指令：
  n  # 新建分区
  p  # 主分区
  4  # 分区号
  [直接回车使用默认起始/结束扇区]  # 占用全部未分配空间
  t  # 更改分区类型
  4  # 选择分区号
  8e # LVM类型
  w  # 保存并退出
  ```
* 更新内核分区表：

  ```bash
  sudo partprobe /dev/sda
  ```
3.**扩展物理卷、卷组与逻辑卷**

```bash
sudo pvcreate /dev/sda4                 # 创建物理卷
sudo vgextend ubuntu-vg /dev/sda4       # 扩展卷组
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv  # 扩展逻辑卷
```
4. **调整文件系统大小**

    * 若为`ext4`文件系统：

      ```bash
      sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
      ```
    * 若为`XFS`文件系统：

      ```bash
      sudo xfs_growfs /
      ```
5. **验证扩容结果**

    ```bash
    df -h | grep " /$"  # 确认根分区容量已更新
    ```
#### **场景2：非LVM分区（直接扩展）**

1. **使用**​**​`parted`​**​**调整分区大小**

    ```bash
    sudo parted /dev/sda(parted) resizepart 3 100%  # 假设需扩展的分区号为3(parted) quit
    ```
2. **调整文件系统大小**

    * 对于`ext4`：

      ```bash
      sudo resize2fs /dev/sda3
      ```
    * 对于`XFS`：

      ```bash
      sudo xfs_growfs /dev/sda3
      ```
### **三、自动化脚本（进阶用户）**

以下脚本可简化LVM环境下的扩容流程（需根据实际环境调整参数）：

```bash
#!/bin/bash
# 扩容LVM逻辑卷的脚本
read -p "请输入新分区设备路径（如/dev/sda4）: " NEW_PARTITION
read -p "请输入卷组名称（如ubuntu-vg）: " VG_NAME
read -p "请输入逻辑卷路径（如/dev/mapper/ubuntu--vg-ubuntu--lv）: " LV_PATH

# 创建物理卷并扩展卷组
sudo pvcreate $NEW_PARTITION && \
sudo vgextend $VG_NAME $NEW_PARTITION && \

# 扩展逻辑卷并调整文件系统
sudo lvextend -l +100%FREE $LV_PATH && \
if [ -f "/sbin/resize2fs" ]; then
    sudo resize2fs $LV_PATH
elif [ -f "/sbin/xfs_growfs" ]; then
    sudo xfs_growfs $(df -h | awk '$NF=="/"{print $1}')
else
    echo "未检测到支持的文件系统调整工具！"
    exit 1
fi

echo "扩容完成！当前根分区大小："
df -h | grep " /$"
```