# <center>注意事项</center>

## 安装问题

如果因选择版本以及编译环境差异，执行make时可能会报错： 例如aclocal-1.10: command not found和automake-1.10: command not found

```shell
cd . && /bin/sh /home/linux-ftools-master/missing --run aclocal-1.10 
/home/linux-ftools-master/missing: line 54: aclocal-1.10: command not found
WARNING: `aclocal-1.10' is missing on your system.  You should only need it if
         you modified `acinclude.m4' or `configure.ac'.  You might want
         to install the `Automake' and `Perl' packages.  Grab them from
         any GNU archive site.
 cd . && /bin/sh /home/linux-ftools-master/missing --run automake-1.10 --gnu 
/home/linux-ftools-master/missing: line 54: automake-1.10: command not found
WARNING: `automake-1.10' is missing on your system.  You should only need it if
         you modified `Makefile.am', `acinclude.m4' or `configure.ac'.
         You might want to install the `Automake' and `Perl' packages.
         Grab them from any GNU archive site.
cd . && /bin/sh /home/linux-ftools-master/missing --run autoconf
configure.ac:7: error: possibly undefined macro: AM_INIT_AUTOMAKE
      If this token and others are legitimate, please use m4_pattern_allow.
      See the Autoconf documentation.
make: *** [configure] Error 1
```

对于类似错误，网上一般会说要安装 automake 对应版本之类的。但实际上，服务器上已经有更高版本的 automake 工具。这其实是因为 Makefile.in 中写死了使用低版本的 automake 工具导致的，可以通过以下步骤解决：

1. 执行 aclocal，产生 aclocal.m4 文件
2. 执行 autoconf，生成 configure 文件
3. 执行 automake 命令，产生 Makefile.in： automake --add-missing
4. 执行 configure 命令，生成 Makefile 文件
5. 重新执行 make 和 make install，一切顺利

### 安装 code

```shell
cd linux-ftools/
aclocal && automake && autoconf
automake --add-missing
./configure; make; make install
```

## 命令行参数

```shell
 linux-fincore --pages=false --summarize --only-cached * 
```

- 其中 **星号** 代表查看任意文件的cache。也可以指定某个目录 /*，表示该目录中的所有文件。或指定某个具体的文件。
- 直接执行 linux-fincore --pages=false --summarize --only-cached * 将打印所有 cache 缓存文件，包括各种动态库文件，执行文件，log 文件等。
- 直接执行 linux-fincore --pages=false --summarize --only-cached /home/user/* 将打印 user 下对应的 cache 缓存。

```shell
find /mnt/disk1/djw/hippo/hippo_16c64g/data -type f -print0 | xargs -0 linux-fincore --pages=false --summarize --only-cached  > cache_data.txt
# 按照 total_pages 降序排序
awk 'NR == 1; NR > 1 {print $0 | "sort -k3,3nr"}' cache_data.txt > sorted_cache_data.txt
```

## 脚本

找到使用 RES 最高的前 10 个进程，然后用 lsof 找到该进程正在使用的文件，最后把这些文件交给 fincore 来处理

```shell
#!/bin/sh

#****************************************************************
# Copyright (C) 2019 uMore Ltd. All rights reserved.
# 
# File Name：fincore.sh
# Author：Pudding
# Date：2024 年 04 月 23 日
# Description：Script to identify files in cache used by top memory-consuming processes.
#
#****************************************************************

# Check if linux-fincore is installed
fincore_path="/usr/local/bin/linux-fincore"
if [ ! -x "$fincore_path" ]; then
    echo "You haven't installed linux-fincore yet. Please install it before running this script."
    exit 1
fi

# Check if lsof is available
if ! command -v lsof >/dev/null; then
    echo "lsof is not installed. Please install lsof before running this script."
    exit 1
fi

# Temporary files
pids_file="/tmp/cache.pids"
files_file="/tmp/cache.files"
fincore_file="/tmp/cache.fincore"

# Clean up previous temporary files if they exist
[ -f "$pids_file" ] && rm -f "$pids_file"
[ -f "$files_file" ] && rm -f "$files_file"
[ -f "$fincore_file" ] && rm -f "$fincore_file"

# Find the top 10 processes (by RSS) and write their PIDs to a temporary file
ps -e -o pid,rss | sort -nk2 -r | head -10 | awk '{print $1}' > "$pids_file"

# Use lsof to list the files opened by these processes
while read -r pid; do
    lsof -n -p "$pid" 2>/dev/null | awk '{print $9}' | grep '^/' >> "$files_file"
done < "$pids_file"

# Use fincore to analyze each file and append the results to the fincore file
sort -u "$files_file" | while read -r file_path; do
    if [ -f "$file_path" ]; then
        linux-fincore --only-cached --pages=false "$file_path" >> "$fincore_file"
    fi
done

# Output results
echo "Cache analysis complete."
echo "Results saved in $fincore_file"
exit 0
```