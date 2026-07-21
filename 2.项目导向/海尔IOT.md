# 本项目最重要操作

## 网址及账号

网址： https://gitlab.ldrobot.com

登录邮箱： bohan.wang@ce-link.com

用户名：celink-dev-wangbohan

## 常用指令

```bash
# 使用指令进行重启配网
cd /tmp/approm/
./event_tool -p -t "/event/keyboard" -e "network_config" -d "{}"
```

```bash
# 读取安卓日志--输入指令、开启读取日志；`ctrl+C `结束读取日志
adb shell logcat > D:\app_log\HW2T_android.log
```


## 抓取设备端日志

```bash
1. USB转网口连接待测机器，为机器分配固定IP地址
2. 乐动有权限的上位机软件，找到对应IP地址的机器，发送开始调试指令
3. MobaXterm进入ssh上位机
   用户名：root
   密码：autopack1688
4. 进入特定目录，杀掉自动挂起其他进程的进程，否则重启进程会有冲突
   [root@AutoPack_Sweeper:~]# cd /tmp/approm
   [root@AutoPack_Sweeper:/oem/bin/app]# killall autopack_manager
5. 杀掉旧的IOT进程，并进行重启
   [root@AutoPack_Sweeper:/oem/bin/app]# killall network*
   [root@AutoPack_Sweeper:/oem/bin/app]# ./network
```
- 中间可以使用`procrank`查看进程情况
- 海尔印度机型点击配网按键之后，不会杀死 `network`进程，可以直接取相关日志
- 海尔泰国水洗机`./network`之后点击配网案件，会自动杀掉`network`进程，需要再次执行一下`network`进行取日志

```bash
# 使用指令进行重启配网
cd /tmp/approm/
./event_tool -p -t "/event/keyboard" -e "network_config" -d "{}"
```




## 安卓手机抓取app日志

进入开发者选项，开启USB调试
- 开启开发者选项：设置->关于手机->连续点击版本号->手机提示“您现在处于开发者模式”
- 搜索“开发者选项”：进入->开启USB调试

###### CMD配置ADB

```bash
1.以管理员身份打开CMD
2.安装自动安装ADB并配置环境的包管理器：
@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"
3.一键安装ADB
choco install adb -y
4.验证是否安装成功
adb version
5.输入指令、开启读取日志；`ctrl+C `结束读取日志
adb shell logcat > D:\app_log\HW2T_android.log
```

**手机需要全程亮屏，并在手机上操作允许USB调试


### 处理海尔OSDK

1. 编译Linux适配层
```bash
注意linux适配层不再编进OSDK库里。单独提供源代码 随SDK一起提供 在tag包里。
使用方法：改一下linux目录里面的编译器路径，直接在源码目录一下 执行make
即可直接生成库
```
- 使用Linux适配层源码进行编译的时候，同时打包出静态库（`.a`）与动态库（`.so`），两者在内容上一直，在使用时二选一
- 本项目使用静态库（`.a`）编译IOT源码

编译之前需要改变目录权限，或者直接而进行加权处理

```bash
# 用 root（或 sudo）
sudo chown -R wangbohan808:wangbohan808 ~/sources/OSDK5.6.0-RC2_ssc9211d_sea_MQTT_softap_ble_test_20260618/OSDK5.6.0-RC2_ssc9211d_sea_MQTT_softap_ble_test_20260618/src/linux/output

# 或者直接删掉，再以 wangbohan808 重新 make
sudo rm -rf ~/sources/OSDK5.6.0-RC2_ssc9211d_sea_MQTT_softap_ble_test_20260618/OSDK5.6.0-RC2_ssc9211d_sea_MQTT_softap_ble_test_20260618/src/linux/output
```

sudo make

2. 将编译后的内容以及原本OSDK中的内容统一打包到IOT代码仓库（头文件和代码编译后的文件分别放到对应位置）
![](Pasted%20image%2020260717090610.png)

3. IOT代码的编译技巧
```bash
李新编写了一个 make.sh文件，可以直接进行执行编译

在build目录执行 make install 可以将需要替换的系统文件直接进行打包到iot目录
```

4. 替换文件到系统后需要进行加权



### IOT编译环境部署

```bash
# 在线下载安装Ubuntu18.04
wsl --install -d Ubuntu-18.04
```


- 查看当前环境下，所有用户的~目录配置
`/etc/passwd` 是 Linux 用户账户核心配置文件，**所有用户可读**，每行代表一个用户，用 `:` 分割 7 个字段，格式： `用户名:密码占位符:UID:GID:注释信息:家目录:登录Shell`

```bash
# 直接查询可登录的账户
grep "/bin/bash" /etc/passwd
```

- 切换用户
```bash
wangbohan808@wangbohan:~$ sudo -i
root@wangbohan:~# su - wangbohan808
wangbohan808@wangbohan:~$
```

- 创建统一的软链接`peter`，源代码中指向`peter`的内容都可以重定向到自己的用户
```bash
root@wangbohan:/home# ls
wangbohan808
root@wangbohan:/home# ln -s wangbohan808 peter
root@wangbohan:/home# ls
peter  wangbohan808
```


- 安装gcc工具
```bash
sudo apt-get install build-essential
```

- 复制芯片的交叉编译工具链到/home/wangbohan808 目录
```bash
wangbohan808@wangbohan:~$ cd /mnt/f/Resource/Linux/Haier_IOT
wangbohan808@wangbohan:/mnt/f/Resource/Linux/Haier_IOT$ cp ssd222d_sdk.tar.gz ~
wangbohan808@wangbohan:/mnt/f/Resource/Linux/Haier_IOT$ cd ~
wangbohan808@wangbohan:~$ tar -zxvf ssd222d_sdk.tar.gz
```

- 克隆代码
```bash
git clone https://gitlab.ldrobot.com/ldrobot/autopack/network_haier_normal.git

celink-dev-wangbohan
```

- 切换实际的代码开发分支
```bash
# 更新远程分支列表，同步远端所有分支名称
git fetch origin
# 查看所有远程分支，确认目标分支名
git branch -r

git checkout -b dev origin/dev
# 语法：git checkout -b 本地分支名 远程仓库/远程分支名
# 创建并切换本地dev分支，关联远程dev
git switch -c dev origin/dev

# 旧版git
git checkout dev
# 新版git
git switch dev
```

- 查看当前ubuntu版本
```bash
wangbohan808@wangbohan:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.2 LTS
Release:        18.04
Codename:       bionic
wangbohan808@wangbohan:~$
```

- 库名进行软连接，以防潜在的风险
```bash
sudo ln -s /usr/lib/x86_64-linux-gnu/libmpfr.so.6 /usr/lib/x86_64-linux-gnu/libmpfr.so.4
```
