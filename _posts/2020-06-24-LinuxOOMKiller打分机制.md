---
layout:     post
title:      Linux OOM Killer打分机制
subtitle:   OOM Killer如何选择杀死哪个进程
date:       2020-06-24
author:     J
catalog: true
tags:
    - Linux
---

当您的Linux机器内存**不足时**，内核会调用内存**OOM Killer**来释放一些内存。在运行大量内存密集型进程的服务器上经常会遇到这种情况。

在本文中，我们将更深入地了解何时调用OOM Killer，如何确定要杀死哪个进程以及是否可以防止它杀死重要的进程（如数据库）。

Linux内核会为每个正在运行的进程提供一个分数，以`oom_score`显示在可用内存不足的情况下终止该进程的可能性。分数与该进程使用的内存量成**正比**。分数是`10 x percent of memory used by process`。因此，最大分数为100％x 10 =1000。此外，如果进程以**特权用户**身份运行，则与普通用户进程使用相同的内存相比，该进程的**oom_score略低**。在Linux的早期版本（v2.6.32内核）中，有一种更为详尽的启发式算法可以计算出该分数。

该`oom_score`过程可以在找到`/proc`目录。假设您的流程的流程ID（pid）为42，`cat /proc/42/oom_score`将为您提供该流程的得分。

OOM Killer进行检查`oom_score_adj`以调整其最终计算出的分数。该文件存在于中`/proc/$pid/oom_score_adj`。您可以在此文件中添加较大的负分数，以确保您的进程被OOM Killer选择和终止的机会降低。的范围`oom_score_adj`可以从-1000到1000。如果您为其分配-1000，则它可以使用100％的内存，并且仍然避免被OOM Killer终止。另一方面，如果您为其分配1000，则即使内核使用最少的内存，Linux内核也会继续杀死该进程。

如何更改进程的`oom_score_adj`：

```shell
sudo echo -200 > /proc/$pid/oom_score_adj
```

我们需要以`root`用户身份执行此操作，或者`sudo`因为Linux不允许普通用户降低OOM分数。您可以在没有任何特殊权限的情况下以普通用户身份提高OOM分数。例如`echo 100 > /proc/42/oom_score_adj`

#### 显示所有正在运行的进程的OOM分数

```shell
#!/bin/bash
#    Displays running processes in descending order of OOM score
#      (skipping those with both score and adjust of zero).
#    https://dev.to/rrampage/surviving-the-linux-oom-killer-2ki9

contents-or-0 () { if [ -r "$1" ] ; then cat "$1" ; else echo 0 ; fi ; }

{
    header='# %8s %7s %9s %5s %5s %5s  %s\n'
    format="$(echo "$header" | sed 's/^./ /')"
    declare -a lines output
    IFS=$'\r\n' command eval 'lines=($(ps -e -o user,pid,rss))'
    shown=0 ; omits=0
    for n in $(eval echo "{1..$(expr ${#lines[@]} - 1)}") ; do # 1..skip header
        line="${lines[$n]}"
        case "$line" in *[0-9]*)
            set $line ; user=$1 ; pid=$2 ; rss=$3 ; shift 3
            oom_score=$(    contents-or-0  /proc/$pid/oom_score)
            oom_adj=$(      contents-or-0  /proc/$pid/oom_adj)
            oom_score_adj=$(contents-or-0  /proc/$pid/oom_score_adj)            
            if [ -f /proc/$pid/oom_score ] && \
               [ 0 -ne $oom_score -o 0 -ne $oom_score_adj -o 0 -ne $oom_adj ]
            then
                output[${#output[@]}]="$( \
                   printf "$format" \
                          "$user" \
                          "$pid" \
                          "$rss" \
                          "$oom_score" \
                          "$oom_score_adj" \
                          "$oom_adj" \
                          "$(cat /proc/$pid/cmdline | tr '\0' ' ' )" \
                )"
                (( ++shown ))
            else
                (( ++omits ))
            fi
            ;;
        esac
    done
    printf "$header"   ''   '' '' OOM   OOM   OOM ''
    printf "$header" User PID RSS Score ScAdj Adj \
        "Command (shown $shown, omits $omits)"
    for n in $(eval echo "{0..$(expr ${#output[@]} - 1)}") ; do
        echo "${output[$n]}"
    done | sort -k 4nr -k 5rn
}

#----eof
```

####  检查您的任何进程是否已被OOM杀死

最简单的方法是`grep`查看系统日志。在Ubuntu中：`grep -i kill /var/log/syslog`。如果某个进程被终止，您可能会得到如下结果`my_process invoked oom-killer: gfp_mask=0x201da, order=0, oom_score_adj=0`

#### 调整OOM分数的警告

请记住，OOM是更大问题的征兆-可用内存不足。解决此问题的最佳方法是通过增加可用内存（例如，更好的硬件）或将某些程序移动到其他计算机，或通过减少程序的内存消耗（例如，在可能的情况下分配更少的内存）。

OOM调整分数的太多调整将导致随机进程被杀死并且无法释放足够的内存。



#### 实验示例

```shell
root@v2ryvps:/tmp# bash display_oom_score.sh 
#                              OOM   OOM   OOM  
#     User     PID       RSS Score ScAdj   Adj  Command (shown 19, omits 46)
      root   28445     49800    49     0     0  /usr/bin/v2ray/v2ray -config /etc/v2ray/config.json 
      root   26577     21584    21     0     0  python /usr/local/shadowsocks/server.py -c /etc/shadowsocks.json -d start 
      root    2310     10360    10     0     0  /lib/systemd/systemd-journald 
      root   21305      8412     8     0     0  sshd: root@pts/1     
      root   27793      7460     7     0     0  /root/trojan/src/trojan -l /root/trojan-access.log -c /root/trojan/src/server.conf 
      root   21311      5204     5     0     0  -bash 
      root     419      5780     5     0     0  /lib/systemd/systemd-logind 
  systemd+    2306      5360     5     0     0  /lib/systemd/systemd-timesyncd 
      root     409      4452     4     0     0  /usr/sbin/rsyslogd -n -iNONE 
      root     488      4808     4     0     0  /lib/systemd/systemd --user 
      root    5720      4404     4     0     0  nginx: worker process                            
      root   16059      3100     3     0     0  bash display_oom_score.sh 
      root    4096      3044     3     0     0  nginx: master process /usr/sbin/nginx -g daemon on; master_process on; 
      root    3575      2644     2     0     0  /usr/sbin/cron -f 
      root     489      2100     2     0     0  (sd-pam) 
      root     481      1380     1     0     0  /sbin/agetty -o -p -- \u --noclear tty1 linux 
  message+     422      4136     0  -900   -15  /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only 
      root    3657      4424     0 -1000   -17  /lib/systemd/systemd-udevd 
      root    4503      6644     0 -1000   -17  /usr/sbin/sshd -D 

```

这个示例可以看出目前`oom_score` 分数最高是`v2ray pid:28445`进程，它的`oom_scoreAdj`为0 `oom_Adj`也为0

说明当这个vps的内存使用量不足时会优先把v2ray进程杀死来释放相应的内存，还有一个现象就是sshd 进程具有比较特殊的地位，它的`oom_scoreAdj`为-1000 `oom_Adj`为-17  说明`sshd`进程永远不能被OOM Killer杀死

