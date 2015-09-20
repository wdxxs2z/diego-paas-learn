# RLimit garden-linux 说明
garden在启动每个应用的时候都会在其中加入一个守护进程wsh,这个在早在warden里就已存在，对容器的操作几乎都是通过它来通信的，说白了就是进程间通信socket

## Garden-linux 我们如何调优容器中的性能？
container_daemon:Process -> container_daemon:proc_starter -> linux_container:running -> container_daemon:rlimits_manager </br>

看到linux_container:running：</br>
首先看到wsh的一些列的创建动作，指定用户和环境变量后开始对wsh设置资源限制描述 </br>

		func (c *LinuxContainer) Run(spec garden.ProcessSpec, processIO garden.ProcessIO) (garden.Process, error) {
			wshPath := path.Join(c.ContainerPath, "bin", "wsh")
			sockPath := path.Join(c.ContainerPath, "run", "wshd.sock")

			if spec.User == "" {
				c.logger.Error("linux_container: Run:", errors.New("linux_container: Run: A User for the process to run as must be specified."))
				return nil, errors.New("A User for the process to run as must be specified.")
			}

			args := []string{"--socket", sockPath, "--user", spec.User}

			specEnv, err := process.NewEnv(spec.Env)
			if err != nil {
				return nil, err
			}

			procEnv, err := process.NewEnv(c.Env)
			if err != nil {
				return nil, err
			}

			processEnv := procEnv.Merge(specEnv)

			for _, envVar := range processEnv.Array() {
				args = append(args, "--env", envVar)
			}

			if spec.Dir != "" {
				args = append(args, "--dir", spec.Dir)
			}

			processID := c.processIDPool.Next()
			c.logger.Info("next pid", lager.Data{"pid": processID})

			if c.Version.Compare(MissingVersion) == 0 {
				pidfile := path.Join(c.ContainerPath, "processes", fmt.Sprintf("%d.pid", processID))
				args = append(args, "--pidfile", pidfile)
			}

			args = append(args, spec.Path)

			wsh := exec.Command(wshPath, append(args, spec.Args...)...)
			
			//这里是对该描述符设置限制的地方
			setRLimitsEnv(wsh, spec.Limits)

			return c.processTracker.Run(processID, wsh, processIO, spec.TTY, c.processSignaller())
		}
		
进到container_daemon:rlimits_manager看到rlimits有哪些变量被限制</br>
		
		rLimitsMap := map[string]*rlimitEntry{
			"cpu":        &rlimitEntry{Id: RLIMIT_CPU, Max: RLIMIT_INFINITY},
			"fsize":      &rlimitEntry{Id: RLIMIT_FSIZE, Max: RLIMIT_INFINITY},
			"data":       &rlimitEntry{Id: RLIMIT_DATA, Max: RLIMIT_INFINITY},
			"stack":      &rlimitEntry{Id: RLIMIT_STACK, Max: RLIMIT_INFINITY},
			"core":       &rlimitEntry{Id: RLIMIT_CORE, Max: RLIMIT_INFINITY},
			"rss":        &rlimitEntry{Id: RLIMIT_RSS, Max: RLIMIT_INFINITY},
			"nproc":      &rlimitEntry{Id: RLIMIT_NPROC, Max: RLIMIT_INFINITY},
			"nofile":     &rlimitEntry{Id: RLIMIT_NOFILE, Max: maxNoFile},
			"memlock":    &rlimitEntry{Id: RLIMIT_MEMLOCK, Max: RLIMIT_INFINITY},
			"as":         &rlimitEntry{Id: RLIMIT_AS, Max: RLIMIT_INFINITY},
			"locks":      &rlimitEntry{Id: RLIMIT_LOCKS, Max: RLIMIT_INFINITY},
			"sigpending": &rlimitEntry{Id: RLIMIT_SIGPENDING, Max: RLIMIT_INFINITY},
			"msgqueue":   &rlimitEntry{Id: RLIMIT_MSGQUEUE, Max: RLIMIT_INFINITY},
			"nice":       &rlimitEntry{Id: RLIMIT_NICE, Max: RLIMIT_INFINITY},
			"rtprio":     &rlimitEntry{Id: RLIMIT_RTPRIO, Max: RLIMIT_INFINITY},
		}
		
为了后面的调优，分别说一下这写名次的意义:</br>

**RLIMIT_CPU:** 最大允许的CPU使用时间，秒为单位。当进程达到软限制，内核将给其发送SIGXCPU信号，这一信号的默认行为是终止进程的执行。然而，可以捕捉信号，处理句柄可将控制返回给主程序。如果进程继续耗费CPU时间，核心会以每秒一次的频率给其发送SIGXCPU信号，直到达到硬限制，那时将给进程发送 SIGKILL信号终止其执行。</br>
**RLIMIT_FSIZE:** 可以创建的文件的最大字节长度。当超过此软限制时，则向该进程发送SIGXFSZ信号。</br>
**RLIMIT_DATA:** 进程数据段的最大值 </br>
**RLIMIT_STACK:** 最大的进程堆栈，以字节为单位 默认为8M </br>
**RLIMIT_CORE:** 内核转存文件的最大长度 </br>
**RLIMIT_RSS:**  </br>
**RLIMIT_NPROC:** 用户可拥有的最大进程数 </br>
**RLIMIT_NOFILE:** 每个进程能打开的最多文件数，超出此值，将会产生EMFILE错误 </br>
**RLIMIT_MEMLOCK:** 进程可锁定在内存中的最大数据量，字节为单位 </br>
**RLIMIT_AS:** 进程的最大虚内存空间，字节为单位 </br>
**RLIMIT_LOCKS:** 进程可建立的锁和租赁的最大值 </br>
**RLIMIT_SIGPENDING:** 用户可拥有的最大挂起信号数 </br>
**RLIMIT_MSGQUEUE:** 进程可为POSIX消息队列分配的最大字节数 </br>
**RLIMIT_NICE:** 进程可通过setpriority() 或 nice()调用设置的最大完美值 </br>
**RLIMIT_RTPRIO:** 进程可通过sched_setscheduler 和 sched_setparam设置的最大实时优先级 </br>

* 默认设置：</br>

		root@ubuntu:~# ulimit -a
		core file size          (blocks, -c) 0
		data seg size           (kbytes, -d) unlimited
		scheduling priority             (-e) 0
		file size               (blocks, -f) unlimited
		pending signals                 (-i) 11727
		max locked memory       (kbytes, -l) 64
		max memory size         (kbytes, -m) unlimited
		open files                      (-n) 1024
		pipe size            (512 bytes, -p) 8
		POSIX message queues     (bytes, -q) 819200
		real-time priority              (-r) 0
		stack size              (kbytes, -s) 8192
		cpu time               (seconds, -t) unlimited
		max user processes              (-u) 11727
		virtual memory          (kbytes, -v) unlimited
		file locks                      (-x) unlimited
		
* 系统软，硬对比：</br>

		root@ubuntu:~# ulimit -c -n -s
		core file size          (blocks, -c) 0
		open files                      (-n) 1024
		stack size              (kbytes, -s) 8192
		
		root@ubuntu:~# ulimit -c -n -s -H
		core file size          (blocks, -c) unlimited
		open files                      (-n) 4096
		stack size              (kbytes, -s) unlimited
		
32位的linux系统里，默认用户空间为3G，默认栈为8M，所以3096/8=384个线程，也就是说默认每个进程可以创建384个线程，但码段和数据段等还要占用一些空间，主线程算一个线程，所以一共可以创建382个线程。</br>
为了突破内存的限制，可以有两种方法</br>
**1)** 用 ulimit -s 1024 减小默认的栈大小</br>
**2)** 调用 pthread_create 的时候用 pthread_attr_getstacksize 设置一个较小的栈大小</br>
要注意的是，即使这样的也无法突破**1024**个线程的硬限制，除非重新编译 C 库,ulimit -s 1024，则可以得到3054个线程</br>

在Linux内核2.2.x中能用如下命令修改： </br>

		echo ’8192’ > /proc/sys/fs/file-max
		echo ’32768’ > /proc/sys/fs/inode-max 

并将以上命令加到/etc/rc.c/rc.local文件中，以使系统每次重新启动时设置以上值。 </br>
在Linux内核2.4.x中需要修改原始码，然后重新编译内核才生效。编辑Linux内核原始码中的 include/linux/fs.h文件，将 NR_FILE 由8192改为 65536，将NR_RESERVED_FILES 由10 改为 128。编辑fs/inode.c 文件将 MAX_INODE 由16384改为262144。 </br>
一般情况下，最大打开文件数比较合理的设置为每4M物理内存256，比如256M内存能设为16384，而最大的使用的i节点的数目应该是最大打开文件数目的3倍到4倍。

### PS:
LINUX中进程的最大理论数计算：</br>

每个进程的局部段描述表LDT都作为一个独立的段而存在，在全局段描述表GDT中要有一个表项指向这个段的起始地址，并说明该段的长度以及其他一些 参数。除上之外，每个进程还有一个TSS结构(任务状态段)也是一样。所以，每个进程都要在全局段描述表GDT中占据两个表项。那么，GDT的容量有多大 呢？段寄存器中用作GDT表下标的位段宽度是13位，所以GDT中可以有8192个描述项。除一些系统的开销(例如GDT中的第2项和第3项分别用于内核 的代码段和数据段，第4项和第5项永远用于当前进程的代码段和数据段，第1项永远是0，等等)以外，尚有8180个表项可供使用，所以理论上系统中最大的 进程数量是4090。
</br>
最后看一下warden中的进程资源限制:</br>

		container_rlimits:
		  as: 4294967296 (4G)
		  nofile: 8192 (每个进程最多能打开8192个文件)
		  nproc: 512 (用户可拥有最大512个进程)