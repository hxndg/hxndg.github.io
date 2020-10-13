---
layout:     post   				    # 使用的布局（不需要改）
title:      USTACk系统的kernelapi
subtitle:   kernelapi怎么做的？ #副标题
date:       2020-10-12 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 进程间通信
---

# USTACk系统的kernelapi

## 前言
我原先一直没有很认真的看看kernelapi的实现，只是大致浏览了一下实现。今天来仔细看一下。kernelapi实际上是跨进程调用函数的范例，backend进程和atcp进程通过socket通信并将相应的结果写回去。

## 代码分析
先看看有哪些文件：
```
-rwxr-xr-x. 1 root root 267679 Sep 25 06:16 addCommands.pm
-rwxr-xr-x. 1 root root   4006 Sep 25 06:01 cmd_api.c
-rwxr-xr-x. 1 root root    762 Sep 25 06:01 cmd_api.h
-rwxr-xr-x. 1 root root    828 Sep 25 06:01 kernelapiVersion.h
-rwxr-xr-x. 1 root root   1499 Sep 25 06:01 kernelWrapGenMain2Coms.pl
-rwxr-xr-x. 1 root root    668 Sep 25 06:01 kernelWrapGenMain.pl
-rwxr-xr-x. 1 root root  22959 Sep 25 06:01 kernelWrapGen.pm
-rw-r--r--. 1 root root    576 Sep 25 06:01 Makefile
-rwxr-xr-x. 1 root root   1622 Sep 25 06:01 userWrapGenMain2Coms.pl
-rwxr-xr-x. 1 root root   1257 Sep 25 06:01 userWrapGenMain.pl
-rwxr-xr-x. 1 root root  17077 Sep 25 06:01 userWrapGen.pm

```

addCommands.pm对应于要添加的kernelapi，也就是atcp进程中真正调用的函数，cmd_api.c对应于kernelapi的实现，剩下的几个pl与pm文件，负责生成真正的被调用函数。

### MakeFile部分
先看看makefile干了点什么，核心是userWrapper.c userWrapper.h 和userWrapperAdd.c等文件的生成，这几个文件通过调用两个命令，`userWrapGenMain2Coms.pl userWrapGen.pm addCommands.pm commands.pm`与`perl -I${.CURDIR}/../libparser -I${.CURDIR} ${.CURDIR}/userWrapGenMain2Coms.pl`来生成。所以这两条命令究竟做了什么？
```
	# ArrayOS

LIB=    kernelapi
SRCS=   cmd_api.c cmd_api.h kernelapiVersion.h userWrapper.c userWrapper.h
INCS=   cmd_api.h kernelapiVersion.h userWrapper.h userWrapperAdd.h
CLEANFILES+= userWrapper.c userWrapper.h userWrapperAdd.c userWrapperAdd.h

.PATH: ${.CURDIR}/../libparser

CFLAGS+= -I${.CURDIR} -fPIC \
        -I${.CURDIR}/../../../src/sys

userWrapper.c userWrapper.h userWrapperAdd.c userWrapperAdd.h: userWrapGenMain2Coms.pl userWrapGen.pm addCommands.pm commands.pm
        perl -I${.CURDIR}/../libparser -I${.CURDIR} ${.CURDIR}/userWrapGenMain2Coms.pl

.include <bsd.lib.mk>

```

### perl执行部分

userWrapGenMain2Coms.pl脚本会调用userWrapGen.pm对addCommands.pm和commands.pm的每个函数做操作，这个文件的核心实际上是`write1Func`，根据固定的格式生成每个真实调用的函数。把一个kernelapi函数的原始格式列在了首部。我们来分析下这段核心代码干了些什么：首先获得函数名和参数列表。之后生成一个返回类型为int 函数名字同获得的函数名相同的新函数，之后对函数参数进行转换（修改后写到新函数的定义里），比方说OUTDATA就是转换为内存地址+长度，到这里新函数的定义都不就完成了。下面生成函数体。

函数体部分的操作实际上就是调用kernelapi的主体了。这里生命了一个pCmd结构，pCmd本身是一个struct funcName_s的结构体，这个结构体本身实际上就包含了各种参数的长度。但没有计算字符串，因为字符串往往是边长。因此这里需要并对参数进行分析，如果参数有字符串，那么就需要把字符串的值拷贝进来。

之后对pCmd的版本，及函数序列号赋值，并指定好诸如string参数的内容，这里可以看到，每次都是把这类型的参数放到最后的位置，这样操作的好处很明显，简单。之后调用`cmd_api( pCmd, totalSize, &pOut, &pOutLen)`去执行被调用的函数。在最后我附加了一个`ssl_get_special_user_all_host_config_kern`的转换结果。

```
	{
		cmd_attribute => "CMD_KERN_API|CMD_KAPI_NOLOCK",
		function_name => "ssl_get_special_user_all_host_config_kern",
		function_args => [
					{type => "STRING"},
					{type => "OUTDATA"},
				],
	}
	
	
	sub write1Func{
	my ($fh, $rec) = @_;

	# extruct function name
	my $funcName = $rec->{function_name};

	# extract arguments list
	my $argList = $rec->{function_args};

	# write a function entry 
	print $fh "int $funcName( ";  
	
	########################################################
	#
	# list of arguments
	#
	my $i = 0;   # no. of arguments to the function

	for $a ( @$argList ) {
		$i += 1;
		if($i > 1) { print $fh "\, "; }  # second argument on
		if( $a->{type} eq "U64" ) {
			print $fh " uint64_t arg$i ";
		}
		if( $a->{type} eq "U32" ) {
			print $fh " uint32_t arg$i ";
		}
		if( $a->{type} eq "U16" ) {
			print $fh " uint16_t arg$i ";
		}
		if( $a->{type} eq "STRING" || $a->{type} eq "XSTRING" || $a->{type} eq "IPADDR") {	# null terminated string
			print $fh " char \*arg$i ";
		}
		if( ($a->{type} eq "DOTTEDIP") || ($a->{type} eq "IPMASK") ) {  
			print $fh " uint32_t arg$i ";
		}
		if( $a->{type} eq "INDATA" ) {	
			print $fh " void *arg$i, int32_t lenArg$i";
		}
		if( $a->{type} eq "OUTDATA" ) {	
			print $fh " void **arg$i, int *lenArg$i";
		}
	}
	print $fh "  )\n";  # end of argument list

	############################################################
	#
	# variable definitions
	#
	print $fh "{\n";  
	print $fh "    struct $funcName\_s  *pCmd = NULL;\n";
	print $fh "    int   retval;\n";
	print $fh "    int   totalSize = sizeof(struct $funcName\_s);\n"; 

	# can be no output data but this is a must
	print $fh "    void  *pOut;\n";
	print $fh "    int   pOutLen;\n\n";
	
	if( isPointerData($argList) eq 1 ) {
		print $fh "    /* pointer for STRING and INDATA  */\n";
		print $fh "    char *pData;\n";
			
	}
	############################################################
	#
	# create malloc for input pointer to cmd_api()
	#
	print $fh "    /* set parameters to cmd_api() */\n"; 
	
	# if it has input data, allocate memory for the data too
	$i = 0;
	for $a ( @$argList ) {	# add the lengths of input data
		$i += 1;
		if( $a->{type} eq "U64" ) { }
		if( $a->{type} eq "U32" ) { }	# just for count args
		if( $a->{type} eq "U16" ) { }	
		if( ($a->{type} eq "DOTTEDIP") || ($a->{type} eq "IPMASK") ) { }
		if( $a->{type} eq "OUTDATA" ) {}	# skip output 
		if( $a->{type} eq "STRING" || $a->{type} eq "XSTRING" || $a->{type} eq "IPADDR") {  # string is out of the struct
			print $fh "    totalSize += strlen(arg$i) + 1;\n";
		}
		if( $a->{type} eq "INDATA" ) {  # indata is out of the struct
			print $fh "    totalSize += lenArg$i;\n";
		}
	}
	print $fh "\n    /* allocate memory for interface with cmd_api */\n";
	print $fh "    pCmd = (struct $funcName\_s *)malloc( totalSize );\n"; 
	print $fh "    /*  check if memory is allocated */\n";
	print $fh "    if( pCmd == NULL ) {\n";
        print $fh "           perror(\"userWrapper $funcName : malloc\");\n";
	print $fh "           return(-1);\n";
	print $fh "    }\n\n";

	############################################################
	#
	# parameter setting part.
	# 

	# set version number defined in kernelapiVersion.h
	print $fh "    pCmd->version = KERNELAPI_VERSION;\n";

	# function index number is a function name in capital letters
	my $funcNoConstant = $rec->{function_name};
	$funcNoConstant =~ tr/a-z/A-Z/;	
	print $fh "    pCmd->index = $funcNoConstant;\n";

	print $fh "    /* cut function index and length */\n";
	print $fh "    pCmd->paramLength = totalSize - SIZE_NOT_PARAM;\n\n";

	if( isPointerData($argList) eq 1 ) {
		print $fh "    /* set the pointer for the fist data  */\n";
		print $fh "    pData = (char *)((char *)pCmd + sizeof(struct $funcName\_s));\n\n";	
	}
	
	$i = 0;   # count arg 
	my $j = 0;
	for $a ( @$argList ) {
		$i += 1;
		
		if( $a->{type} eq "OUTDATA") {}  # for counting arguments

		if( $a->{type} eq "U64") {
			$j += 1;
			print $fh "    pCmd->param$j = htobe64(arg$i);\n";
		}
		if( $a->{type} eq "U32") {
			$j += 1;
			print $fh "    pCmd->param$j = htonl(arg$i);\n";
		}
		if( $a->{type} eq "U16") {
			$j += 1;
			print $fh "    pCmd->param$j = htons(arg$i);\n";
		}
		if( ($a->{type} eq "DOTTEDIP") || ($a->{type} eq "IPMASK") ) {
			$j += 1;
			print $fh "    pCmd->param$j = htonl(arg$i);\n";
		}
		if( $a->{type} eq "STRING" || $a->{type} eq "XSTRING" || $a->{type} eq "IPADDR") {	# copy input data 
			print $fh "    /* set input data in the allocated memory */\n";
			print $fh "    bcopy( arg$i, pData, (strlen(arg$i) + 1) );\n";
			print $fh "    /* for the next data */\n";
			print $fh "    pData += strlen(arg$i) + 1;\n\n";
		}
		if( $a->{type} eq "INDATA" ) {	# set length in struct and copy input data 
			$j += 1;
			print $fh "    pCmd->param$j = lenArg$i;\n"; 

			print $fh "    /* set input data in the allocated memory */\n";
			print $fh "    bcopy( arg$i, pData, lenArg$i );\n";
			print $fh "    /* for the next data */\n";
			print $fh "    pData += lenArg$i;\n\n";
		}
	}

	############################################################
	#
	# API function calling part
	#
	print $fh "\n    retval = cmd_api( pCmd, totalSize, &pOut, &pOutLen);\n";

	############################################################
	#
	# set an output pointer and the length 
	# 
	$i = 0;
	for $a ( @$argList ) {
		$i += 1;
		if( $a->{type} eq "U64") {}
		if( $a->{type} eq "U32") {} # do nothing for counting argments
		if( $a->{type} eq "U16") {}
		if( ($a->{type} eq "DOTTEDIP") || ($a->{type} eq "IPMASK") ) {}
		if( $a->{type} eq "STRING" || $a->{type} eq "XSTRING" || $a->{type} eq "IPADDR") {}
		if( $a->{type} eq "INDATA") {}
		if( $a->{type} eq "OUTDATA" ) {
			print $fh "    /* set output data pointer and length */\n";
			print $fh "    *arg$i = pOut;\n";
			print $fh "    *lenArg$i = pOutLen;\n";
		}
	}

	############################################################
	#
	# free pCmd
	#
	print $fh "    free(pCmd);\n";

	############################################################
	#
	# return statement and the end of the function
	#
	print $fh "\n    return(retval);\n";
	print $fh " }\n\n";
}


int ssl_get_special_user_all_host_config_kern(  char *arg1 ,  void **arg2, int *lenArg2  )
{
    struct ssl_get_special_user_all_host_config_kern_s  *pCmd = NULL; 
    int   retval;
    int   totalSize = sizeof(struct ssl_get_special_user_all_host_config_kern_s);
    void  *pOut;
    int   pOutLen;

    /* pointer for STRING and INDATA  */
    char *pData;
    /* set parameters to cmd_api() */
    totalSize += strlen(arg1) + 1;

    /* allocate memory for interface with cmd_api */
    pCmd = (struct ssl_get_special_user_all_host_config_kern_s *)malloc( totalSize );
    /*  check if memory is allocated */
    if( pCmd == NULL ) {
           perror("userWrapper ssl_get_special_user_all_host_config_kern : malloc");
           return(-1);
    }

    pCmd->version = KERNELAPI_VERSION;
    pCmd->index = SSL_GET_SPECIAL_USER_ALL_HOST_CONFIG_KERN;
    /* cut function index and length */
    pCmd->paramLength = totalSize - SIZE_NOT_PARAM;

    /* set the pointer for the fist data  */
    pData = (char *)((char *)pCmd + sizeof(struct ssl_get_special_user_all_host_config_kern_s));

    /* set input data in the allocated memory */
    bcopy( arg1, pData, (strlen(arg1) + 1) );
    /* for the next data */
    pData += strlen(arg1) + 1;


    retval = cmd_api( pCmd, totalSize, &pOut, &pOutLen);
    /* set output data pointer and length */
    *arg2 = pOut;
    *lenArg2 = pOutLen;
    free(pCmd);

    return(retval);
 }

```

### cmd_api的执行

上面看到末尾发现本质上是包裹了一层cmd_api，因此还是得看看cmd_api的实现。这个函数实际上非常简单就是穿个命令过去，然后去去执行的结果，我们重点看linux的实现版本，首先申明一个socket，如果不使用全局的cmd_kern_sd描述符就创建新的socket描述符，之后获取socket地址，socket地址是以文件方式保存，并建立连接。该socket为unix域socket，且没有修改为非阻塞模式。这里有个变量`multithread_enable`，这东西负责干什么？实际上是负责是否新创建socket描述符还是使用全局的cmd_kern_sd描述符，我们这里认为只是用cmd_kern_sd描述符。

接着向下看，向描述符写入cmd，判断是否写成功，之后读取rsp。如果rsp的p_length大于0，那么还需要读取并打印p_length，如果d_length大于0，那么还需要读取出d_length，并且存储到outp，也就是pCmd当中，也就是说跨内存通信的内容最终是内容被读取。这两个结构体的作用是什么？我们等到下面分析atcp中执行操作的时候再说。

好，总结下，这里藏着两个问题
+ 我们如果传入了诸如outdata的东西的话，backend进程和atcp进程并不共享内存地址，如何传递结构呢？就拿`ssl_get_special_user_all_host_config_kern`函数来说，backend进程调用的时候命令为：
```c
	char	*vhostdata = NULL;
	ssl_get_special_user_all_host_config_kern("", (void**)&vhostdata, &vhostdataLen);
```
	将vhostdata取地址转换为通用类型，并对长度取地址。在cmd_api调用时，pOut就是要输出的内容的地址，我们对vhostdata取地址再取值并赋值为运算结束的pOut，相当于vhostdata==pOut。后面赋值的时候发现存储的buf和vhostdata的地址一样，所以这里没有共享内存，只有网络字节序等等。
+ rsp结构体里面的d_length和p_length到底有什么用呢？因为有时需要获取一些输出的结果，那么就使用rsp->d_length来表示该数据的长度。并使用套接字传递这部分数据。对于客户端而言，读取时阻塞读，会不断尝试读取。当读取完成，会关闭socket。

这里我们还要再赘述一下unix域socket，UNIX域套接字用于同一台机器的进程通信。UNIX域套接字效率更高，只复制数据，不需要校验和，不需要顺序号。

```
	
	int
cmd_api(void *cmd, int size, void **outp, int *out_size)
{
#if defined(__linux__)
	struct sockaddr_un sun;
	char *spath;
#else
	struct sockaddr_in server_addr;
#endif
	cmd_rsp_hdr_t rsp;
	int sd;
	int	 new_sd = 0;
	char *buf;
	int  len, nbytes;

	/* setup default values; otherwise, if we get no data back, */
	/* these values are never set and the calling function is   */
	/* left with stack garbage values.                          */
	*out_size = 0;
	*outp = NULL;
	buf = NULL;

again:
	if (multithread_enable) {
		/* do not touch cmd_kern_sd, always create new socket */
		sd = 0;
	} else {
		sd = cmd_kern_sd;
	}

	if (sd == 0) {
#if defined(__linux__)
		if ((sd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0 ) {
#else
		if ((sd = socket(AF_INET, SOCK_STREAM, 0)) < 0 ) {
#endif
			perror("cmd_api: socket");
			return(-1);
		}

#if defined(__linux__)
		spath = getenv("KERNELAPI_SOCK");
		if (spath == NULL) {
			spath = "/tmp/kernelapi.sock";
		}

		/* Connect to the destination socket */
		bzero(&sun, sizeof(sun));
		strcpy(sun.sun_path, spath);
		sun.sun_family = AF_UNIX;
		if (connect(sd, (struct sockaddr *) &sun, sizeof(struct sockaddr_un)) < 0) {
#else
		bzero((char *)&server_addr, sizeof(server_addr));
		server_addr.sin_family = AF_INET;
		server_addr.sin_port = htons(KERNAPI_PORT);
		server_addr.sin_addr.s_addr = inet_addr("127.0.0.1");

		if (connect(sd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
#endif
			perror("cmd_api: connect");
			goto error;
		}
		new_sd = 1;
		cmd_kern_sd = sd;
	}

	if (write(sd, cmd, size) != size) {
		if(new_sd == 1) {
			perror("cmd_api: write");
			goto error;
		} else {
			close(sd);
			cmd_kern_sd = 0;
			goto again;
		}
	}

	if (read(sd, &rsp, sizeof(rsp)) != sizeof(rsp)) {
		perror("cmd_api: read");
		goto error;
	} else {
		if (rsp.p_length > 0) {
			/* handle the printf */
			if ((buf = (char *)malloc(rsp.p_length+1)) == NULL) {
				perror("cmd_api: malloc");
				goto error;
			}
			len = rsp.p_length;
			while (len && (nbytes = read(sd, buf, len)) > 0) {
				buf[nbytes] = '\0';
				printf("%s", buf);
				len -= nbytes;
			}
			if ( len != 0) {
				perror("cmd_api: read");
				goto error;
			}

			free(buf);
			buf = NULL;
		}
		if (rsp.d_length > 0) {
			/* handle configuration saving */
			if ((buf = (char *)malloc(rsp.d_length+1)) == NULL) {
				perror("cmd_api: malloc");
				goto error;
			}

 			len = rsp.d_length;
			buf[len] = '\0';    /* make sure string data terminated with '\0' */
			*out_size = rsp.d_length;
			*outp = buf;

			while (len && (nbytes = read(sd, buf, len)) > 0) {
				buf += nbytes;
				len -= nbytes;
			}

			if ( len != 0) {
				perror("cmd_api: read");
				if (*outp != NULL) {
					free(*outp);
					*outp = NULL;
					*out_size = 0;
				}
				buf = NULL;
				goto error;
			}

			buf = NULL;
		}
	}

	if (multithread_enable) {
		close(sd);
	}

	return(rsp.status);
error:
	close(sd);
	if (!multithread_enable) {
		cmd_kern_sd = 0;
	}

	if (buf != NULL) {
		free(buf);
		buf = NULL;
	}

	return -1;
}

```

### kernelapi真正的执行部分

#### 初始化与management_epoll_wait
kernelapi真正的执行部分需要有一些前置的东西，这里我直接简单说说带过了，因为涉及到我们系统的设计。管理线程原本是freebsd上的内核线程，现在移植到Linux变成用户态的一个线程，该线程负责kernelapi通信，从而对atcp进程进行操作。管理线程首先需要初始化监听socket等待客户端的连接，函数是`management_epoll_init`，创建监听描述符绑之后，调用epoll来添加读事件(EPOLLIN)，该函数的末尾有一段:
```
	bzero(&kapi_conns[0], sizeof(kapi_conns));
	for (int i = 0; i < KAPI_MAX_CONNS; i++) {
		kapi_conns[i].ns = -1;
		kapi_conns[i].reqs[0].p_iov_idx = -1;
	}
```
这部分实际上对应于建立的连接，即同时最多多少个backend进程和管理线程通信。简单而言，一开始没有客户连接的时候，epoll只监听listen_fd监听描述符，如果有客户端连接了，那么新创建描述符(new socket，ns)并存储到kapi_conn[i]的ns中，并添加到epoll中监听，并设置该新socket为非阻塞socket。新添加ns和等待ns/listen_fd的操作就是在`management_epoll_wait`中实现的。

`management_epoll_init`函数执行完毕，进入一个for循环，这个循环里面是管理线程的核心。

```c
	for(;;) {
		/* kernel api */
		if (management_epoll_wait()) {
			kapi_handle_req_atcp();
			kapi_handle_rsp();
		}
...
	
```

`management_epoll_wait`函数负责监听描述符，如果是listen_fd就新创建连接描述符，如果是已经创建出来的描述符就监听读事件和可写事件，因为可能写的数据超过了缓冲区的大小，比方说超过可8k，那么管理线程需要等待缓冲区数据被读取，从而能够继续读写。这里对listen_fd的处理不多说，我们看看读事件的处理。backend进程通信的时候我们知道实际上是向全局描述符kern_cmd_sd写信息，那么此时调用的实际上是`kapi_handle_req`函数，该函数负责读取监听到的数据，并处理读取到的数据。

```
int
management_epoll_wait(void)
{
	int n;
	struct sockaddr_un sun_n;
	socklen_t sl;
	int ns;
	struct epoll_event events[MAX_EPOLL_EVENT];
	int req_available = 0;

	bzero(events, sizeof(events));
	n = epoll_wait(kapi_epoll_fd, events, MAX_EPOLL_EVENT, 1);
	if (n <= 0) {
		return 0;
	}
	for (int i = 0; i < n; i++) {
		if (events[i].data.fd == listen_fd) {
			bzero(&sun_n, sizeof(sun_n));
			sl = sizeof(struct sockaddr_un);
			ns = accept(listen_fd, (struct sockaddr *) &sun_n, &sl);
			ARRAY_DEBUG("debug accept %d\n", ns);
			if (ns < 0) {
				fprintf(stderr, "%s: accept failed: %d\n", __func__, errno);
			} else {
				ARRAY_DEBUG("debug kapi_register_conn %d\n", ns);
				if (kapi_register_conn(ns) < 0) {
					fprintf(stderr, "%s: register conn failed: %d\n", __func__, errno);
					/* if register epoll failed, close it*/
					close(ns);
					continue;
				}
			}
		} else {
			if (events[i].events & EPOLLIN) {
				ARRAY_DEBUG("debug kapi_handle_req\n");
				kapi_handle_req(&events[i]);
				req_available = 1;
			}
			if (events[i].events & EPOLLOUT) {
				kapi_send_left_data(&events[i]);
			}
		}
	}
	return req_available;
}
```

`kapi_handle_req`函数处理读取到的数据，首次调用的时候先创建缓存`conn->buf`，长度为`KAPI_CONN_BUF_SIZE`。之后判断会尝试向buf中读取数据，我们的数据和der格式有点像，可以知道到底请求的长度是多少，所以如果已经读到的数据小于cmd_req_hdr+req->cmd_size，那么说明请求长度太长了，还没读完等待下次再进来就可以了。如果已经读完完整的请求，调用函数`kapi_req_enqueue`把请求入队。

```c
	typedef struct cmd_req_hdr
{
    uint32_t cmd_index;
    uint32_t cmd_size;
    int32_t version;
} cmd_req_hdr_t;


	
	static void
kapi_handle_req(struct epoll_event *ev)
{
	int ret;
	struct kapi_conn * conn = (ev->data.ptr);
	int ns = conn->ns;
	uint32_t cmd_size = 0;
	struct cmd_req_hdr *req;
	char * buf = NULL;
	int left_buf_len = 0;

	/* malloc buffer after establish connection*/
	if (conn->buf == NULL) {
		conn->buf = malloc(KAPI_CONN_BUF_SIZE);
		if (conn->buf == NULL) {
			fprintf(stderr, "%s: malloc failed: %d\n", __func__, errno);
			kapi_unregister_conn(conn);
			return;
		}
		bzero(conn->buf, KAPI_CONN_BUF_SIZE);
		conn->buf_malloc_len = KAPI_CONN_BUF_SIZE;
	} else {
		if (conn->buf_read == 0) {
			bzero(conn->buf, conn->buf_malloc_len);
		}
	}
	if (conn->buf_read >= sizeof(struct cmd_req_hdr)) {
		req = (struct cmd_req_hdr *)conn->buf;
		if (req->cmd_size > 0 && (conn->buf_read < (sizeof(struct cmd_req_hdr) + req->cmd_size))) {
			if (sizeof(struct cmd_req_hdr) + req->cmd_size > conn->buf_malloc_len) {
				int new_length = conn->buf_malloc_len;
				while (sizeof(struct cmd_req_hdr) + req->cmd_size > new_length) {
					new_length = 2 * new_length;
				}
				void *p = malloc(new_length);
				if (p == NULL) {
					fprintf(stderr, "%s: malloc length %d failed: %d\n", __func__, 2 * conn->buf_malloc_len, errno);
					kapi_unregister_conn(conn);
					return;
				}
				bzero(p, new_length);
				conn->buf_malloc_len = new_length;
				memcpy(p, conn->buf, conn->buf_read);
				free(conn->buf);
				conn->buf = p;
			}
			buf = (char *)conn->buf + conn->buf_read;
			left_buf_len = conn->buf_malloc_len - conn->buf_read;
			ARRAY_DEBUG("debug left_buf_len:%d, conn->buf_read: %d\n", left_buf_len, conn->buf_read);
			ret = recv(ns, buf, sizeof(struct cmd_req_hdr) + req->cmd_size - conn->buf_read, MSG_DONTWAIT);
		} else {
			/* This should not happen */
			fprintf(stderr, "%s: there is nothing to be read, ns = %d\n", __func__, conn->ns);
			conn->reqs[0].req = NULL;
			conn->buf_read = 0;
			ARRAY_DEBUG("debug kapi_handle_req conn->buf_read = 0\n");
			return;
		}
	} else {
		buf = (char *)conn->buf + conn->buf_read;
		left_buf_len = conn->buf_malloc_len - conn->buf_read;
		ARRAY_DEBUG("debug left_buf_len:%d, conn->buf_read: %d\n", left_buf_len, conn->buf_read);
		ret = recv(ns, buf, left_buf_len, MSG_DONTWAIT);
	}
	if (ret > 0) {
		conn->buf_read += ret;
		left_buf_len -= ret;
	} else if (ret == 0) {
		/* how to know the client close the connection? */
		ARRAY_DEBUG("debug kapi_unregister_conn ns :%d\n", conn->ns);
		kapi_unregister_conn(conn);
		return;
	} else if (ret == -1) {
		if (errno != EAGAIN) {
			fprintf(stderr, "%s: readv failed: %d\n", __func__, errno);
			kapi_unregister_conn(conn);
		}
		return;
	}

	ARRAY_DEBUG("debug kapi_handle_req ns :%d, ret = %d\n", ns, ret);
	if (conn->buf_read >= sizeof(struct cmd_req_hdr)) {
		req = (struct cmd_req_hdr *)conn->buf;
		if (conn->buf_read == (sizeof(struct cmd_req_hdr) + req->cmd_size)) {
			/* wrapper this request and enqueue */
			conn->reqs[0].req = req;
			ARRAY_DEBUG("debug kapi_req_enqueue\n");
			kapi_req_enqueue(conn);
			conn->buf_read = 0;
			ARRAY_DEBUG("debug kapi_handle_req conn->buf_read = 0\n");
			/* how to handle the remaining data? copy to the head? */
		} else if (conn->buf_read > (sizeof(struct cmd_req_hdr) + req->cmd_size)) {
			fprintf(stderr, "%s: error, buf_read:%d exceed the total message length:%d\n", 
				__func__, conn->buf_read, (int)(sizeof(struct cmd_req_hdr) + req->cmd_size));
			kapi_unregister_conn(conn);
			return;
		} else {
			/* not read the request completely */
			ARRAY_DEBUG("debug not read the request completely\n");
		}
	}
}
```

请求入队以后就到了atcp执行kernelapi的地方了，这里面就是包裹了一个kernelapi_input，这里才是我们的Kernel_api执行的真正部分，kernelapi_input函数最值得关注的实际上是内存和锁的操作，没有办法，你正在新建连接，忽然配置变了，搁谁都蛋疼。
```
kapi_handle_req_atcp(void)
{
	uint32_t rq_rd = kapi_queue.rq_rd;

	while (rq_rd != kapi_queue.rq_wr) {
		kernelapi_input(kapi_queue.que[rq_rd]);
		d_cmd_index = kapi_queue.que[rq_rd]->reqs[0].req->cmd_index;
		kapi_queue.que[rq_rd]->reqs[0].req = NULL; /* Don't use this any more */
		rq_rd = (rq_rd + 1)%KAPI_RQ_LEN;
	}

	kapi_queue.rq_rd = rq_rd;
}
```

#### kernelapi的执行
现在我们看看`kernelapi_input`函数，`kernelapi_input`本质上是调用index对应的函数，也就是说我们写的函数虽然是`ssl_get_special_user_all_host_config_kern`，但atcp进程真正调用的是一个index对应于`ssl_get_special_user_all_host_config_kern`的函数，这个函数实际上是`kernel_ssl_get_special_user_all_host_config_kern`(没想到吧.jpg)。好，我们看看`kernelapi_input`函数具体流程。

`kernelapi_input`函数首先获取backend和atcp的之间的连接（此时，读取的数据已经填好了），拿到请求的kernelapi函数的index，也拿到rsp，初始化rsp的两个长度都为0。获取该kernelapi函数的lock，看看需要锁哪种lock。这个地方是出于数据一致性的要求，比方说显示当前所有的连接，`atcp_sync_stop_broadcast`会把全局变量`click_kernelapi_want_lock`赋值为一，然后将自己线程的`atcp_sync_stop_flag[curatcp]`赋值为一，然后调用`tsleep(atcpwait[curatcp], 0, "atcpsync", 1);`，管理线程会一直等待该资源就绪从而被wakeup，同时等待其他线程的终止也就是其他线程的`atcp_sync_stop_flag[i]`为1，注意，这里管理线程只有对于`atcp_sync_stop_flag[i]`的读，没有同时写操作，因此没有竞争！而其他线程检查函数会调用`atcp_sync_stop_check`这个函数竞争atcpwait[another_curatcp]，也没发生资源的竞争。所以不需要锁！很好，那么问题来了，其他线程修改完了`atcp_sync_stop_flag[curatcp]`之后，哪里唤醒管理线程呢？实际上是没有地方唤醒的，这里就是不断的等待。

代码运行完成后
会保存outptr到conn->reqs[0].d_ptr里，这个东西就是我们最终要写出去数据。这里有个问题是，只可以写一层数据，不可以写指向指针的指针，因为内存不共享，写了会发生内存越界，直接coredump。而且这里只能写一个都出去的指针，也很糟糕。
```
	int kernelapi_input(struct kapi_conn *conn)
{
	int kapi_sync_atcp;
	int start_ticks=0, wait_ticks=0;
	struct cmd_req_hdr  *req;
	struct cmd_rsp_hdr  *rsp = NULL;
	int cmd_index, cmd_size;
	void *outptr;
	int32_t outsize = 0;
	uint16_t lock_flags = CMD_KAPI_LOCK_INVALID;

	/* 
	 * Store these sysctl values into local variables in case they
	 * are changed partway through processing. The new values will
	 * be picked up on the next run.
	 */
	kapi_sync_atcp = kapi_sync_atcp_on;

	req = conn->reqs[0].req;
	cmd_index = req->cmd_index;
	cmd_size = req->cmd_size;


	rsp = &conn->reqs[0].rsp_hdr;
	rsp->p_length= 0;
	rsp->d_length= 0;

	if (cmd_index > 0 && cmd_index <= MAX_CMD_INDEX) {
		lock_flags = cli_cmd_lock[cmd_index-1];
		/* ATCP synchronize configuration mechanism - batch mode */
		if ((lock_flags & CMD_KAPI_LOCK) && kapi_sync_batch_fl) {
			if (!kapi_sync_atcp_fl) {
				start_ticks = ticks;
				atcp_sync_stop_broadcast();
				kapi_sync_atcp_fl = 1;
				wait_ticks = ticks - start_ticks;

				/* check for new min tick value */
				if (wait_ticks < kapi_batch_sync_wait_min) {
					kapi_batch_sync_wait_min = wait_ticks;
				}
				/* check for new max tick value */
				if (wait_ticks > kapi_batch_sync_wait_max) {
					kapi_batch_sync_wait_max = wait_ticks;
				}
				/* calculate new avg tick value */
				kapi_batch_sync_wait_avg *= kapi_batch_sync_count;
				kapi_batch_sync_wait_avg += wait_ticks;
				kapi_batch_sync_count++;
				kapi_batch_sync_wait_avg /= kapi_batch_sync_count;
			}
			/* do not need to lock again in batch mode */
			lock_flags = CMD_KAPI_NOLOCK;
		}
		switch (lock_flags & CMD_KAPI_LOCK_MASK) {
		case CMD_KAPI_LOCK:
			/* ATCP synchronize configuration mechanism */
			if (kapi_sync_atcp && (lock_flags & (CMD_KAPI_LOCK_ATCP | CMD_KAPI_LOCK_UPROXY))) {
				/* only lock ATCP if CMD_KAPI_LOCK_ATCP flag set,
				 * or if CMD_KAPI_LOCK_UPROXY flag set after merge uproxy to atcp thread in ustack. */
				start_ticks = ticks;
				atcp_sync_stop_broadcast();
				wait_ticks = ticks - start_ticks;
				/* check for new min tick value */
				if (wait_ticks < kapi_atcp_sync_wait_min) {
					kapi_atcp_sync_wait_min = wait_ticks;
				}
				/* check for new max tick value */
				if (wait_ticks > kapi_atcp_sync_wait_max) {
					kapi_atcp_sync_wait_max = wait_ticks;
				}
				/* calculate new avg tick value */
				kapi_atcp_sync_wait_avg *= kapi_atcp_sync_count;
				kapi_atcp_sync_wait_avg += wait_ticks;
				kapi_atcp_sync_count++;
				kapi_atcp_sync_wait_avg /= kapi_atcp_sync_count;
				/* set flag, indicating in atcp lock mode */
				kapi_sync_atcp_fl = 1;
			}

			if(kapi_debug){
				englog(ENGLOG_CLICKTCP, AMP_KDB_LOG, "Calling kernelapi %d\n", cmd_index);
			}
			cli_cmd_index = cmd_index;
			cli_cmd_start = ticks;
			cli_cmd_runtime = -1;
			rsp->status = (*app_cmd_switch[cmd_index-1])((void *)(req), conn,
			                                             &outptr, &outsize);
			cli_cmd_runtime = ticks - cli_cmd_start;
			kapi_cmd_stats[cmd_index-1].count++;
			kapi_cmd_stats[cmd_index-1].ticks += (uint16_t)cli_cmd_runtime;

			if (kapi_sync_atcp && (lock_flags & (CMD_KAPI_LOCK_ATCP | CMD_KAPI_LOCK_UPROXY))) {
				/* if ATCP was locked, unlock it */
				atcp_sync_resume_notify();
				/* unset flag, indicating not in atcp lock mode */
				kapi_sync_atcp_fl = 0;
			}
			break;
		case CMD_KAPI_NOLOCK:
			cli_cmd_index = cmd_index;
			cli_cmd_start = ticks;
			cli_cmd_runtime = -1;
			rsp->status = (*app_cmd_switch[cmd_index-1])((void *)(req), conn,
			                                             &outptr, &outsize);
			cli_cmd_runtime = ticks - cli_cmd_start;
			break;
		default:
			KASSERT(0, ("unknown lock flags %08X for cmd %d\n",
			            lock_flags, cmd_index));
			break;
		}
	} else {
		dbg_printf("invalid cmd %d\n", cmd_index);
		rsp->status = -1;
	}

	if (outsize) {
		rsp->d_length = outsize;
		conn->reqs[0].d_ptr = outptr;
	}
	rsp->p_length = conn->reqs[0].p_used_len;

	return 0;
}
```

#### kernelapi的rsp回写
如果kernelapi不需要获取输出内容，那么回写很简单，但如果需要输出内容，那么就需要把内容写到socket，让客户端读出来。上一步末尾，我们看到了要读出的内容写到了outptr的地址上，此时就需要把内容写到socket对应的缓冲区，让客户读取结果。下面我们看看`kapi_handle_rsp`函数，先尝试写一个cmd_rsp_hdr长度的数据，相当于能够让客户端清楚多少数据要读取。之后如果有要读出的内容，那就调用函数`kapi_send_d_data`尝试发送，在`kapi_send_d_data`里，可能数据过多，缓冲区满了，就将`conn->reqs[0].d_left_flag = 1;`，从而调用`kapi_register_write_event(conn)`像`kapi_epoll_fd`注册`EPOLLOUT|EPOLLET`等待数据可写。数据可写之后就在`management_epoll_wait`被唤醒，然后尝试继续发送，直到发送结束，关闭SOCKET。

```
	void
kapi_handle_rsp(void)
{
	int i = 0;
	int ret = 0;
	uint32_t cleanup_idx = kapi_queue.rq_cleanup;

	while(cleanup_idx != kapi_queue.rq_rd) {
		struct kapi_conn *conn = kapi_queue.que[cleanup_idx];
		int ns = conn->ns;
		struct cmd_rsp_hdr *rsp_hdr = &conn->reqs[0].rsp_hdr;

		ARRAY_DEBUG("debug kapi_handle_rsp, status = %d, p_length = %d, d_length = %d\n", 
			rsp_hdr->status, rsp_hdr->p_length, rsp_hdr->d_length);

		int write_bytes = send(ns, rsp_hdr, sizeof(struct cmd_rsp_hdr), 0);
		if (write_bytes != sizeof(struct cmd_rsp_hdr)) {
			fprintf(stderr, "%s: write_bytes(%d) != sizeof(struct cmd_rsp_hdr)(%ld): %d\n", __func__, write_bytes, sizeof(struct cmd_rsp_hdr), errno);
		}
		if (write_bytes == -1) {
			fprintf(stderr, "%s: send rsp_hdr failed: %d\n", __func__, errno);
			kapi_unregister_conn(conn);
			goto LOOP_TAIL;
		}

		if (rsp_hdr->p_length) {
			ret = kapi_send_p_data(conn);
			if (ret == -1) {
				goto LOOP_TAIL;
			}
			if (conn->reqs[0].p_left_flag) {
				kapi_register_write_event(conn);
				/* if did't finish send p_data, need go to next loop directly, don't try to send d_data*/
				goto LOOP_TAIL;
			}
		}

		if (rsp_hdr->d_length) {
			ret = kapi_send_d_data(conn);
			if (ret == -1) {
				goto LOOP_TAIL;
			}

			if (conn->reqs[0].d_left_flag) {
				kapi_register_write_event(conn);
			}
		}

LOOP_TAIL:
		cleanup_idx = (cleanup_idx + 1) % KAPI_RQ_LEN;
	}
	kapi_queue.rq_cleanup = cleanup_idx;
}

static int
kapi_send_d_data(struct kapi_conn *conn)
{
	struct cmd_rsp_hdr *rsp_hdr = &conn->reqs[0].rsp_hdr;
	int write_bytes = 0;
	int ns = conn->ns;

	if (rsp_hdr->d_length - conn->reqs[0].d_have_send_len) {
		write_bytes = kapi_send_all_data(ns, conn->reqs[0].d_ptr + conn->reqs[0].d_have_send_len, 
			rsp_hdr->d_length - conn->reqs[0].d_have_send_len);
		if (write_bytes > 0) {
			conn->reqs[0].d_have_send_len += write_bytes;
		}
		if (conn->reqs[0].d_have_send_len < rsp_hdr->d_length) {
			ARRAY_DEBUG("%s: write_bytes(%d) , rsp_hdr->d_length(%d), conn->reqs[0].d_have_send_len(%d), errno(%d)\n", 
				__func__, write_bytes, rsp_hdr->d_length, conn->reqs[0].d_have_send_len, errno);
			if (write_bytes == -1) {
				kapi_unregister_conn(conn);
				return -1;
			} else {
				conn->reqs[0].d_left_flag = 1;
				return 0;
			}
		}
		conn->reqs[0].d_left_flag = 0;
		conn->reqs[0].d_have_send_len = 0;
		ARRAY_DEBUG("debug %s, d_length = %d, write_bytes: %d\n", __func__, rsp_hdr->d_length, write_bytes);
		if (conn->reqs[0].d_ptr) {
			free(conn->reqs[0].d_ptr);
			conn->reqs[0].d_ptr = NULL;
		}
		rsp_hdr->d_length = 0;
	}
	return 1;
}

static int
kapi_send_all_data(int sockfd, char *buf, uint32_t len)
{
	int write_bytes = 0;
	int ret = 0;
	while (len - write_bytes > 0) {
		ret = send(sockfd, buf + write_bytes, len - write_bytes, 0);
		if (ret > 0) {
			write_bytes += ret;
		} else if (ret == -1) {
			if (errno != EAGAIN) {
				fprintf(stderr, "%s: send failed, ret: %d, socket: %d, errno: %d\n", __func__, ret, sockfd, errno);
				return -1;
			} else {
				ARRAY_DEBUG("%s: send failed, ret: %d, socket: %d, errno: %d\n", __func__, ret, sockfd, errno);
				break;
			}
		} else {
			fprintf(stderr, "%s: send failed, ret: %d, socket: %d, errno: %d\n", __func__, ret, sockfd, errno);
		}
	}
	return write_bytes;
}
```
#### kapi queue的代码
这部分内容实质上是一个无锁环的内容，一个全局的无锁环`struct kapi_ring_queue *rq = &kapi_queue;`，无锁环的结构为
```c
	struct kapi_ring_queue {
	uint32_t rq_wr;				/* Write index */
	uint32_t rq_rd;				/* Read index */
	uint32_t rq_cleanup;			/* index in the last cleanup */
	struct kapi_conn *que[KAPI_RQ_LEN];
};
```

核心代码只有两个，把请求放到环里，把请求从环里拿出来。注意，这里如果有请求放到无锁环里，那么必然会立刻就把数据出环调用掉。不可能出现环满了，两个撞上的情况。只有管理线程读写，也不会有其他线程无锁环操作的问题。
```c
static void
kapi_req_enqueue(struct kapi_conn *conn)
{
	struct kapi_ring_queue *rq = &kapi_queue;
	uint32_t rq_wr = rq->rq_wr;
	uint64_t count = 1;

	if (((rq_wr+1)%KAPI_RQ_LEN) == rq->rq_rd) {
		printf("%s: kapi_queue is full\n", __func__);
		return;
	}
	/* since never be full, and queue len is 2**n, we be done in 2 line */
	rq->que[rq_wr] = conn;
	rq->rq_wr = (rq_wr+1)%KAPI_RQ_LEN;
}

static void
kapi_handle_req_atcp(void)
{
	uint32_t rq_rd = kapi_queue.rq_rd;

	while (rq_rd != kapi_queue.rq_wr) {
		kernelapi_input(kapi_queue.que[rq_rd]);
		d_cmd_index = kapi_queue.que[rq_rd]->reqs[0].req->cmd_index;
		kapi_queue.que[rq_rd]->reqs[0].req = NULL; /* Don't use this any more */
		rq_rd = (rq_rd + 1)%KAPI_RQ_LEN;
	}

	kapi_queue.rq_rd = rq_rd;
}
```

### 总结
代码分析的差不多，可以看到一个小小的kernelapi代码涉及到了epoll，线程等待，socket通信，无锁环等多方面内容。下面我们再回顾+提几个新问题
+ backend进程和atcp进程内存不共享，怎么获得一块指针指向内容呢？尽管内存共享，但是我们通过socket把读写的数据写到指向的地址上，这里隐藏一个问题是，backend进程和atcp进程都malloc一大块内存，如果内存不足可能直接就崩溃了。
+ 为什么要使用rsp_hdr和req_hdr？在unix域socket下，我们如果像知道发送的数据总长度就必须使用hdr来标记长度。
+ kernelapi的实现优劣？优点时实现很简单，缺点包括1描述符数量最多才255个，没必要用epoll2输出内存的情况下涉及到大量的内存申请，可能失败3只能输出一块内存，格式过于复杂4一条kernelapi完成就断开，对于性能是个累赘(实际上还好)5没有处理客户端socket描述符EPOLLHUP等错误情况，代码考虑不足



## 结尾的闲言碎语
写到这里差不多就可以结束了，就不多说了。
![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)
