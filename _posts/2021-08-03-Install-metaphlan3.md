---
layout: article
title:  Recipe to Install Metaphlan 3 on Linux HPC
author: Ciao
description: Metaphlan 3 集群安装偏方
categories:
 - tech
tags:
 - tech
 - HPC
 - Metaphlan
 - bioinformatics
 - Linux
key: page-article
---
> Metaphlan 3 集群安装偏方

集群环境更新了，又到了重新部署软件和流程的季节。这次需要部署的是[MetaPhlAn 3.0](https://huttenhower.sph.harvard.edu/metaphlan/)。作为一款当下相当流行的metagenomics分析软件，哪怕你不想用也不得不拿来跑一遍做一做横评吧。

[MetaPhlAn 3.0](https://huttenhower.sph.harvard.edu/metaphlan/)主要利用了[conda](https://docs.conda.io/en/latest/index.html)进行更新管理，后者也是一款非常流行的python包管理软件。我这次直接抛弃了很久没有更新过的conda旧版本，打算重新装一个。

# HPC environment
当前的工作环境是 CentOS7:
```
uname -a
Linux ████-login-0-7.████.hpc  3.10.0-1127.el7.x86_64 #1 SMP Tue Mar 31 23:36:51 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```
> Sensitive content were covered


# Conda installation

从文档页面很快找到了对应版本的[安装教程](https://conda.io/projects/conda/en/latest/user-guide/install/linux.html)。
```bash
bash Miniconda3-latest-Linux-x86_64.sh
```
安装完成后可能会有一段初始化代码被添加到`~/.bashrc`中：
```
# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
__conda_setup="$('/████/chao/miniconda3/bin/conda' 'shell.bash' 'hook' 2> /dev/null)"
#__conda_setup=$(/████/chao/miniconda3/bin/conda shell.bash hook 2> /dev/null)
if [ $? -eq 0 ]; then
    eval "$__conda_setup"
else
    if [ -f "/████/chao/miniconda3/etc/profile.d/conda.sh" ]; then
        . "/████/chao/miniconda3/etc/profile.d/conda.sh"
    else
        export PATH="/████/chao/miniconda3/bin:$PATH"
    fi
fi
unset __conda_setup
# <<< conda initialize <<<
```
但这并不是我想要的。某些情况下，当安装`conda`的阵列出现不稳定的情况时，登录加载会占用很长时间，甚至会导致一直等待无法完成登录的尴尬情况。所以，我希望只在有需要的时候初始化`conda`。

将这段代码剪切到一个单独的启动脚本中，例如`~/.conda.init.sh`。设置一个快捷命令到`~/.bashrc`中：
```
alias condaInit="source ~/.conda.init.sh"
```
这样只需要输入`condaInit`就可以随时启动conda环境。

顺便，为了安装方便快速，可以设置[清华开源镜像](https://mirror.tuna.tsinghua.edu.cn/help/anaconda/)，或是它的姐妹站[北京外国语开源镜像站](https://mirrors.bfsu.edu.cn/help/anaconda/)。

# MetaPhlAn 3 installation

根据提供的[安装教程](https://github.com/biobakery/MetaPhlAn/wiki/MetaPhlAn-3.0#installation)，可以方便地利用`conda`进行安装:
```
conda create --name mpa3 -c bioconda python=3.7 metaphlan
```

### Troubleshooting

##### 1. glib version conflicts

若这时候遇到报错：
```
Collecting package metadata (current_repodata.json): done
Solving environment: failed with repodata from current_repodata.json, will retry with next repodata source.
Collecting package metadata (repodata.json): done
Solving environment: \                                                                                                                 failed

UnsatisfiableError: The following specifications were found to be incompatible with each other:

Output in format: Requested package -> Available versions

Package python conflicts for:
metaphlan -> python[version='2.7.*|>=3|>=3.7']
metaphlan -> biom-format -> python[version='3.4.*|3.5.*|3.6.*|>=2.7,<2.8.0a0|>=3.5,<3.6.0a0|>=3.6,<3.7.0a0|>=3.8,<3.9.0a0|>=3.7,<3.8.0a0|>=3.9,<3.10.0a0|>=2.7,<3|>=3.7.1,<3.8.0a0|<3.0.0']
python=3.7The following specifications were found to be incompatible with your system:

 - feature:/linux-64::__glibc==2.17=0
 - feature:|@/linux-64::__glibc==2.17=0

Your installed version is: 2.17
```
这可能是`bioconda`与`default`channel之间存在冲突导致的。这时候可以尝试不指定特定channel，让`conda`自己解决冲突。

首先在`~/.condarc`中设定频道优先级：
```
channels:
  - conda-forge
  - bioconda
  - defaults
```
然后重新尝试安装：
```
conda create --name mpa3 python=3.7 metaphlan
```

##### 2. rm_rf failed  

```
rm_rf failed for /████/chao/miniconda3/pkgs/seaborn-base-0.11.1-pyhd8ed1ab_1

rm_rf failed for /████/chao/miniconda3/pkgs/libssh2-1.9.0-ha56f1ee_6

rm_rf failed for /████/chao/miniconda3/pkgs/_libgcc_mutex-0.1-conda_forge

rm_rf failed for /████/chao/miniconda3/pkgs/wheel-0.36.2-pyhd3deb0d_0
```
这里可能是我安装清华源的过程中嫌网速太慢又换了一个源导致的……
参考[conda issue #8977](https://github.com/conda/conda/issues/8977#issuecomment-513779278):
```bash
set CONDA_USE_ONLY_TAR_BZ2=1
conda clean -ay
```
清除缓存后重新安装。  

##### 3. symbol lookup error

在安装数据库时：
```bash
metaphlan --install --bowtie2db ./
```
报错：
```
/████/chao/.conda/envs/mpa3/bin/bowtie2-build-s: symbol lookup error: /████/chao/.conda/envs/mpa3/bin/bowtie2-build-s: undefined symbol: _ZN3tbb10interface78internal15task_arena_base19internal_initializeEv
```
参考[bowtie2 issue #336](https://github.com/BenLangmead/bowtie2/issues/336),对`tbb`进行降级：

```
conda install tbb=2020.2
```
完成后重试安装，成功。
