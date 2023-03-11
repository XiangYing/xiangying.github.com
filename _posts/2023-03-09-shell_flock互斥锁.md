---
layout: post
title: shell_flock互斥锁
date: 2023-03-09
tags: shell   
---

## 什么是flock？
man手册的解释是使用shell脚本来管理锁

## 使用场景
### 参数
```
OPTIONS
-s, --shared
    Obtain a shared lock, sometimes called a read lock.

-x, -e, --exclusive
    Obtain an exclusive lock, sometimes called a write lock.  This is the default.

-u, --unlock
    Drop a lock.  This is usually not required, since a lock is automatically dropped when the file is closed.  However, it may be required in  special  cases,  for  example  if  the enclosed command group may have forked a background process which should not be holding the lock.

-n, --nb, --nonblock
    Fail rather than wait if the lock cannot be immediately acquired.  See the -E option for the exit code used.

-w, --wait, --timeout seconds
    Fail  if  the lock cannot be acquired within seconds.  Decimal fractional values are allowed.  See the -E option for the exit code used. The zero number of seconds is interpreted as --nonblock.

-o, --close
    Close the file descriptor on which the lock is held before executing command .  This is useful if command spawns a child process which should not be holding the lock.

-E, --conflict-exit-code number
    The exit code used when the -n option is in use, and the conflicting lock exists, or the -w option is in use, and the timeout is reached. The default value is 1.

-c, --command command
    Pass a single command, without arguments, to the shell with -c.

-h, --help
    Print a help message.

-V, --version
    Show version number and exit.
```
### 通过shell脚本或者命令行加锁方式实现互斥
#### 语法及实例
- `flock [options] <file|directory> <command> [command args]`
	- 实例
	```shell
	shell1>flock mylockfile sleep 60
	```
	```shell
	shell2> flock -E 4  -w 1 mylockfile echo abc;echo $?
	4
	```

- `flock [options] <file|directory> -c <command>`
	- 实例
	```shell
	shell1>flock /tmp -c cat
	```
	```shell
	shell2>flock -w 1 /tmp -c echo; /bin/echo $?
	1
	```

### 通过文件描述符加锁方式实现互斥
#### 语法及实例
- flock [options] <file descriptor number>
	- 实例1
	```shell
	shell1>cat test1.sh
	#/bin/bash
	(
         flock -n 9 || exit 1
         # ... commands executed under lock ...
         sleep 20
    ) 9>mylockfile
	```
	
		```
		shell1>./test1.sh
		```
		

		```shell
		shell2>./test1.sh
		echo $?
		1
		````
	> 解释一下：
	9>"$LOCK_FILE"  表示创建文件描述符9，指向锁文件，为何是9?8其实也是可以的，只是为了和当前脚本可能打开的文件描述符冲突(例如和0,1,2冲突)。
	flock -n 99 尝试对该文件描述符加锁

	- 实例2
	```shell
	shell1> cat test2.sh
	```

		```shell
		#!/bin/bash

		[ "${FLOCKER}" != "$0" ] && exec env FLOCKER="$0" flock -en -E 4  "$0" "$0" "$@" || :
		sleep 20
		```
		
		```
		shell1>./test2.sh
		```
		```shell
		shell2> ./test2.sh; echo $?
		4
		```
	>解释：这是flock的man手册中的例子，如果${FLOCKER}环境变量没有设置，则尝试将脚本本身加锁，如果加锁成功，则运行当前脚本，(并且带上原有的参数)，否则的话静默退出。
	

	
## 互斥任务实践
### 背景
由于业务脚本只能串行执行，为此通过引入flock工具可以很好的保证某一个脚本同一时刻有且只能有一个实例在执行。具体如下：

### 改造样例
- 在服务器分别创建如下三个脚本
	```shell
	# merge_script.sh
	#!/bin/bash
	[ "${FLOCKER}" != "$0" ] && exec env FLOCKER="$0" flock -en "$0" "$0" "$@" || :
	echo "start test.."
	./test1.sh
	./test2.sh
	echo "test end.."	
	```

	```shell
	# test1.sh
	#!/bin/bash

	echo "start run test1.."
	for i in `seq 1 20`
	do
	echo ${i}
	sleep 1
	done
	```

	```shell
	# test2.sh
	#!/bin/bash

	echo "start run test2.."
	for i in `seq 1 10`
	do
	echo ${i}
	sleep 1
	done
	```

	- 分别前后起两个同样的任务，内容如下：

	```shell
	while true
	do
	./merge_script.sh
	if [ $? -ne 0 ]
	then
		echo "exist merge task, wait 5s.."
		sleep 5
	else
	echo "merge done"
	break
	fi 
	done
	```
- 观察运行过程
在第一个任务执行完毕之前，第二个任务一直在返回输出`exist merge task, wait 5s..`,直到第一个任务执行完毕，第二个任务才开始打印输出。

