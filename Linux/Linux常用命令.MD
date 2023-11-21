# Linux常用命令

## 系统管理
+ 内存命令
  - `free -m`
  - 使用ps统计所有进程占用的总内存
    + `ps aux | awk '{mem += $6} END {print mem/1024/1024}'`

- 网络连接统计
  + `ss -ant | awk '{++S[$1]} END {for(a in S) print a, S[a]}'`
  + `netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'`

- 磁盘
  + IO 查看  `iostat -d -k -x`
  + 磁盘测速 `time dd if=/dev/zero of=/tmp/test bs=8k count=1000000`
  
## 文件查找

* 文件查找删除
    ```bash
    time find ./ -type f -name "tt*" -exec rm -f {} \;
    time find ./ -type f -name "tt*" -delete
    ```
* 在gz文件中查找内容
    ```bash
    zgrep -R --include=*.gz -H "pattern match" .
    zgrep -i -H "pattern match" *.gz  #  Ubuntu 16+
    ```

## 压缩解压


1. 压缩当前目录下文件夹/文件test到test.tar.gz: `tar -zcvf test.tar.gz test`
2. 解压缩当前目录下的file.tar.gz到file: `tar -zxvf file.tar.gz`
3. 在不解压的情况下查看压缩包的内容：`tar -tf test.tar.gz`
4. 解压： `gzip -d xxxxxxxxxxxxxxxxx.gz​`

> 参数详解
* 五个命令中必选一个
  + `-c`：建立压缩档案
  + `-x`：解压
  + `-t`：查看内容
  + `-r`：向压缩归档文件末尾追加文件
  + `-u`：更新原压缩包中的文件

* 这几个参数是可选的
  + `-z`：有gzip属性的
  + `-j`：有bz2属性的
  + `-Z`：有compress属性的
  + `-v`：显示所有过程
  + `-O`：将文件解开到标准输出

* -f必选参数
  + `-f`：使用档案名字，这个参数是最后一个参数，后面只能接档案名。

