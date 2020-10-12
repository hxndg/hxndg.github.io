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

上面看到末尾发现本质上是包裹了一层cmd_api，因此还是得看看cmd_api的实现。这个函数实际上非常简单就是穿个命令过去，然后去去执行的结果，我们重点看linux的实现版本，首先申明一个sock，如果不适用全局的cmd_kern_sd描述符就创建新的socket描述符，之后获取socket地址，socket地址是以文件方式保存，并建立连接。这里有个变量`multithread_enable`，这东西负责干什么？实际上是负责是否新创建socket描述符还是使用全局的cmd_kern_sd描述符，我们这里认为只是用cmd_kern_sd描述符。

接着向下看，向描述符写入cmd，判断是否写成功，之后读取rsp。如果rsp的p_length大于0，那么还需要读取并打印p_length，如果d_length大于0，那么还需要读取出d_length。这两个结构体的作用是什么？我们等到下面分析atcp中执行操作的时候再说。

好，总结下，这里藏着两个问题
+ 我们如果传入了诸如outdata的东西的话，backend进程和atcp进程并不共享内存地址，如何传递结构呢？
+ rsp结构体里面的d_length和p_length到底有什么用呢？

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

kernelapi真正的执行部分需要有一些前置的东西，这里我直接简单说说带过了，因为涉及到我们系统的设计。管理线程原本是freebsd上的内核线程，现在移植到Linux变成用户态的一个线程，该线程负责kernelapi通信，从而对atcp进程进行操作。管理线程首先需要初始化监听socket等待客户端的连接，函数是`management_epoll_init`，创建监听描述符绑之后，调用epoll来添加读事件(EPOLLIN)，该函数的末尾有一段:
```
	bzero(&kapi_conns[0], sizeof(kapi_conns));
	for (int i = 0; i < KAPI_MAX_CONNS; i++) {
		kapi_conns[i].ns = -1;
		kapi_conns[i].reqs[0].p_iov_idx = -1;
	}
```
这部分实际上对应于建立的连接，即同时最多多少个backend进程和管理线程通信。简单而言，一开始没有客户连接的时候，epoll只监听listen_fd监听描述符，如果有客户端连接了，那么新创建描述符(new socket，ns)并存储到kapi_conn[i]的ns中，并添加到epoll中监听，并设置该socket为非阻塞socket。新添加ns和等待ns/listen_fd的操作就是在`management_epoll_wait`中实现的。

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

`management_epoll_wait`函数负责监听描述符，如果是listen_fd就新创建连接描述符，如果是已经创建出来的描述符就监听读事件。这里对listen_fd的处理不多说，我们看看读事件的处理。backend进程通信的时候我们知道实际上是向全局描述符kern_cmd_sd写信息，那么此时调用的实际上是`kapi_handle_req`函数，该函数负责读取监听到的数据。

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









## 结尾的闲言碎语
写到这里差不多就可以结束了，就不多说了。
![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)
