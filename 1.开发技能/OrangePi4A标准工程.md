---
type: 原子
status: 整理中
mastery: 2
tags:
  - 技术/Linux
created: "2026-06-01"
updated: "2026-06-01"
---

# OrangePi4A标准工程

## SSD

- M.2 M-Key：M.2 M-Key Interface
- PCIe：Peripheral Component Interconnect Express 高速串行总线
- NVMe：Non-Volatile Memory Express 非易失性内存协议

**系统软件 → NVMe 协议 → PCIe 总线 → M.2 M-Key 插槽 → NVMe SSD**
1. 软件发读写指令，按**NVMe 规则**打包
2. 数据包通过**PCIe 串行总线**高速传输
3. 从**M.2 M-Key 物理卡槽**送入固态硬盘



## 📅2026-05-27

## 专有名词

- console：控制台

- Display Setting：显示设置
- resolution：分辨率   --- 选择合适的分辨率，解决虚拟机是否占满整个屏幕的问题
- scale：字体设置   --- 解决图标与控制台大小的问题


## 解决Git拉取代码失败问题

- git的机制：拉取速度过慢、文件过大，服务器会自动断开连接

- 扩大缓冲区：解决因缓冲区不足导致的禁用上传与拉取
- 放宽延时断开连接的条件
```bash
git config --global http.postBuffer 524288000
git config --global http.lowSpeedLimit 0
git config --global http.lowSpeedTime 999999
```




## ubuntu包管理机制汇总

## 源-公钥配对机制

**可以随意添加软件下载地址，但非官方的第三方地址，必须配套导入对方的官方公钥完成签名校验，才能正常下载使用；系统官方地址因预装公钥，无需额外操作**

```bash
# 添加官方 GPG 密钥（信任仓库）
curl -fsSL https://downloads.cursor.com/aptrepo/keyring.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/cursor.gpg

# 添加仓库源
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/cursor.gpg] https://downloads.cursor.com/aptrepo stable main" | sudo tee /etc/apt/sources.list.d/cursor.list >/dev/null

# 安装
sudo apt update && sudo apt install cursor
```


## 📅2026-05-28

## ubuntu安装软件步骤

源-公钥配对机制虽然安全并且常用，但是不是很通用，比较通用的类似于windows的软件安装，从官网下载安装包，然后解压安装

```bash
下载：
wget https://download.cursor.sh/linux/cursor_latest_amd64.deb

安装：
sudo dpkg -i cursor_latest_amd64.deb 
# 若提示依赖缺失 
sudo apt-get install -f
```
- AI给的路由地址，可能会有各种DNS问题或者网址问题



使用官网登陆下载Linux版本，在使用上述安装操作进行安装，似乎比`wget`更加靠谱

```bash
cd ~/Downloads

sudo dpkg -i cursor_3.5.38_amd64.deb

sudo apt install -f
```

- 使用这种方法安装，系统会帮你添加真正正确的“源+GPG密钥”，后续可以使用这种最正统的方式更新以及安装



## ubuntu上不同软件包的安装

**如果下载github代码的时候，使用公司的网络很慢，使用个人热点也许会快很多**










