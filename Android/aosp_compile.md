# AOSP源码编译（Android 9.0）

AOSP全称Android Open Source Project,是Android操作系统的源码仓库,国内的手机厂商都是在Android源码的基础上开发了国内的定制系统

1. 安装git

   ```shell
   sudo apt install git
   ```

2. 安装Repo

   ```shell
    mkdir ~/bin
    PATH=~/bin:$PATH
    curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
    chmod a+x ~/bin/repo
   ```

   上面脚本的作用是创建bin目录，下载repo文件到bin目录，修改repo的权限为可执行，把bin添加到环境变量中。最后一步添加环境变量的方式只在当前命令行窗口有效。如果想要对所有命令行窗口都有效，需要修改.bashrc配置文件

3. 下载源码。官方下载源码的步骤需要链接外网，国内下载巨慢而且非常不稳定，所以强烈推荐使用国内镜像，[清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)或者[中科大开源软件镜像](https://lug.ustc.edu.cn/wiki/mirrors/help/aosp)，这里我选择了清华大学开源软件镜像站，由于Android源码现在非常巨大，所以推荐先下载初始化包，这个包大约46G左右
4. 建立工作目录，记得选择容量大的硬盘

   ```shell
    mkdir WORKING_DIRECTORY
    cd WORKING_DIRECTORY
   ```

   把之前下载的初始化包拷到工作目录并解压

   ```shell
   tar -xvf aosp-latest.tar
   ```

5. 初始化仓库并同步源码

   ```shell
   repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest
   ```

    如果需要某个特定版本的Android源码，可用-b参数指定分支，这里我指定为android-9.0.0_r1

    ```shell
    repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-9.0.0_r1
    ```

    同步源码树

    ```shell
    repo sync
    ```
