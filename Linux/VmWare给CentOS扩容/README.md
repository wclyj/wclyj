# 给 VMware 中的 CentOS 加一块磁盘

## 前言

系统盘被镜像占满后，构建会失败。不必把原来的 20 GB 系统盘扩成 40 GB，可以新加一块 30 GB 的盘，挂到 `/data/images`。

## 添加新磁盘

关掉客户机。在设置里添加 SCSI 硬盘，容量 30 GB，磁盘文件单独存放。

## 进入系统

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT
```

示意：`sda` 20G 已挂到 `/`，`sdb` 30G 没有分区。`fdisk -l /dev/sdb` 会提示没有有效分区表。动手前对照容量，避免把系统盘认错。

## 磁盘分区

```bash
sudo fdisk /dev/sdb
```

依次输入 `n`、`p`，分区号和扇区直接回车，最后 `w` 写入。含义是新建一个占满整盘的主分区。执行 `sudo partprobe /dev/sdb`，确认出现 `sdb1`。此时还没有文件系统。

## 格式化磁盘

```bash
sudo mkfs.ext4 -L images /dev/sdb1
```

成功时会打印 journal 与 superblock 写完。这一步会清掉该分区上的数据。

## 挂载新磁盘

```bash
sudo mkdir -p /data/images
sudo mount /dev/sdb1 /data/images
df -h /data/images
```

`/data/images` 约 30G、使用率接近 0，就可以把构建目录指过来。

## 查看磁盘信息

再跑一次 `lsblk`，`sdb1` 的挂载点应为 `/data/images`，类型为 ext4。若 `df` 看不到，多半挂到了非空目录，先 `umount` 再查。

## 开启自动挂载

只 `mount` 的话，重启后会消失。用 UUID 写入 `/etc/fstab`，避免盘符漂移：

```bash
sudo blkid /dev/sdb1
echo 'UUID=c2a91e70-88d1-4b0e-9a11-images001 /data/images ext4 defaults 0 2' | sudo tee -a /etc/fstab
sudo mount -a
```

`0 2` 表示开机检查排在根文件系统之后。`mount -a` 没有报错再重启。写错 fstab 会进紧急模式。
