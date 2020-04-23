> 云原生系列第一课，容器的基本概念 

**容器是什么**

首先，我们先看一下操作系统对进程的管理，为了让人们能高效的使用计算机上的软件，操作系统对进程有如下规范：

- 这些进程可以相互看到、相互通信
- 这些系统使用同一个文件系统，可以对同一个文件进行读写操作（只要执行该进程的用户有权限）
- 这些进程使用相同的系统资源

随着人们对计算机的要求越来越高，使用越来越复杂，进程越来越多，出现了一些进程冲突的问题，比如多个进程改同一个文件，一个要读，一个要删，删的进程会影响读的进程，再比如一个进程消耗大量CPU和内存的时候，其他进程是无法正常工作的。

为了解决上述问题，Linux 和 Unix 操作系统做了如下工作：

- 通过 chroot 系统调用可以将子目录作为独立的文件系统，让进程只能看到这个独立文件系统，而看不到根目录文件系统
- 使用 namespace 技术，让进程之间不可见(不能看到此namespace之外的进程)，不能相互通信，认为自己就是 pid = 0 的进程
- 使用 cgroups 限制独立环境下的进程对资源的使用率，设置其CPU和内存限制

基于上述工作，我们就得到了一个独立文件系统、只能看到部分进程、只能使用有限的系统资源的进程集合，这个进程集合就称之为容器。

**镜像是什么**

而上述文件隔离，进程隔离，资源限制等技术称之为容器化技术。我们说容器具有独立的文件系统，只是该容器的进程只能看到这个文件系统，而看不到系统级的文件系统。但其实使用的仍然是系统的资源，驱动程序的内核功能依然是系统内核提供的。所以我们在容器中只需提供该容器内服务所需的依赖文件，让这个服务能够运行起来即可。而容器运行时所需的文件集合就称之为容器镜像。

**镜像与容器的关系**

- 抽象的说，就好像镜像是一个模版，而容器是基于这个模版生成的运行实例
- 具体的说，镜像是一个服务所需的所有文件集合，而容器就是一个运行着的服务

**容器的生命周期**

- 在使用 docker run 命令选择一个镜像来启动一个容器时，需要指定相应的运行程序，称之为 initial 进程。initial 进程启动，容器也随之启动，当 initial 进程退出，容器也随之退出。
- 容器的生命周期和 initial 进程的生命周期是一致的，initial 进程可以产生其他的子进程，当 initial 进程退出时，所有的子进程也随之退出，所有资源释放。
- 容器运行过程中可能会产生一些重要数据，当容器退出后，所有资源会被释放，文件会被删除，数据也就丢了。但我们有的时候并不想丢失这部分数据，想要将容器运行时产生的重要数据持久化下来，解决方案是我们需要声明一个数据卷(Volume)
- 数据卷其实就是一个文件目录，我们可以将需要持久化的数据保存到这个数据卷上。数据卷的生命周期是独立于容器的生命周期的。