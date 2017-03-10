# 概要
ARM Trusted Firmware实现了一个可信引导过程，并给操作系统提供运行环境（SMC调用服务）。本文分析基于ARM Trusted Firmware - version 1.3，代码托管与[GitHub](https://github.com/ARM-software/arm-trusted-firmware)。

引导过程分为多步，每一步为一个独立的程序。上一步负责验证下一步的程序（可能是多个，BL2会加载认证其他所有的bootloader），并把控制权交给下一级程序。每一级程序需要为下一级程序提供运行环境。最终引导一个通用的bootloader，并实现SMC调用服务。

本文主要分析AArch64相关部分代码

# 安全相关
## auth_common
此模块主要用于声明一些公共的数据结构，这些数据结构主要用与描述认证的方式`auth_method_desc_s`和认证的参数`auth_param_desc_t`。

认证参数`auth_param_desc_t`是由两个基本结构组成：类型`auth_param_type_desc_t`和数据`auth_param_data_desc_t`。

并定义了两个宏用于快速实现一个类型和数据
```c
#define AUTH_PARAM_TYPE_DESC(_type, _cookie) \
	{ \
		.type = _type, \
		.cookie = (void *)_cookie \
	}

#define AUTH_PARAM_DATA_DESC(_ptr, _len) \
	{ \
		.ptr = (void *)_ptr, \
		.len = (unsigned int)_len \
	}
```

## crypto_mod
此模块主要实现一个框架，并非具体实现。主要用于校验哈希和签名。

此框架主要基于一个结构体，结构体定义如下:
```c
typedef struct crypto_lib_desc_s {
	const char *name;/* 名称， */

	void (*init)(void);/* 初始化方法 */

	/* 校验签名的方法 */
	int (*verify_signature)(
                void *data_ptr, unsigned int data_len,   /* 要签名的数据 */
				void *sig_ptr, unsigned int sig_len,     /* 签名 */
				void *sig_alg, unsigned int sig_alg_len, /* 签名算法 */
				void *pk_ptr, unsigned int pk_len);      /* 公钥 */

	/* 校验哈希的方法 */
	int (*verify_hash)(
               void *data_ptr, unsigned int data_len,       /* 要计算哈希的数据 */
			   void *digest_info_ptr, unsigned int digest_info_len);/* 哈希值 */
} crypto_lib_desc_t;
```

通过`REGISTER_CRYPTO_LIB`宏实现一个名为`crypto_lib_desc`类型为`crypto_lib_desc_t`结构体。宏实现如下:
```c
#define REGISTER_CRYPTO_LIB(_name, _init, _verify_signature, _verify_hash) \
	const crypto_lib_desc_t crypto_lib_desc = { \
		.name = _name, \
		.init = _init, \
		.verify_signature = _verify_signature, \
		.verify_hash = _verify_hash \
	}
```

此模块通过操作crypto_lib_desc变量，实现模块初始化、校验签名、校验哈希。函数声明如下：
```c
/* 模块初始化 */
void crypto_mod_init(void);

/* 校验签名 */
int crypto_mod_verify_signature(void *data_ptr, unsigned int data_len,
				void *sig_ptr, unsigned int sig_len,
				void *sig_alg, unsigned int sig_alg_len,
				void *pk_ptr, unsigned int pk_len);
/* 校验哈希值 */
int crypto_mod_verify_hash(void *data_ptr, unsigned int data_len,
			   void *digest_info_ptr, unsigned int digest_info_len);
```

## img_parser_mod
此模块实现一个框架，并非具体实现。主要用于校验镜像完整性，以及从镜像中提取内容。

镜像被分为以下几种类型
```c
typedef enum img_type_enum {
	IMG_RAW,			/* Binary image */
	IMG_PLAT,			/* Platform specific format */
	IMG_CERT,			/* X509v3 certificate */
	IMG_MAX_TYPES,
} img_type_t;
```

镜像解析器被包装成一个结构体，定义如下
```c
typedef struct img_parser_lib_desc_s {
	img_type_t img_type;   /* 镜像解析器操作的镜像的类型 */
	const char *name;      /* 镜像解析器的名字 */

	void (*init)(void);    /* 初始化镜像解析器 */

    /* 校验镜像完整性 */
	int (*check_integrity)(void *img, unsigned int img_len);

    /* 从镜像中提取内容 */
	int (*get_auth_param)(
            const auth_param_type_desc_t *type_desc,  /* 要提取的内容的描述信息 */
			void *img, unsigned int img_len,          /* 镜像内容 */
			void **param, unsigned int *param_len);   /* 输出：提取的内容 */
} img_parser_lib_desc_t;
```

通过宏`REGISTER_IMG_PARSER_LIB`实现一个解析器，定义如下：
```c
#define REGISTER_IMG_PARSER_LIB(_type, _name, _init, _check_int, _get_param) \
	static const img_parser_lib_desc_t __img_parser_lib_desc_##_type \
	__section(".img_parser_lib_descs") __used = { \
		.img_type = _type, \
		.name = _name, \
		.init = _init, \
		.check_integrity = _check_int, \
		.get_auth_param = _get_param \
	}
```
多个解析器被放在同一个段`.img_parser_lib_descs`。配合链接脚本此段开始符号为`__PARSER_LIB_DESCS_START__`，结束符号为`__PARSER_LIB_DESCS_END__`。两个符号可以看作解析器数组的起始符号和结束符号。

`img_parser_init`通过遍历解析器数组，调用解析器的初始化函数`init`，对解析器进行初始化。并根据`img_type`构建索引（通过镜像类型找到解析器下标）。并防止有多个相同类型的镜像解析器存在。

`img_parser_check_integrity`校验镜像完整性

`img_parser_get_auth_param`从镜像中提取需要的内容

## auth_mod
auth_mod实现了一个校验镜像的模型，此模型通过结构体`auth_img_desc_t`描述。
```c
typedef struct auth_img_desc_s {
	/* 镜像的ID,标志是哪一个镜像 */
	unsigned int img_id;

	/* 镜像类型(Binary、证书等) */
	img_type_t img_type;

	/* 父镜像，保存了认证当前镜像的公钥、哈希等 */
	const struct auth_img_desc_s *parent;

	/* 认证当前镜像的方法 */
	auth_method_desc_t img_auth_methods[AUTH_METHOD_NUM];

	/* 用于校验子镜像的公钥、哈希等 */
	auth_param_desc_t authenticated_data[COT_MAX_VERIFIED_PARAMS];
} auth_img_desc_t;
```

并定义一个宏`REGISTER_COT`，用于注册`auth_img_desc_t`数组
```c
#define REGISTER_COT(_cot) \
	const auth_img_desc_t *const cot_desc_ptr = \
			(const auth_img_desc_t *const)&_cot[0]; \
	unsigned int auth_img_flags[sizeof(_cot)/sizeof(_cot[0])]
```

`auth_mod_verify_img`通过`img_id`访问`cot_desc_ptr`数组，找到对应的镜像描述符`auth_method_desc_t`，即可知道当前镜像的认证方式，访问父节点找到签名的公钥或哈希，即可认证当前镜像是否合法。在认证完当前镜像后，从镜像中解析出公钥哈希等放入当前的镜像描述符中，便于对下一级镜像校验
```c
int auth_mod_verify_img(unsigned int img_id,
			void *img_ptr,
			unsigned int img_len)
{
	const auth_img_desc_t *img_desc = NULL;
	const auth_method_desc_t *auth_method = NULL;
	void *param_ptr;
	unsigned int param_len;
	int rc, i;

	/* 根据img_id获取镜像描述符 */
	img_desc = &cot_desc_ptr[img_id];

	/* 校验镜像完整性 */
	rc = img_parser_check_integrity(img_desc->img_type, img_ptr, img_len);
	return_if_error(rc);

	/* 根据镜像描述符的仍正方式对镜像进行认证 */
	for (i = 0 ; i < AUTH_METHOD_NUM ; i++) {
		auth_method = &img_desc->img_auth_methods[i];
		switch (auth_method->type) {
		case AUTH_METHOD_NONE:/* 不需要认证 */
			rc = 0;
			break;
		case AUTH_METHOD_HASH:/* 哈希认证 */
			rc = auth_hash(&auth_method->param.hash,
					img_desc, img_ptr, img_len);
			break;
		case AUTH_METHOD_SIG:/* 签名认证 */
			rc = auth_signature(&auth_method->param.sig,
					img_desc, img_ptr, img_len);
			break;
		case AUTH_METHOD_NV_CTR:/* Non-Volatile counter认证？ */
			rc = auth_nvctr(&auth_method->param.nv_ctr,
					img_desc, img_ptr, img_len);
			break;
		default:
			/* 未知认证类型，报错 */
			rc = 1;
			break;
		}
		return_if_error(rc);
	}

	/* 从镜像中解析出公钥哈希等，以便对下一级镜像进行认证 */
	for (i = 0 ; i < COT_MAX_VERIFIED_PARAMS ; i++) {
		if (img_desc->authenticated_data[i].type_desc == NULL) {
			continue;
		}

		/* 通过镜像解析器从镜像中提取内容 */
		rc = img_parser_get_auth_param(img_desc->img_type,
				img_desc->authenticated_data[i].type_desc,
				img_ptr, img_len, &param_ptr, &param_len);
		return_if_error(rc);

		/* 异常检查
		   防止从镜像中解析出的数据字节数大于镜像描述符中的字节数
		   出现内存访问溢出 */
		if (param_len > img_desc->authenticated_data[i].data.len) {
			return 1;
		}

		/* 把解析出的内容拷贝到镜像描述符中，便于解析下一级BL */
		memcpy((void *)img_desc->authenticated_data[i].data.ptr,
				(void *)param_ptr, param_len);
	}

	/* 标记镜像以认证过 */
	auth_img_flags[img_desc->img_id] |= IMG_FLAG_AUTHENTICATED;

	return 0;
}
```


# 设备抽象

为了便于移植ARM Trusted Firmware提供了一种设备抽象机制，这里主要用来抽象镜像读取操作，以便加载镜像到系统内存中。

## 主要数据结构
### io_dev_info_t
io_dev_info_t对应一个设备描述符
```c
/* 设备操作相关的函数句柄 */
typedef struct io_dev_funcs {
	io_type_t (*type)(void);
	int (*open)(io_dev_info_t *dev_info, const uintptr_t spec,
			io_entity_t *entity);
	int (*seek)(io_entity_t *entity, int mode, ssize_t offset);
	int (*size)(io_entity_t *entity, size_t *length);
	int (*read)(io_entity_t *entity, uintptr_t buffer, size_t length,
			size_t *length_read);
	int (*write)(io_entity_t *entity, const uintptr_t buffer,
			size_t length, size_t *length_written);
	int (*close)(io_entity_t *entity);
	int (*dev_init)(io_dev_info_t *dev_info, const uintptr_t init_params);
	int (*dev_close)(io_dev_info_t *dev_info);
} io_dev_funcs_t;

typedef struct io_dev_info {
	const struct io_dev_funcs *funcs;/* 设备操作相关的函数句柄 */
	uintptr_t info;/* 设备信息（比如设备地址） */
} io_dev_info_t;
```

### io_entity_t
io_entity_t对应一个文件描述符
```c
typedef struct io_entity {
	struct io_dev_info *dev_handle;/* 设备信息 */
	uintptr_t info;/* 文件有关信息 */
} io_entity_t;
```

### io_dev_connector_t
io_dev_connector_t对应一个驱动
```c
typedef struct io_dev_connector {
	/* dev_open opens a connection to a particular device driver */
	int (*dev_open)(const uintptr_t dev_spec, io_dev_info_t **dev_info);
} io_dev_connector_t;
```

### 关联
系统通过**dev_count**、**devices**，来维护一个注册的设备列表
```c
/* 注册一个设备此变量加1 */
static unsigned int dev_count;
/* 指向设备的指针数组，注册的设备会被添加进来 */
static const io_dev_info_t *devices[MAX_IO_DEVICES];
```

系统通过**entity_pool**、**entity_map**、**entity_count**来维护已经打开的文件
```c
/* 实际的文件句柄 */
static io_entity_t entity_pool[MAX_IO_HANDLES];

/* 指针数组用于标记entity_pool中的文件句柄是否空闲 */
static io_entity_t *entity_map[MAX_IO_HANDLES];

/* 记录当前打开的文件句柄书目 */
static unsigned int entity_count;
```

## 接口
### 打开设备
```c
int io_dev_open(const struct io_dev_connector *dev_con,/* 驱动描述符 */
		const uintptr_t dev_spec,/* 用于指定打开那个设备，与具体驱动有关 */
		uintptr_t *dev_handle);/* 返回值：设备句柄（io_dev_info_t） */
```

### 设备初始化
```c
/*
 * dev_handle设备句柄
 * init_params设备初始化的参数（特定设备相关）
 */
int io_dev_init(uintptr_t dev_handle, const uintptr_t init_params);
```

### 关闭设备
```c
int io_dev_close(uintptr_t dev_handle);
```

### 文件相关操作
```c
/*
 * 打开文件
 * dev_handle设备句柄
 * spec文件信息（特定设备相关）
 * handle文件句柄
 */
int io_open(uintptr_t dev_handle, const uintptr_t spec, uintptr_t *handle);

/*
 * 移动文件指针
 * handle文件句柄
 * mode移动文件指针的相对初始地址
 * offset移动文件指针的相对地址
 */
int io_seek(uintptr_t handle, io_seek_mode_t mode, ssize_t offset);

/*
 * 获取文件大小
 * handle文件句柄
 * length返回值，文件大小
 */
int io_size(uintptr_t handle, size_t *length);

/*
 * 读文件
 * handle文件句柄
 * buffer是读出缓存的地址
 * length是读出缓存的大小
 * length_read返回值，实际读出的大小
 */
int io_read(uintptr_t handle, uintptr_t buffer, size_t length,
		size_t *length_read);

/*
 * 写文件
 * handle文件句柄
 * buffer要写入文件的缓存的起始地址
 * length要写如文件的缓存的大小
 * length_written返回值，实际写入的大小
 */
int io_write(uintptr_t handle, const uintptr_t buffer, size_t length,
		size_t *length_written);

/* 关闭文件，handle文件句柄 */
int io_close(uintptr_t handle);
```

# EL3初始化

EL3初始化通过宏**el3_entrypoint_common**实现，此宏位于`include/common/aarch64/el3_common_macros.S`中

此宏有6个参数分别控制初始化的内容

```assembly
.macro el3_entrypoint_common					\
		_set_endian, _warm_boot_mailbox, _secondary_cold_boot,	\
		_init_memory, _init_c_runtime, _exception_vectors
```

## 大小端设置

```assembly
.if \_set_endian
	/* sctlr_el3.ee = 0 EL3设置为小端 */
	mrs	x0, sctlr_el3
	bic	x0, x0, #SCTLR_EE_BIT
	msr	sctlr_el3, x0
	isb
.endif
```

## 热启动处理
此部分，用在热启动时跳过初始化代码
```assembly
.if \_warm_boot_mailbox
	/* 冷启动返回0，
	   热启动返回PLAT_ARM_TRUSTED_MAILBOX_BASE处的值 */
	bl	plat_get_my_entrypoint
	cbz	x0, do_cold_boot
	/* 热启动跳转到PLAT_ARM_TRUSTED_MAILBOX_BASE内存处指定的地址继续执行 */
	br	x0

/* 冷启动继续执行 */
do_cold_boot:
.endif
```

## 次处理器处理

为了便于处理，在初始化时会让其他处理器暂停（执行while循环）

```assembly
.if \_secondary_cold_boot
	bl	plat_is_my_cpu_primary		/* 判断当前CPU是不是主CPU */
	cbnz	w0, do_primary_cold_boot

	/* 非主CPU暂停，
	   循环等待PLAT_ARM_TRUSTED_MAILBOX_BASE出出现地址，跳转 */
	bl	plat_secondary_cold_boot_setup
	/* 此处不会执行 */
	bl	el3_panic

/* 主处理器继续 */
do_primary_cold_boot:
.endif
```

## 内存初始化

此部分与具体硬件相关，需要具体平台实现**void platform_mem_init(void)**

```assembly
.if \_init_memory
	bl	platform_mem_init
.endif 
```

## C运行环境初始化

```assembly
	.if \_init_c_runtime
#ifdef IMAGE_BL31
		/* 初始化cache */
		adr	x0, __RW_START__
		adr	x1, __RW_END__
		sub	x1, x1, x0
		bl	inv_dcache_range
#endif /* IMAGE_BL31 */

		/* bss内存初始化为0 */
		ldr	x0, =__BSS_START__
		ldr	x1, =__BSS_SIZE__
		bl	zeromem16

#if USE_COHERENT_MEM
		/* 多个内核共享的内存，初始化为0 */
		ldr	x0, =__COHERENT_RAM_START__
		ldr	x1, =__COHERENT_RAM_UNALIGNED_SIZE__
		bl	zeromem16
#endif

#ifdef IMAGE_BL1
		/* data段初始化 */
		ldr	x0, =__DATA_RAM_START__
		ldr	x1, =__DATA_ROM_START__
		ldr	x2, =__DATA_SIZE__
		bl	memcpy16
#endif
	.endif
```

## 栈初始化

使用SP_EL0作为C语言的堆栈指针，此处有单内核堆栈初始化与多内核堆栈初始化，这里只关注多内核堆栈初始化。这样每个内核会获取到一个不同的堆栈指针。

```assembly
msr	spsel, #0 /* 切换当前SP指针为SP_EL0 */
bl	plat_set_my_stack

func plat_set_my_stack
	mov	x9, x30 /* 保护LR下面要调用函数 */
	bl 	plat_get_my_stack
	mov	sp, x0/* 修改SP */
	ret	x9/* 返回 */
endfunc plat_set_my_stack

func plat_get_my_stack
	mrs	x0, mpidr_el1 /* 读取CPU标示 */
	b	platform_get_stack
endfunc plat_get_my_stack

func plat_get_my_stack
	mov	x10, x30 /* 保护LR下面要调用函数 */
	get_my_mp_stack platform_normal_stacks, PLATFORM_STACK_SIZE
	ret	x10/* 返回 */
endfunc plat_get_my_stack

/* 
 * _name为栈顶地址
 * _size一个内核的栈的大小
 * 栈地址 = 栈顶地址 + （内核编号 + 1）* 单个内核栈大小
 */
.macro get_my_mp_stack _name, _size
	bl  plat_my_core_pos /* 获取处理器编号 */
	ldr x2, =(\_name + \_size)/* 起始地址，栈向低地址增长 */
	mov x1, #\_size /* 一个内核栈的大小 */
	madd x0, x0, x1, x2 /* x0=x0*x1+x2 */
.endm
```

## 异常向量初始化

```assembly
	/* 使能指令缓存、对齐检查 初始化异常向量 */
	el3_arch_init_common \_exception_vectors
	
.macro el3_arch_init_common _exception_vectors
	/* 使能指令缓存、对齐检查 */
	mov	x1, #(SCTLR_I_BIT | SCTLR_A_BIT | SCTLR_SA_BIT)
	mrs	x0, sctlr_el3
	orr	x0, x0, x1
	msr	sctlr_el3, x0
	isb

#ifdef IMAGE_BL31

	bl	init_cpu_data_ptr
#endif /* IMAGE_BL31 */

	/* 异常向量设置 */
	adr	x0, \_exception_vectors
	msr	vbar_el3, x0
	isb
	
	/* 不允许从非安全存储器获取安全状态指令
       外部中止和SError路由到EL3*/
	mov	x0, #(SCR_RES1_BITS | SCR_EA_BIT | SCR_SIF_BIT)
	msr	scr_el3, x0

	/* 复位mdcr_el3寄存器 */
	msr	mdcr_el3, xzr

	/* 使能外部中止和SError异常 */
	msr	daifclr, #DAIF_ABT_BIT

	/* ---------------------------------------------------------------------
	 * The initial state of the Architectural feature trap register
	 * (CPTR_EL3) is unknown and it must be set to a known state. All
	 * feature traps are disabled. Some bits in this register are marked as
	 * reserved and should not be modified.
	 *
	 * CPTR_EL3.TCPAC: This causes a direct access to the CPACR_EL1 from EL1
	 *  or the CPTR_EL2 from EL2 to trap to EL3 unless it is trapped at EL2.
	 *
	 * CPTR_EL3.TTA: This causes access to the Trace functionality to trap
	 *  to EL3 when executed from EL0, EL1, EL2, or EL3. If system register
	 *  access to trace functionality is not supported, this bit is RES0.
	 *
	 * CPTR_EL3.TFP: This causes instructions that access the registers
	 *  associated with Floating Point and Advanced SIMD execution to trap
	 *  to EL3 when executed from any exception level, unless trapped to EL1
	 *  or EL2.
	 * ---------------------------------------------------------------------
	 */
	mrs	x0, cptr_el3
	bic	w0, w0, #TCPAC_BIT
	bic	w0, w0, #TTA_BIT
	bic	w0, w0, #TFP_BIT
	msr	cptr_el3, x0
.endm
```
# CPU数据

每个处理器有私有的数据

## 数据结构

其中保存了上下文信息、电源管理信息等

```c
typedef struct cpu_data {
#ifndef AARCH32
	void *cpu_context[2];/* cpu的上下文，根据security state分为两部分 */
#endif
	uintptr_t cpu_ops_ptr;/* 指针，执行的结构存放cpu操作的函数以及cpu的MIDR（cpu编号） */
#if CRASH_REPORTING/* 崩溃信息 */
	u_register_t crash_buf[CPU_DATA_CRASH_BUF_SIZE >> 3];
#endif
#if ENABLE_RUNTIME_INSTRUMENTATION/* PMF服务相关 */
	uint64_t cpu_data_pmf_ts[CPU_DATA_PMF_TS_COUNT];
#endif
	struct psci_cpu_data psci_svc_cpu_data;/* cpu电源管理状态 */
#if PLAT_PCPU_DATA_SIZE/* 平台特定数据 */
	uint8_t platform_cpu_data[PLAT_PCPU_DATA_SIZE];
#endif
} __aligned(CACHE_WRITEBACK_GRANULE) cpu_data_t;
```

## 初始化

每个cpu有一个**cpu_data_t**的数据，定义在`lib/el3_runtime/aarch64/cpu_data_array.c`中

```
cpu_data_t percpu_data[PLATFORM_CORE_COUNT];
```

为了便于cpu访问到自己的数据通过**TPIDR_EL3**指向自己的**cpu_data_t**，初始化过程如下

```assembly
func init_cpu_data_ptr
	mov	x10, x30 /* 保存lr便于下面调用函数 */
	bl	plat_my_core_pos /* 获取cpu的编号 */
	bl	_cpu_data_by_index/* 通过cpu编号获取cpu_data_t结构 */
	msr	tpidr_el3, x0
	ret	x10
endfunc init_cpu_data_ptr

func _cpu_data_by_index
	adr	x1, percpu_data
	add	x0, x1, x0, LSL #CPU_DATA_LOG2SIZE 
	ret
endfunc _cpu_data_by_index
```

## 访问CPU数据

通过读取**TPIDR_EL3**获取**cpu_data_t**

```c
static inline struct cpu_data *_cpu_data(void)
{
	return (cpu_data_t *)read_tpidr_el3();
}
```

# 中断现场保存恢复

## 单处理器相关

此部分代码位于`lib/el3_runtime/aarch64/context.S`中，声明在`include/lib/el3_runtime/aarch64/context.h`

`include/lib/el3_runtime/aarch64/context.h`声明了现场保护结构体（寄存器保存在内存中的结构），以及各个寄存器在现场保护结构体中的偏移量（在汇编程序中有效），并且声明了一些C语言访问现场抱回结构的方法

**DEFINE_REG_STRUCT**用于定义一组寄存器，**name**寄存器组的名字，**num_regs**为寄存器的个数

```c
#define DEFINE_REG_STRUCT(name, num_regs)	\
	typedef struct name {			\
		uint64_t _regs[num_regs];	\
	}  __aligned(16) name##_t
```

现场保护结构定义如下

```c
/* 声明一些寄存器组 */
DEFINE_REG_STRUCT(gp_regs, CTX_GPREG_ALL);
DEFINE_REG_STRUCT(el1_sys_regs, CTX_SYSREG_ALL);
#if CTX_INCLUDE_FPREGS
DEFINE_REG_STRUCT(fp_regs, CTX_FPREG_ALL);
#endif
DEFINE_REG_STRUCT(el3_state, CTX_EL3STATE_ALL);

/* 现场保护结构体 */
typedef struct cpu_context {
	gp_regs_t gpregs_ctx;/* 通用寄存器组 */
	el3_state_t el3state_ctx;/* 状态寄存器组 */
	el1_sys_regs_t sysregs_ctx;/* 系统寄存器组 */
#if CTX_INCLUDE_FPREGS
	fp_regs_t fpregs_ctx;/* 浮点寄存器组 */
#endif
} cpu_context_t;
```

为了方便访问现场保护结构体中的一组寄存器，特别定义如下宏

```c
#define get_el3state_ctx(h)	(&((cpu_context_t *) h)->el3state_ctx)
#if CTX_INCLUDE_FPREGS
#define get_fpregs_ctx(h)	(&((cpu_context_t *) h)->fpregs_ctx)
#endif
#define get_sysregs_ctx(h)	(&((cpu_context_t *) h)->sysregs_ctx)
#define get_gpregs_ctx(h)	(&((cpu_context_t *) h)->gpregs_ctx)
```

为了便于访问寄存器组的寄存器，特别定义如下宏用于读写寄存器

```c
#define read_ctx_reg(ctx, offset)	((ctx)->_regs[offset >> DWORD_SHIFT])
#define write_ctx_reg(ctx, offset, val)	(((ctx)->_regs[offset >> DWORD_SHIFT]) = val)
```

`context.S`文件中通过汇编实现了8个函数，用于现场保护还原。下面是这8个函数的C语言声明

```c
/* 保存系统寄存器到regs中 */
void el1_sysregs_context_save(el1_sys_regs_t* regs);

/* 从regs中恢复系统寄存器 */
void el1_sysregs_context_restore(el1_sys_regs_t *regs);

/* 保存浮点寄存器到regs中 */
void fpregs_context_save(fp_regs_t *regs);

/* 从regs中恢复浮点寄存器 */
void fpregs_context_restore(fp_regs_t *regs);

/* 保存X0-X29、SP_EL0到堆栈
   X30需要在CALL此函数前保存，CALL指令会修改X30 */
void save_gp_registers(void);

/* 从堆栈中恢复X0-X30、SP_EL0并退出异常 */
void restore_gp_registers_eret(void);

/* 从堆栈中恢复X4-X30、SP_EL0并退出异常
   此函数在特定情况下有效（SMC调用通过X0-X3返回调用结果） 
   但此函数实际应用较少，程序会通过访问现场保护的指针修改X0-X4 */
void restore_gp_registers_callee_eret(void)
```
## 多处理器相关

中断现场保护通过**SP_EL3**完成，但每个处理器需要分别初始化自己的**SP_EL3**。这部分在异常退出之前（引导下一级代码之前）通过设置**SP_EL3**完成，这样每个处理器发生中断时就有了独立的**上下文（context）**，具体通过**cm_set_next_context**函数实现

```c
static inline void cm_set_next_context(void *context)
{
#if DEBUG
  	/* 调试代码
  	   用于确定当前使用的是SP_EL0
  	   C语言运行环境使用的是SP_EL0 */
	uint64_t sp_mode;
	__asm__ volatile("mrs	%0, SPSel\n"
			 : "=r" (sp_mode));

	assert(sp_mode == MODE_SP_EL0);
#endif
	/* 切换到SP_EL3赋值后，修改为SP_EL0 */
	__asm__ volatile("msr	spsel, #1\n"
			 "mov	sp, %0\n"
			 "msr	spsel, #0\n"
			 : : "r" (context));
}
```

为了实现在**Secure Word**、**Non-Secure Word**之间切换，每个处理器有两个上下文。

```c
#define get_cpu_data(_m)		_cpu_data()->_m
#define set_cpu_data(_m, _v)	_cpu_data()->_m = _v

/* 获取当前安全状态下的上下文 */
void *cm_get_context(uint32_t security_state)
{
	assert(security_state <= NON_SECURE);
	return get_cpu_data(cpu_context[security_state]);
}

/* 设置当前安全状态下的上下文 */
void cm_set_context(void *context, uint32_t security_state)
{
	assert(security_state <= NON_SECURE);
	set_cpu_data(cpu_context[security_state], context);
}
```

# 加载镜像

镜像加载通过**load_auth_image**函数实现，并实现对镜像的认证

```c
int load_auth_image(meminfo_t *mem_layout,
		    unsigned int image_id,
		    uintptr_t image_base,
		    image_info_t *image_data,
		    entry_point_info_t *entry_point_info)
{
  	/* 实际的加载方法 */
	return load_auth_image_internal(mem_layout, image_id, image_base,
					image_data, entry_point_info, 0);
}
```

具体实现通过**load_auth_image_internal**实现

```c
static int load_auth_image_internal(meminfo_t *mem_layout,
				    unsigned int image_id,
				    uintptr_t image_base,
				    image_info_t *image_data,
				    entry_point_info_t *entry_point_info,
				    int is_parent_image)
{
	int rc;

/* 可信加载要先加载父镜像，这样才能校验下一级镜像 */
#if TRUSTED_BOARD_BOOT
	unsigned int parent_id;

	/* 获取父镜像id */
	rc = auth_mod_get_parent_id(image_id, &parent_id);
	if (rc == 0) {
		/* 递归，现加载认真父镜像 */
		rc = load_auth_image_internal(mem_layout, parent_id, image_base,
				     image_data, NULL, 1);
		if (rc != 0) {/* 出错直接退出 */
			return rc;
		}
	}
#endif /* TRUSTED_BOARD_BOOT */

	/* 镜像加载到内存，更新entry_point_info image_data */
	rc = load_image(mem_layout, image_id, image_base, image_data,
			entry_point_info);
	if (rc != 0) {
		return rc;
	}

#if TRUSTED_BOARD_BOOT
	/* 认证镜像 */
	rc = auth_mod_verify_img(image_id,
				 (void *)image_data->image_base,
				 image_data->image_size);
	if (rc != 0) {
		/* 镜像认证失败，清除加载的内容防止执行未认证的内容 */
		memset((void *)image_data->image_base, 0x00,
		       image_data->image_size);
		flush_dcache_range(image_data->image_base,
				   image_data->image_size);
		return -EAUTH;
	}
	/* 镜像加载认证成功，将镜像同步到主存储器，以便每个cpu都可以执行
	 * 不需要同步父镜像，应为只有当前镜像需要执行
	 */
	if (!is_parent_image) {
		flush_dcache_range(image_data->image_base,
				   image_data->image_size);
	}
#endif /* TRUSTED_BOARD_BOOT */

	return 0;
}
```

# 镜像认证

## 镜像认证描述符

认证描述符为**auth_img_desc_t**，结构体。

```c
typedef struct auth_img_desc_s {
	/* 镜像的ID,标志是哪一个镜像 */
	unsigned int img_id;

	/* 镜像类型(Binary、证书等) */
	img_type_t img_type;

	/* 父镜像，保存了认证当前镜像的公钥、哈希等 */
	const struct auth_img_desc_s *parent;

	/* 认证当前镜像的方法 */
	auth_method_desc_t img_auth_methods[AUTH_METHOD_NUM];

	/* 用于校验子镜像的公钥、哈希等 */
	auth_param_desc_t authenticated_data[COT_MAX_VERIFIED_PARAMS];
} auth_img_desc_t;
```

系统通过一个数组记录每个镜像的认证方式，并且通过镜像id获取对应镜像的认证方式

## 认证方法

**auth_mod_verify_img**根据镜像描述符描述的认证方式对镜像进行校验认证

```c

int auth_mod_verify_img(unsigned int img_id,
			void *img_ptr,
			unsigned int img_len)
{
	const auth_img_desc_t *img_desc = NULL;
	const auth_method_desc_t *auth_method = NULL;
	void *param_ptr;
	unsigned int param_len;
	int rc, i;

	/* 根据img_id获取镜像描述符 */
	img_desc = &cot_desc_ptr[img_id];

	/* 校验镜像完整性 */
	rc = img_parser_check_integrity(img_desc->img_type, img_ptr, img_len);
	return_if_error(rc);

	/* 根据镜像描述符的仍正方式对镜像进行认证 */
	for (i = 0 ; i < AUTH_METHOD_NUM ; i++) {
		auth_method = &img_desc->img_auth_methods[i];
		switch (auth_method->type) {
		case AUTH_METHOD_NONE:/* 不需要认证 */
			rc = 0;
			break;
		case AUTH_METHOD_HASH:/* 哈希认证 */
			rc = auth_hash(&auth_method->param.hash,
					img_desc, img_ptr, img_len);
			break;
		case AUTH_METHOD_SIG:/* 签名认证 */
			rc = auth_signature(&auth_method->param.sig,
					img_desc, img_ptr, img_len);
			break;
		case AUTH_METHOD_NV_CTR:/* Non-Volatile counter认证？ */
			rc = auth_nvctr(&auth_method->param.nv_ctr,
					img_desc, img_ptr, img_len);
			break;
		default:
			/* 未知认证类型，报错 */
			rc = 1;
			break;
		}
		return_if_error(rc);
	}

	/* 从镜像中解析出公钥哈希等，以便对下一级镜像进行认证 */
	for (i = 0 ; i < COT_MAX_VERIFIED_PARAMS ; i++) {
		if (img_desc->authenticated_data[i].type_desc == NULL) {
			continue;
		}

		/* 通过镜像解析器从镜像中提取内容 */
		rc = img_parser_get_auth_param(img_desc->img_type,
				img_desc->authenticated_data[i].type_desc,
				img_ptr, img_len, &param_ptr, &param_len);
		return_if_error(rc);

		/* 异常检查
		   防止从镜像中解析出的数据字节数大于镜像描述符中的字节数
		   出现内存访问溢出 */
		if (param_len > img_desc->authenticated_data[i].data.len) {
			return 1;
		}

		/* 把解析出的内容拷贝到镜像描述符中，便于解析下一级BL */
		memcpy((void *)img_desc->authenticated_data[i].data.ptr,
				(void *)param_ptr, param_len);
	}

	/* 标记镜像以认证过 */
	auth_img_flags[img_desc->img_id] |= IMG_FLAG_AUTHENTICATED;

	return 0;
}
```



# 执行镜像

## 执行环境

执行环境，用于初始化**上下文**。此结构一般在加载镜像时初始化，通过系统定义的或链接器导出的符号初始化。

```c
/* 结构体的一个标记，用于校验和错误检测 */
typedef struct param_header {
	uint8_t type;		/* 结构体类型 */
	uint8_t version;    /* 版本信息 */
	uint16_t size;      /* 结构体大小 */
	uint32_t attr;      /* 附加属性，第0比特标记是否为secure */
} param_header_t;

/* 用于向镜像传递参数 */
typedef struct aapcs32_params {
	u_register_t arg0;
	u_register_t arg1;
	u_register_t arg2;
	u_register_t arg3;
} aapcs32_params_t;

/* 用于向镜像传递参数 */
typedef struct aapcs64_params {
	u_register_t arg0;
	u_register_t arg1;
	u_register_t arg2;
	u_register_t arg3;
	u_register_t arg4;
	u_register_t arg5;
	u_register_t arg6;
	u_register_t arg7;
} aapcs64_params_t;

/* 执行环境结构体 */
typedef struct entry_point_info {
	param_header_t h;
	uintptr_t pc;/* 镜像加载在内存中的位置 */
	uint32_t spsr;/* 程序状态字（决定异常等级） */
#ifdef AARCH32
	aapcs32_params_t args;/* 用于向镜像传递参数 */
#else
	aapcs64_params_t args;/* 用于向镜像传递参数 */
#endif
} entry_point_info_t;
```

## 通过异常退出执行下一级BL

这种方式下，上一级BL可以为下一级BL提供服务。主要通过两个函数实现

```c
/* 初始化当前处理器的上下文 */
void cm_init_my_context(const entry_point_info_t *ep)
{
	cpu_context_t *ctx;
    /* 根据secure的状态获取上下文 */
	ctx = cm_get_context(GET_SECURITY_STATE(ep->h.attr));
    /* 根据执行环境初始化上下文 */
	cm_init_context_common(ctx, ep);
}

/* 为异常退出作准备 */
void cm_prepare_el3_exit(uint32_t security_state)
{
	uint32_t sctlr_elx, scr_el3, cptr_el2;
    
    /* 根据secure的状态获取上下文 */
	cpu_context_t *ctx = cm_get_context(security_state);

	assert(ctx);
	/* 设置一些控制寄存器，为异常退出作准备 */
	if (security_state == NON_SECURE) {
		scr_el3 = read_ctx_reg(get_el3state_ctx(ctx), CTX_SCR_EL3);
		if (scr_el3 & SCR_HCE_BIT) {
			/* Use SCTLR_EL1.EE value to initialise sctlr_el2 */
			sctlr_elx = read_ctx_reg(get_sysregs_ctx(ctx),
						 CTX_SCTLR_EL1);
			sctlr_elx &= ~SCTLR_EE_BIT;
			sctlr_elx |= SCTLR_EL2_RES1;
			write_sctlr_el2(sctlr_elx);
		} else if (read_id_aa64pfr0_el1()
                   & (ID_AA64PFR0_ELX_MASK << ID_AA64PFR0_EL2_SHIFT)) {
			write_hcr_el2((scr_el3 & SCR_RW_BIT) ? HCR_RW_BIT : 0);
			cptr_el2 = read_cptr_el2();
			cptr_el2 &= ~(TCPAC_BIT | TTA_BIT | TFP_BIT);
			write_cptr_el2(cptr_el2);
			write_cnthctl_el2(EL1PCEN_BIT | EL1PCTEN_BIT);
			write_cntvoff_el2(0);
			write_vpidr_el2(read_midr_el1());
			write_vmpidr_el2(read_mpidr_el1());
			write_vttbr_el2(0);
			write_mdcr_el2((read_pmcr_el0() & PMCR_EL0_N_BITS)>> PMCR_EL0_N_SHIFT);
			write_hstr_el2(0);
			write_cnthp_ctl_el2(0);
		}
	}
	
    /* 从上下文恢复系统寄存器 */
	el1_sysregs_context_restore(get_sysregs_ctx(ctx));

  	/* 切换使用SP_EL3，并指向上下文 */
	cm_set_next_context(ctx);
}
```

异常退出通过，汇编函数实现

```assembly
func el3_exit
    /* 保存SP_EL0到上下文中 */
	mov	x17, sp
	msr	spsel, #1
	str	x17, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]

	/* 
	 * 从上下文恢复SCR_EL3、SPSR_EL3、ELR_EL3
	 * SCR_EL3  : security state有关
	 * SPSR_EL3 : 异常等级相关
	 * ELR_EL3  : 异常返回地址
     */
	ldr	x18, [sp, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]
	ldp	x16, x17, [sp, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]
	msr	scr_el3, x18
	msr	spsr_el3, x16
	msr	elr_el3, x17

	/* 从上下文恢复通用寄存器 */
	b	restore_gp_registers_eret
endfunc el3_exit
```

## 通过SMC执行下一级BL

这是BL1给BL2提供的中断服务，用于加载程序到EL3

```assembly
func smc_handler64

	/* 确定是BL1_SMC_RUN_IMAGE调用 */
	mov	x30, #BL1_SMC_RUN_IMAGE
	cmp	x30, x0
	b.ne	smc_handler/* 其他SMC调用 */

	/* 确定在secure world执行SMC调用 */
	mrs	x30, scr_el3
	tst	x30, #SCR_NS_BIT
	b.ne	unexpected_sync_exception/* Non-Secure调用BL1_SMC_RUN_IMAGE */
	
	/* 从上下文恢复C运行环境堆栈 */
	ldr	x30, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	msr	spsel, #0
	mov	sp, x30

	/* X1执行执行环境 */
	mov	x20, x1
	mov	x0, x20
	bl	bl1_print_next_bl_ep_info/* 打印执行环境 */

	ldp	x0, x1, [x20, #ENTRY_POINT_INFO_PC_OFFSET]
	msr	elr_el3, x0/* 修改返回地址，用来在异常退出时执行指定的镜像文件 */
	msr	spsr_el3, x1/* 修改返回的cpsr,可以修改异常等级 */
	ubfx	x0, x1, #MODE_EL_SHIFT, #2
	cmp	x0, #MODE_EL3/* 执行的镜像必须为EL3，为后面的代码放开权限 */
	b.ne	unexpected_sync_exception

	bl	disable_mmu_icache_el3 /* 关闭指令cache */
	tlbi	alle3

/* 用于打印一些调试信息 */
#if SPIN_ON_BL1_EXIT
	bl	print_debug_loop_message
debug_loop:
	b	debug_loop
#endif

	mov	x0, x20
	bl	bl1_plat_prepare_exit/* 为异常退出准备调节 */
	/* 从x1指定的结构体中获取参数，并退出异常 */
	ldp	x6, x7, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x30)]
	ldp	x4, x5, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x20)]
	ldp	x2, x3, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x10)]
	ldp	x0, x1, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x0)]
	eret
endfunc smc_handler64
```

此服务在BL2中使用

```c
/*
 * smc用于触发一条SMC调用，参数对应X0-X7
 * X0标记服务
 * X1执行环境结构体指针
 */
smc(BL1_SMC_RUN_IMAGE, (unsigned long)next_bl_ep_info, 0, 0, 0, 0, 0, 0);
```

# SMC服务框架

SMC调用通过W0标示当前调用的类型。W0根据如下方式标记调用类型

```
W0用于确定调用的功能
b31    调用类型 1->Fast Call 0->Yielding Call
b30    标示32bit还是64bit调用
b29:24 服务类型
        0x0        ARM架构相关
        0x1        CPU服务
        0x2        SiP服务
        0x3        OEM服务
        0x4        标准安全服务
        0x5        标准Hypervisor服务
        0x6        厂商定制的Hypervisor服务
        0x07-0x2f  预留
        0x30-0x31  Trusted Application Calls
        0x32-0x3f  Trusted OS Calls
b23:16 在Fast Call时这些比特位必须为0，其他值保留未用
       ARMv7传统的Trusted OS在Fast Call时必须为1
b15:0  函数号
```

## SMC数据结构

SMC服务通过结构体描述

```C
typedef struct rt_svc_desc {
  	/* start_oen-end_oen 对应W0 b29:24中的一个片段 */
	uint8_t start_oen;     /* 当前服务可以处理的SMC调用编号的起始编号 */
	uint8_t end_oen;       /* 当前服务可以处理的SMC调用编号的结束编号 */
	uint8_t call_type;     /* 调用的类型，对应W0 b30 */
	const char *name;      /* 名字 */
	rt_svc_init_t init;    /* 服务初始化函数 */
	rt_svc_handle_t handle;/* 服务句柄 */
} rt_svc_desc_t;
```

通过**DECLARE_RT_SVC**的宏声明一个SMC服务，服务被放在**rt_svc_descs**段中，并在链接脚本中指定**rt_svc_descs**的起始结束符号（**\_\_RT\_SVC\_DESCS\_START\_\_**/**\_\_RT\_SVC\_DESCS\_END\_\_**）

```c
#define DECLARE_RT_SVC(_name, _start, _end, _type, _setup, _smch) \
	static const rt_svc_desc_t __svc_desc_ ## _name \
		__section("rt_svc_descs") __used = { \
			.start_oen = _start, \
			.end_oen = _end, \
			.call_type = _type, \
			.name = #_name, \
			.init = _setup, \
			.handle = _smch }
```

通过如上结构，构建了一个可扩展的SMC服务框架

## SMC服务初始化构建索引

每个服务需要初始化（**rt_svc_desc_t.init**），而且链接后所有的服务形成一个数组被放在**\_\_RT\_SVC\_DESCS\_START\_\_**和**\_\_RT\_SVC\_DESCS\_END\_\_**符号之间，需要建立一个索引通过服务号找到对应的**rt_svc_desc_t**

```c
void runtime_svc_init(void)
{
	int rc = 0, index, start_idx, end_idx;

	/* 异常检查 */
	assert((RT_SVC_DESCS_END >= RT_SVC_DESCS_START) &&
			(RT_SVC_DECS_NUM < MAX_RT_SVCS));

	/* 没有系统服务退出 */
	if (RT_SVC_DECS_NUM == 0)
		return;

	/* 初始化索引*/
	memset(rt_svc_descs_indices, -1, sizeof(rt_svc_descs_indices));

    /* 遍历所有的SMC服务 */
    rt_svc_descs = (rt_svc_desc_t *) RT_SVC_DESCS_START;
	for (index = 0; index < RT_SVC_DECS_NUM; index++) {
		rt_svc_desc_t *service = &rt_svc_descs[index];

      	/* 检查SMC服务是否合法 */
		rc = validate_rt_svc_desc(service);
		if (rc) {
			ERROR("Invalid runtime service descriptor %p\n",
				(void *) service);
			panic();
		}

		/*
		 * 调用初始话函数
		 */
		if (service->init) {
			rc = service->init();
			if (rc) {
				ERROR("Error initializing runtime service %s\n",
						service->name);
				continue;
			}
		}

		/*
		 * 构建索引
		 * 通过调用类型和服务类型查找服务在rt_svc_descs中的下标
		 */
		start_idx = get_unique_oen(rt_svc_descs[index].start_oen,
				service->call_type);
		assert(start_idx < MAX_RT_SVCS);
		end_idx = get_unique_oen(rt_svc_descs[index].end_oen,
				service->call_type);
		assert(end_idx < MAX_RT_SVCS);
		for (; start_idx <= end_idx; start_idx++)
			rt_svc_descs_indices[start_idx] = index;
	}
}
```
## SMC中断服务

通过初始化构建的索引找到对应的服务，执行**rt_svc_desc_t.handle**

```assembly
     /* 同步异常处理只处理smc相关的调用 */
	.macro	handle_sync_exception
	/* 使能SError中断 */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x30，下面要使用X30作中间变量分析异常来源 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]

#if ENABLE_RUNTIME_INSTRUMENTATION
	/*
	 * Read the timestamp value and store it in per-cpu data. The value
	 * will be extracted from per-cpu data by the C level SMC handler and
	 * saved to the PMF timestamp region.
	 */
	mrs	x30, cntpct_el0/* 时钟 */
	str	x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X29]
	mrs	x29, tpidr_el3 /* tpidr_el3指向cpu_data */
	str	x30, [x29, #CPU_DATA_PMF_TS0_OFFSET]
	ldr	x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X29]
#endif
	/* 获取异常代码 */
	mrs	x30, esr_el3
	ubfx	x30, x30, #ESR_EC_SHIFT, #ESR_EC_LENGTH

	cmp	x30, #EC_AARCH32_SMC
	b.eq	smc_handler32 /* 32bit SMC调用 */

	cmp	x30, #EC_AARCH64_SMC
	b.eq	smc_handler64 /* 64bit SMC调用 */

	no_ret	report_unhandled_exception/* 不处理其他的同步异常 */
	.endm
```



```assembly
smc_handler64:
	/*
	 * Populate the parameters for the SMC handler.
	 * We already have x0-x4 in place. x5 will point to a cookie (not used
	 * now). x6 will point to the context structure (SP_EL3) and x7 will
	 * contain flags we need to pass to the handler Hence save x5-x7.
	 *
	 * Note: x4 only needs to be preserved for AArch32 callers but we do it
	 *       for AArch64 callers as well for convenience
	 */
	stp	x4, x5, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X4]
	stp	x6, x7, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X6]

	/* Save rest of the gpregs and sp_el0*/
	save_x18_to_x29_sp_el0

	mov	x5, xzr
	mov	x6, sp

	/* Get the unique owning entity number */
    /* X0用于确定调用的功能
     * b31    调用类型 1->Fast Call 0->Yielding Call
     * b30    标示32bit还是64bit调用
     * b29:24 服务类型
     *           0x0  ARM架构相关
     *           0x1  CPU服务
     *           0x2  SiP服务
     *           0x3  OEM服务
     *           0x4  标准安全服务
     *           0x5  标准Hypervisor服务
     *           0x6  厂商定制的Hypervisor服务
     *     0x07-0x2f  预留
     *     0x30-0x31  Trusted Application Calls
     *     0x32-0x3f  Trusted OS Calls
     * b23:16 在Fast Call时这些比特位必须为0，其他值保留未用
     *        ARMv7传统的Trusted OS在Fast Call时必须为1
     * b15:0  函数号
     */
	ubfx	x16, x0, #FUNCID_OEN_SHIFT, #FUNCID_OEN_WIDTH   /* 服务类型 */
	ubfx	x15, x0, #FUNCID_TYPE_SHIFT, #FUNCID_TYPE_WIDTH /* 调用类型 */
	orr	x16, x16, x15, lsl #FUNCID_OEN_WIDTH /* 拼接 调用类型：服务类型 */

    /* __RT_SVC_DESCS_START__为rt_svc_descs段的起始地址，此段存放服务描述符
     * RT_SVC_DESC_HANDLE为服务句柄在服务描述符结构体中的偏移量
     * 以下代码保存第一个复位句柄的地址到x11中
     */
	adr	x11, (__RT_SVC_DESCS_START__ + RT_SVC_DESC_HANDLE)

    /* 因为描述符在内存中布局是乱的通过rt_svc_descs_indices数组来查找描述符在内存中的位置
     * rt_svc_descs_indices为rt_svc_descs的索引
     * 获取数组下标 -> W15
     */
	adr	x14, rt_svc_descs_indices
	ldrb	w15, [x14, x16]

	/* 从上下文中提取C运行环境的栈指针 */
	ldr	x12, [x6, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	
	/* 索引的值不应该大于127 */
	tbnz	w15, 7, smc_unknown

	/* 切换到C的指针 */
	msr	spsel, #0

	/*
	 * Get the descriptor using the index
	 * x11 = (base + off), x15 = index
	 *
	 * handler = (base + off) + (index << log2(size))
	 */
	lsl	w10, w15, #RT_SVC_SIZE_LOG2/* 计算偏移量 */
	ldr	x15, [x11, w10, uxtw]/* 加载出对应服务的句柄 */

	/*
	 * Save the SPSR_EL3, ELR_EL3, & SCR_EL3 in case there is a world
	 * switch during SMC handling.
	 * TODO: Revisit if all system registers can be saved later.
	 */
	mrs	x16, spsr_el3
	mrs	x17, elr_el3
	mrs	x18, scr_el3
	stp	x16, x17, [x6, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]
	str	x18, [x6, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]

	/* Copy SCR_EL3.NS bit to the flag to indicate caller's security */
	bfi	x7, x18, #0, #1/* x7用于传递给服务函数security state */

	mov	sp, x12

	/*
	 * Call the Secure Monitor Call handler and then drop directly into
	 * el3_exit() which will program any remaining architectural state
	 * prior to issuing the ERET to the desired lower EL.
	 */
#if DEBUG
	cbz	x15, rt_svc_fw_critical_error/* x15等于0说明有错误存在 */
#endif
	blr	x15 /* 调用对应的服务 */

	b	el3_exit/* 异常恢复 */
```

# PSCI

## 初始化

在BL1阶段除主处理器外，其他处理器被暂停

```assembly
/* bl1入口第一行 */
el3_entrypoint_common					\
		_set_endian=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=bl1_exceptions
		
/* el3_entrypoint_common宏中的片段 */
.if \_secondary_cold_boot
		bl	plat_is_my_cpu_primary /* 判断是否为主处理器 */
		cmp	r0, #0
		bne	do_primary_cold_boot

		/* 次处理器处理 */
		bl	plat_secondary_cold_boot_setup/* 此函数与具体平台实现有关 */
		/* 此代码不会执行，因为plat_secondary_cold_boot_setup不会返回 */
		no_ret	plat_panic_handler
	/* 主处理器继续执行 */
	do_primary_cold_boot:
.endif /* _secondary_cold_boot */

```

在BL31阶段，在SMC的一个服务中（Standard Service）中实现PSCI服务。

```c
/* 服务初始化函数 */
static int32_t std_svc_setup(void)
{
	uintptr_t svc_arg;
	
    /* cpu被唤醒的入口函数 */
	svc_arg = get_arm_std_svc_args(PSCI_FID_MASK);
	assert(svc_arg);

	/*
	 *PSCI作为标准服务实现，初始化PSCI服务
	 */
	return psci_setup((const psci_lib_args_t *)svc_arg);
}

/* PSCI服务初始化的参数 */
uintptr_t get_arm_std_svc_args(unsigned int svc_mask)
{
	/* 构造PSCI服务初始化的参数，cpu被唤醒入口为bl31_warm_entrypoint */
	DEFINE_STATIC_PSCI_LIB_ARGS_V1(psci_args, bl31_warm_entrypoint);

	/* PSCI is the only ARM Standard Service implemented */
	assert(svc_mask == PSCI_FID_MASK);

	return (uintptr_t)&psci_args;
}

/* PSCI服务初始化 */
int psci_setup(const psci_lib_args_t *lib_args)
{
	const unsigned char *topology_tree;
	
  	/* 校验参数是否有效 */
	assert(VERIFY_PSCI_LIB_ARGS_V1(lib_args));

	/* 架构相关初始化 */
	psci_arch_setup();

	/* 获取芯片拓扑图（第一个字节标示有几个簇，后面的字节标示各个簇有几个cup） */
	topology_tree = plat_get_power_domain_tree_desc();

	/* Populate the power domain arrays using the platform topology map */
	populate_power_domain_tree(topology_tree);

	/* Update the CPU limits for each node in psci_non_cpu_pd_nodes */
	psci_update_pwrlvl_limits();

	/* Populate the mpidr field of cpu node for this CPU */
	psci_cpu_pd_nodes[plat_my_core_pos()].mpidr =
		read_mpidr() & MPIDR_AFFINITY_MASK;

	psci_init_req_local_pwr_states();

	/*
	 * Set the requested and target state of this CPU and all the higher
	 * power domain levels for this CPU to run.
	 */
	psci_set_pwr_domains_to_run(PLAT_MAX_PWR_LVL);
	
    /* 获取平台支持的电源管理操作句柄psci_plat_pm_ops
     * 并设置芯片被唤醒的入口lib_args->mailbox_ep，即bl31_warm_entrypoint
     */
	plat_setup_psci_ops((uintptr_t)lib_args->mailbox_ep, &psci_plat_pm_ops);
	assert(psci_plat_pm_ops);

	/*
	 * Flush `psci_plat_pm_ops` as it will be accessed by secondary CPUs
	 * during warm boot before data cache is enabled.
	 */
	flush_dcache_range((uintptr_t)&psci_plat_pm_ops,
					sizeof(psci_plat_pm_ops));

	/* 标记可用的电源管理操作 */
	psci_caps = PSCI_GENERIC_CAP;

	if (psci_plat_pm_ops->pwr_domain_off)
		psci_caps |=  define_psci_cap(PSCI_CPU_OFF);
	if (psci_plat_pm_ops->pwr_domain_on &&
			psci_plat_pm_ops->pwr_domain_on_finish)
		psci_caps |=  define_psci_cap(PSCI_CPU_ON_AARCH64);
	if (psci_plat_pm_ops->pwr_domain_suspend &&
			psci_plat_pm_ops->pwr_domain_suspend_finish) {
		psci_caps |=  define_psci_cap(PSCI_CPU_SUSPEND_AARCH64);
		if (psci_plat_pm_ops->get_sys_suspend_power_state)
			psci_caps |=  define_psci_cap(PSCI_SYSTEM_SUSPEND_AARCH64);
	}
	if (psci_plat_pm_ops->system_off)
		psci_caps |=  define_psci_cap(PSCI_SYSTEM_OFF);
	if (psci_plat_pm_ops->system_reset)
		psci_caps |=  define_psci_cap(PSCI_SYSTEM_RESET);
	if (psci_plat_pm_ops->get_node_hw_state)
		psci_caps |= define_psci_cap(PSCI_NODE_HW_STATE_AARCH64);

#if ENABLE_PSCI_STAT
	psci_caps |=  define_psci_cap(PSCI_STAT_RESIDENCY_AARCH64);
	psci_caps |=  define_psci_cap(PSCI_STAT_COUNT_AARCH64);
#endif
	return 0;
}

```

## 服务

PSCI提供电源管理相关的服务

```c
u_register_t psci_smc_handler(uint32_t smc_fid,
			  u_register_t x1,
			  u_register_t x2,
			  u_register_t x3,
			  u_register_t x4,
			  void *cookie,
			  void *handle,
			  u_register_t flags)
{
  	/* 只为Non-Secure服务 */
	if (is_caller_secure(flags))
		return SMC_UNK;

	/* 检测调用的服务是否存在 */
	if (!(psci_caps & define_psci_cap(smc_fid)))
		return SMC_UNK;

	if (((smc_fid >> FUNCID_CC_SHIFT) & FUNCID_CC_MASK) == SMC_32) {
		/* 32比特PSCI服务 */

		x1 = (uint32_t)x1;
		x2 = (uint32_t)x2;
		x3 = (uint32_t)x3;

		switch (smc_fid) {
		case PSCI_VERSION:
			return psci_version();

		case PSCI_CPU_OFF:
			return psci_cpu_off();

		case PSCI_CPU_SUSPEND_AARCH32:
			return psci_cpu_suspend(x1, x2, x3);

		case PSCI_CPU_ON_AARCH32:
			return psci_cpu_on(x1, x2, x3);

		case PSCI_AFFINITY_INFO_AARCH32:
			return psci_affinity_info(x1, x2);

		case PSCI_MIG_AARCH32:
			return psci_migrate(x1);

		case PSCI_MIG_INFO_TYPE:
			return psci_migrate_info_type();

		case PSCI_MIG_INFO_UP_CPU_AARCH32:
			return psci_migrate_info_up_cpu();

		case PSCI_NODE_HW_STATE_AARCH32:
			return psci_node_hw_state(x1, x2);

		case PSCI_SYSTEM_SUSPEND_AARCH32:
			return psci_system_suspend(x1, x2);

		case PSCI_SYSTEM_OFF:
			psci_system_off();
			/* We should never return from psci_system_off() */

		case PSCI_SYSTEM_RESET:
			psci_system_reset();
			/* We should never return from psci_system_reset() */

		case PSCI_FEATURES:
			return psci_features(x1);

#if ENABLE_PSCI_STAT
		case PSCI_STAT_RESIDENCY_AARCH32:
			return psci_stat_residency(x1, x2);

		case PSCI_STAT_COUNT_AARCH32:
			return psci_stat_count(x1, x2);
#endif

		default:
			break;
		}
	} else {
		/* 64位的PSCI服务 */

		switch (smc_fid) {
		case PSCI_CPU_SUSPEND_AARCH64:
			return psci_cpu_suspend(x1, x2, x3);

		case PSCI_CPU_ON_AARCH64:
			return psci_cpu_on(x1, x2, x3);

		case PSCI_AFFINITY_INFO_AARCH64:
			return psci_affinity_info(x1, x2);

		case PSCI_MIG_AARCH64:
			return psci_migrate(x1);

		case PSCI_MIG_INFO_UP_CPU_AARCH64:
			return psci_migrate_info_up_cpu();

		case PSCI_NODE_HW_STATE_AARCH64:
			return psci_node_hw_state(x1, x2);

		case PSCI_SYSTEM_SUSPEND_AARCH64:
			return psci_system_suspend(x1, x2);

#if ENABLE_PSCI_STAT
		case PSCI_STAT_RESIDENCY_AARCH64:
			return psci_stat_residency(x1, x2);

		case PSCI_STAT_COUNT_AARCH64:
			return psci_stat_count(x1, x2);
#endif

		default:
			break;
		}
	}
	/* 未定义的服务 */
	WARN("Unimplemented PSCI Call: 0x%x \n", smc_fid);
	return SMC_UNK;
}
```

# TEE

arm-trusted-firware并没有实现一个完整的TEE，只给出了一个简单的与TEE类似的程序（BL32）。AARCH64 TEE应该是BL31（monitor）与BL32（Trusted OS）配合工作的。一般TOS通过monitor的一个SMC服务与Non-Secure通信。

## Monitor SMC服务

此SMC服务需要启动TOS，并且为TOS与Non-Secure提供通信服务。

### TOS启动

应为arm-trusted-firware并没有实际的TOS，这里以tsp（位于`bl32\tsp`，对应的SMC服务位于`services\spd\tspd`）为例作部分介绍。

其中有一个宏**TSP_INIT_ASYNC**

当宏**TSP_INIT_ASYNC**定义时

1. BL31（monitor）在异常退出时执行BL32（TOS）

2. BL32（TOS）初始化完成后通过SMC调用通知BL31（monitor）

3. BL31（monitor）在SMC服务中启动BL33（APPBL）

当宏**TSP_INIT_ASYNC**未定义时

1. BL31（monitor）在SMC服务初始化完成后通过异常退出执行BL32（TOS）

2. BL32（TOS）初始化完成后通过SMC调用返回初始化的位置

3. BL31（monitor）执行完才退出异常执行BL33（APPBL）


此处涉及代码比较多

#### BL31主函数

```c
void bl31_main(void)
{
	//……

	/* 初始化SMC服务 
	 * 为TOS服务的SMC服务会根据TSP_INIT_ASYNC调用部分函数
	 *   void bl31_set_next_image_type(uint32_t security_state)
	 *   void bl31_register_bl32_init(int32_t (*func)(void))
     */
	INFO("BL31: Initializing runtime services\n");
	runtime_svc_init();

	/*
	 * 当宏TSP_INIT_ASYNC未定义时
	 * 给TOS服务的SMC宏将初始化此函数句柄
	 * 此函数句柄将会通过异常退出执行TOS（BL32）
	 */
	if (bl32_init) {
		INFO("BL31: Initializing BL32\n");
		(*bl32_init)();
	}

	 /* 初始化下一级BL的执行环境 */
	bl31_prepare_next_image_entry();

	//……
}
```

以上代码是一个简化的主函数。在SMC服务初始化时，为TOS服务的SMC服务将根据**TSP_INIT_ASYNC**调用**bl31_set_next_image_type**、**bl31_register_bl32_init**

**bl31_set_next_image_type**相关部分，**TSP_INIT_ASYNC**未定义时此不分发挥作用

```c
/* 默认BL31执行完成后异常退出要执行的镜像的security state */
static uint32_t next_image_type = NON_SECURE;

/* 设置BL31执行完成异常退出要执行镜像的security state
 * security_state = NON_SECURE 执行BL33 APPBL
 * security_state = SECURE 执行BL32 TOS
 */
void bl31_set_next_image_type(uint32_t security_state)
{
	assert(sec_state_is_valid(security_state));
	next_image_type = security_state;
}

/* 在bl31_prepare_next_image_entry函数中被调用，用于初始化下一级BL */
uint32_t bl31_get_next_image_type(void)
{
	return next_image_type;
}
```

**bl31_register_bl32_init**相关部分，**TSP_INIT_ASYNC**定义时此不分发挥作用

```c
/* 函数指针，用于初始化TOS */
static int32_t (*bl32_init)(void);
/* 设置bl32_init*/
void bl31_register_bl32_init(int32_t (*func)(void))
{
	bl32_init = func;
}
```

#### TSPD

TSPD并不是一个完整的TOS，只用于测试。不过可以用于启动分析。

##### 服务初始化

```c
int32_t tspd_setup(void)
{
	entry_point_info_t *tsp_ep_info;
	uint32_t linear_id;

	linear_id = plat_my_core_pos();

	/* 获取TOS执行环境并做部分验证工作 */
	tsp_ep_info = bl31_plat_get_next_image_ep_info(SECURE);
	if (!tsp_ep_info) {
		WARN("No TSP provided by BL2 boot loader, Booting device"
			" without TSP initialization. SMC`s destined for TSP"
			" will return SMC_UNK\n");
		return 1;
	}
	if (!tsp_ep_info->pc)
		return 1;

	/*
	 * 执行部分初始化工作
	 */
	tspd_init_tsp_ep_state(tsp_ep_info,
				TSP_AARCH64,
				tsp_ep_info->pc,
				&tspd_sp_context[linear_id]);

#if TSP_INIT_ASYNC
  /* 设置下一级BL的security state
     在异常退出时执行TOS */
	bl31_set_next_image_type(SECURE);
#else
	/* 在bl32_init中初始话BL32*/
	bl31_register_bl32_init(&tspd_init);
#endif
	return 0;
}
```

##### 异常处理

```c
uint64_t tspd_smc_handler(uint32_t smc_fid,
			 uint64_t x1,
			 uint64_t x2,
			 uint64_t x3,
			 uint64_t x4,
			 void *cookie,
			 void *handle,
			 uint64_t flags)
{
	cpu_context_t *ns_cpu_context;
	uint32_t linear_id = plat_my_core_pos(), ns;
	tsp_context_t *tsp_ctx = &tspd_sp_context[linear_id];
	uint64_t rc;
#if TSP_INIT_ASYNC
	entry_point_info_t *next_image_info;
#endif

	/* Determine which security state this SMC originated from */
	ns = is_caller_non_secure(flags);

	switch (smc_fid) {
	//……
	/* TSP_ENTRY_DONE在BL32初始化完才，通过SMC返回 */
	case TSP_ENTRY_DONE:
		if (ns)/* 此异常只能来自Secure */
			SMC_RET1(handle, SMC_UNK);
		
        /* 此异常只能发生一次 */
		assert(tsp_vectors == NULL);
		tsp_vectors = (tsp_vectors_t *) x1;/* 记录服务函数 */

		if (tsp_vectors) {
          	/* 记录状态 */
			set_tsp_pstate(tsp_ctx->state, TSP_PSTATE_ON);

			/* TSP初始化完才，注册电源管理句柄 */
			psci_register_spd_pm_hook(&tspd_pm);

          	/* S-EL1注册中断处理函数 */
			flags = 0;
			set_interrupt_rm_flag(flags, NON_SECURE);
			rc = register_interrupt_type_handler(INTR_TYPE_S_EL1,
						tspd_sel1_interrupt_handler,
						flags);
			if (rc)
				panic();

#if TSP_NS_INTR_ASYNC_PREEMPT
			/* Non-Secure注册中断处理函数 */
			flags = 0;
			set_interrupt_rm_flag(flags, SECURE);

			rc = register_interrupt_type_handler(INTR_TYPE_NS,
						tspd_ns_interrupt_handler,
						flags);
			if (rc)
				panic();

			/* 禁止中断 */
			disable_intr_rm_local(INTR_TYPE_NS, SECURE);
#endif
		}


#if TSP_INIT_ASYNC
		/* 启动BL33(APPBL) */
		assert(cm_get_context(SECURE) == &tsp_ctx->cpu_ctx);
		cm_el1_sysregs_context_save(SECURE);
		next_image_info = bl31_plat_get_next_image_ep_info(NON_SECURE);
		assert(next_image_info);
		assert(NON_SECURE ==
				GET_SECURITY_STATE(next_image_info->h.attr));
		cm_init_my_context(next_image_info);
		cm_prepare_el3_exit(NON_SECURE);
		SMC_RET0(cm_get_context(NON_SECURE));
#else
		/* 返回tspd_synchronous_sp_entry处继续BL31的main方法 */
		tspd_synchronous_sp_exit(tsp_ctx, x1);
#endif
	
	//……
	default:
		break;
	}

	SMC_RET1(handle, SMC_UNK);
}
```

##### 同步执行

上面遗产处理中，有一组特别的函数**tspd_synchronous_sp_entry** 、**tspd_synchronous_sp_exit**

**tspd_synchronous_sp_entry**用于退出异常执行下一级BL

**tspd_synchronous_sp_exit**用于下一级BL执行结束后通过SMC返回，然后跳转到**tspd_synchronous_sp_entry**后继续执行。

具体代码如下

```c
uint64_t tspd_synchronous_sp_entry(tsp_context_t *tsp_ctx)
{
	uint64_t rc;

	assert(tsp_ctx != NULL);
	assert(tsp_ctx->c_rt_ctx == 0);
	assert(cm_get_context(SECURE) == &tsp_ctx->cpu_ctx);
  	
  	/* 恢复SECURE下的系统寄存器 */
	cm_el1_sysregs_context_restore(SECURE);
  	/* 设置遗产返回相关的寄存器，以便异常退出后执行对应的程序 */
	cm_set_next_eret_context(SECURE);
  	/* 保存x19-x30到堆栈，并记录堆栈指针到tsp_ctx->c_rt_ctx */
	rc = tspd_enter_sp(&tsp_ctx->c_rt_ctx);
#if DEBUG
	tsp_ctx->c_rt_ctx = 0;
#endif

	return rc;
}

void tspd_synchronous_sp_exit(tsp_context_t *tsp_ctx, uint64_t ret)
{
	assert(tsp_ctx != NULL);
	assert(cm_get_context(SECURE) == &tsp_ctx->cpu_ctx);
    /* 保存SECURE下的系统寄存器 */
	cm_el1_sysregs_context_save(SECURE);

	assert(tsp_ctx->c_rt_ctx != 0);
  	
  	/* 通过tsp_ctx->c_rt_ctx恢复SP，并从堆栈中恢复x19-x30 */
	tspd_exit_sp(tsp_ctx->c_rt_ctx, ret);

	/* 此处不会执行 */
	assert(0);
}
```

**tspd_enter_sp**、**tspd_exit_sp**实现细节如下

```assembly
	.global	tspd_enter_sp
func tspd_enter_sp
	/* 保存SP到X0指定的地址 */
	mov	x3, sp
	str	x3, [x0, #0]	
	
	/* 开辟栈空间保存x19-x30 */
	sub	sp, sp, #TSPD_C_RT_CTX_SIZE
	stp	x19, x20, [sp, #TSPD_C_RT_CTX_X19]
	stp	x21, x22, [sp, #TSPD_C_RT_CTX_X21]
	stp	x23, x24, [sp, #TSPD_C_RT_CTX_X23]
	stp	x25, x26, [sp, #TSPD_C_RT_CTX_X25]
	stp	x27, x28, [sp, #TSPD_C_RT_CTX_X27]
	stp	x29, x30, [sp, #TSPD_C_RT_CTX_X29]

	/* 遗产退出执行下一级BL */
	b	el3_exit
endfunc tspd_enter_sp

	.global tspd_exit_sp
func tspd_exit_sp
	/* 恢复堆栈指针 */
	mov	sp, x0

	/* 恢复x19-x30 */
	ldp	x19, x20, [x0, #(TSPD_C_RT_CTX_X19 - TSPD_C_RT_CTX_SIZE)]
	ldp	x21, x22, [x0, #(TSPD_C_RT_CTX_X21 - TSPD_C_RT_CTX_SIZE)]
	ldp	x23, x24, [x0, #(TSPD_C_RT_CTX_X23 - TSPD_C_RT_CTX_SIZE)]
	ldp	x25, x26, [x0, #(TSPD_C_RT_CTX_X25 - TSPD_C_RT_CTX_SIZE)]
	ldp	x27, x28, [x0, #(TSPD_C_RT_CTX_X27 - TSPD_C_RT_CTX_SIZE)]
	ldp	x29, x30, [x0, #(TSPD_C_RT_CTX_X29 - TSPD_C_RT_CTX_SIZE)]

	/* 返回x1 */
	mov	x0, x1
	ret/* 通过x30返回，将返回到CALL tspd_enter_sp的下一条指令 */
endfunc tspd_exit_sp

```

### TOS与Non-Secure数据交换

TOS与Non-Secure通过Monitor进行数据交换，异常处理代码如下。

```assembly
smc_handler32:
	/* Check whether aarch32 issued an SMC64 */
	tbnz	x0, #FUNCID_CC_SHIFT, smc_prohibited

	/*
	 * Since we're are coming from aarch32, x8-x18 need to be saved as per
	 * SMC32 calling convention. If a lower EL in aarch64 is making an
	 * SMC32 call then it must have saved x8-x17 already therein.
	 */
	stp	x8, x9, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X8]
	stp	x10, x11, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X10]
	stp	x12, x13, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X12]
	stp	x14, x15, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X14]
	stp	x16, x17, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X16]

	/* x4-x7, x18, sp_el0 are saved below */
smc_handler64:
	/*
	 * Populate the parameters for the SMC handler.
	 * We already have x0-x4 in place. x5 will point to a cookie (not used
	 * now). x6 will point to the context structure (SP_EL3) and x7 will
	 * contain flags we need to pass to the handler Hence save x5-x7.
	 *
	 * Note: x4 only needs to be preserved for AArch32 callers but we do it
	 *       for AArch64 callers as well for convenience
	 */
	stp	x4, x5, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X4]
	stp	x6, x7, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X6]

	/* Save rest of the gpregs and sp_el0*/
	save_x18_to_x29_sp_el0

	mov	x5, xzr
	mov	x6, sp

	/* Get the unique owning entity number */
    /* X0用于确定调用的功能
     * b31    调用类型 1->Fast Call 0->Yielding Call
     * b30    标示32bit还是64bit调用
     * b29:24 服务类型
     *           0x0  ARM架构相关
     *           0x1  CPU服务
     *           0x2  SiP服务
     *           0x3  OEM服务
     *           0x4  标准安全服务
     *           0x5  标准Hypervisor服务
     *           0x6  厂商定制的Hypervisor服务
     *     0x07-0x2f  预留
     *     0x30-0x31  Trusted Application Calls
     *     0x32-0x3f  Trusted OS Calls
     * b23:16 在Fast Call时这些比特位必须为0，其他值保留未用
     *        ARMv7传统的Trusted OS在Fast Call时必须为1
     * b15:0  函数号
     */
	ubfx	x16, x0, #FUNCID_OEN_SHIFT, #FUNCID_OEN_WIDTH   /* 服务类型 */
	ubfx	x15, x0, #FUNCID_TYPE_SHIFT, #FUNCID_TYPE_WIDTH /* 调用类型 */
	orr	x16, x16, x15, lsl #FUNCID_OEN_WIDTH

    /* __RT_SVC_DESCS_START__为rt_svc_descs段的起始地址，此段存放服务描述符
     * RT_SVC_DESC_HANDLE为服务句柄在服务描述符结构体中的偏移量
     * 以下代码保存第一个复位句柄的地址到x11中
     */
	adr	x11, (__RT_SVC_DESCS_START__ + RT_SVC_DESC_HANDLE)

	/* Load descriptor index from array of indices */
    /* 因为描述符在内存中布局是乱的通过rt_svc_descs_indices数组来查找描述符在内存中的位置
     * rt_svc_descs_indices为rt_svc_descs的索引
     */
	adr	x14, rt_svc_descs_indices
	ldrb	w15, [x14, x16]

	/*
	 * Restore the saved C runtime stack value which will become the new
	 * SP_EL0 i.e. EL3 runtime stack. It was saved in the 'cpu_context'
	 * structure prior to the last ERET from EL3.
	 */
	ldr	x12, [x6, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]

	/*
	 * Any index greater than 127 is invalid. Check bit 7 for
	 * a valid index
	 */
	tbnz	w15, 7, smc_unknown/* 索引的值不应该大于127 */

	/* Switch to SP_EL0 */
	msr	spsel, #0

	/*
	 * Get the descriptor using the index
	 * x11 = (base + off), x15 = index
	 *
	 * handler = (base + off) + (index << log2(size))
	 */
	lsl	w10, w15, #RT_SVC_SIZE_LOG2/* 计算偏移量 */
	ldr	x15, [x11, w10, uxtw]/* 加载出对应服务的句柄 */

	/*
	 * Save the SPSR_EL3, ELR_EL3, & SCR_EL3 in case there is a world
	 * switch during SMC handling.
	 * TODO: Revisit if all system registers can be saved later.
	 */
	mrs	x16, spsr_el3
	mrs	x17, elr_el3
	mrs	x18, scr_el3
	stp	x16, x17, [x6, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]
	str	x18, [x6, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]

	/* Copy SCR_EL3.NS bit to the flag to indicate caller's security */
	bfi	x7, x18, #0, #1/* x7用于传递给服务函数security state */

	mov	sp, x12

	/*
	 * Call the Secure Monitor Call handler and then drop directly into
	 * el3_exit() which will program any remaining architectural state
	 * prior to issuing the ERET to the desired lower EL.
	 */
#if DEBUG
	cbz	x15, rt_svc_fw_critical_error/* x15等于0说明有错误存在 */
#endif
	blr	x15 /* 调用对应的服务 */

	b	el3_exit/* 异常恢复 */
```

异常处理函数如下

异常处理保护了X4-X7、X18-X30、SP_EL1

X8-X17没有进行保护需要调用SMC服务的程序自行保存

X0作为SMC的功能传递

X1-X4作为SMC服务的参数传递

cookie固定为0

handle把SP_EL3传递给函数，便于修改返回值（现场保护通过SP_EL3完才）

```c
typedef uintptr_t (*rt_svc_handle_t)(uint32_t smc_fid,
				  u_register_t x1,
				  u_register_t x2,
				  u_register_t x3,
				  u_register_t x4,
				  void *cookie,
				  void *handle,
				  u_register_t flags);
```


## Monitor中断

为了灵活配置中断，Monitor使中断处理函数可以配置

### 数据结构

```c
/* 中断类型定义 */
#define INTR_TYPE_S_EL1			0
#define INTR_TYPE_EL3			1
#define INTR_TYPE_NS			2
#define MAX_INTR_TYPES			3	
#define INTR_TYPE_INVAL			MAX_INTR_TYPES

/* 中断函数句柄 */
typedef uint64_t (*interrupt_type_handler_t)(uint32_t id,
					     uint32_t flags,
					     void *handle,
					     void *cookie);

typedef struct intr_type_desc {
  	/* 中断处理函数 */
	interrupt_type_handler_t handler;
  	/* 中断的路由模式：
    	Bit0在Secure时中断要不要路由到EL3
    	Bit1在Non-Secure时中断要不要路由到EL3*/
	uint32_t flags;	
  	/* 记录两种安全状态下（SCR_EL3.FIQ SCR_EL3.IRQ）的掩码（置位用） */
	uint32_t scr_el3[2];
} intr_type_desc_t;

/* 根据中断分类，保存的函数句柄 */
static intr_type_desc_t intr_type_descs[MAX_INTR_TYPES];
```

### 主要方法

#### 获取EL3路由模式控制位

```c
uint32_t get_scr_el3_from_routing_model(uint32_t security_state)
{
	uint32_t scr_el3;
	assert(sec_state_is_valid(security_state));/* 确定是有效的Security state */
  	
  	/* 获取每一种中断的路由控制位 */
	scr_el3 = intr_type_descs[INTR_TYPE_NS].scr_el3[security_state];
	scr_el3 |= intr_type_descs[INTR_TYPE_S_EL1].scr_el3[security_state];
	scr_el3 |= intr_type_descs[INTR_TYPE_EL3].scr_el3[security_state];
	return scr_el3;
}
```

#### 根据中断类型设置路由控制位

```c
/* type为中断类型
 * flags为路由模式
 * 		Bit0在Secure时中断要不要路由到EL3
 *		Bit1在Non-Secure时中断要不要路由到EL3
 */
int32_t set_routing_model(uint32_t type, uint32_t flags)
{
	int32_t rc;

	rc = validate_interrupt_type(type);/* 校验中断类型 */
	if (rc)
		return rc;

	rc = validate_routing_model(type, flags);/* 校验路由模式 */
	if (rc)
		return rc;
	
	intr_type_descs[type].flags = flags;/* 更新路由模式 */
	set_scr_el3_from_rm(type, flags, SECURE);/* 更新scr_el3[SECURE] */
	set_scr_el3_from_rm(type, flags, NON_SECURE);/* 更新scr_el3[NON_SECURE] */

	return 0;
}
```

#### 设置中断处理函数

```c
int32_t register_interrupt_type_handler(uint32_t type,/* 中断类型 */
					interrupt_type_handler_t handler,/* 中断处理句柄 */
					uint32_t flags)/* 路由模式标志 
									* 	Bit0在Secure时中断要不要路由到EL3
 									*	Bit1在Non-Secure时中断要不要路由到EL3*/
{
	int32_t rc;

	/* 确认中断处理句柄不是空指针 */
	if (!handler)
		return -EINVAL;

	/* 校验路由模式有效 */
	if (flags & INTR_TYPE_FLAGS_MASK)
		return -EINVAL;

	/* 中断处理句柄只能注册一次，此处检查中断处理句柄有无注册 */
	if (intr_type_descs[type].handler)
		return -EALREADY;
	
  	/* 根据中断类型设置路由模式 */
	rc = set_routing_model(type, flags);
	if (rc)
		return rc;

	/* 保存中断处理句柄 */
	intr_type_descs[type].handler = handler;

	return 0;
}
```

#### 获取中断处理句柄

```c
interrupt_type_handler_t get_interrupt_type_handler(uint32_t type)
{
  	/* 校验是否是合法的中断类型 */
	if (validate_interrupt_type(type))
		return NULL;
	/* 根据中断类型获取对应的中断处理句柄 */
	return intr_type_descs[type].handler;
}
```

#### 使能对应安全模式对应中断类型的中断

```c
int enable_intr_rm_local(uint32_t type, uint32_t security_state)
{
	uint32_t bit_pos, flag;
	/* 判断有无中断处理句柄，操作没有中断句柄的中断是非法的 */
	assert(intr_type_descs[type].handler);
	
  	/* 获取对应安全模式对应中断类型的中断的路由模式 */
	flag = get_interrupt_rm_flag(intr_type_descs[type].flags,
				security_state);
	/* 获取对应中断的bit_pos，即FIQ IRQ */
	bit_pos = plat_interrupt_type_to_line(type, security_state);
  	/* 写SCR_EL3的路由控制位 SCR_EL3 |= flag << bit_pos */
	cm_write_scr_el3_bit(security_state, bit_pos, flag);

	return 0;
}
```

####  失能对应安全模式对应中断类型的中断

```c
int disable_intr_rm_local(uint32_t type, uint32_t security_state)
{
	uint32_t bit_pos, flag;
	/* 判断有无中断处理句柄，操作没有中断句柄的中断是非法的 */
	assert(intr_type_descs[type].handler);
	
  	/* 获取对应安全模式对应中断类型的中断的路由模式 */
  	/* INTR_DEFAULT_RM = 0
  	 *     Bit0 = 0 Secure中断不路由到EL3
  	 *     Bit1 = 1 Non-Secure中断不路由到EL3
  	 */
	flag = get_interrupt_rm_flag(INTR_DEFAULT_RM, security_state);
  	/* 获取对应中断的bit_pos，即FIQ IRQ */
	bit_pos = plat_interrupt_type_to_line(type, security_state);
  	/* 写SCR_EL3的路由控制位 SCR_EL3 |= flag << bit_pos */
	cm_write_scr_el3_bit(security_state, bit_pos, flag);

	return 0;
}
```

### 中断处理

```assembly
     /* 此宏用于FIQ IRQ处理 */
	.macro	handle_interrupt_exception label
	/* Enable the SError interrupt */
    /* 使能SError中断 */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x0-x30 sp_el1 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]
	bl	save_gp_registers/* 因为此处为函数调用会修改lr(x30),所以lr需要提前保存 */

	/* Save the EL3 system registers needed to return from this exception */
	mrs	x0, spsr_el3 /* 获取进入异常前的程序状态寄存器 */
	mrs	x1, elr_el3  /* 获取程序被中断的异常地址 */
	/* 保存进入异常的程序状态和异常地址 */
	stp	x0, x1, [sp, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]

	/* Switch to the runtime stack i.e. SP_EL0 */
	ldr	x2, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	mov	x20, sp   /* 保护中断现场保护指针，用于修改返回值 对应中断处理函数的handle参数 */
	msr	spsel, #0
	mov	sp, x2 /* 切换到EL0的SP指针 */

	/*
	 * Find out whether this is a valid interrupt type.
	 * If the interrupt controller reports a spurious interrupt then return
	 * to where we came from.
	 */
	bl	plat_ic_get_pending_interrupt_type /* 获取中断类型 */
	cmp	x0, #INTR_TYPE_INVAL
	b.eq	interrupt_exit_\label /* 非法类型直接退出 */

	/*
	 * Get the registered handler for this interrupt type.
	 * A NULL return value could be 'cause of the following conditions:
	 *
	 * a. An interrupt of a type was routed correctly but a handler for its
	 *    type was not registered.
	 *
	 * b. An interrupt of a type was not routed correctly so a handler for
	 *    its type was not registered.
	 *
	 * c. An interrupt of a type was routed correctly to EL3, but was
	 *    deasserted before its pending state could be read. Another
	 *    interrupt of a different type pended at the same time and its
	 *    type was reported as pending instead. However, a handler for this
	 *    type was not registered.
	 *
	 * a. and b. can only happen due to a programming error. The
	 * occurrence of c. could be beyond the control of Trusted Firmware.
	 * It makes sense to return from this exception instead of reporting an
	 * error.
	 */
	bl	get_interrupt_type_handler /* 获取中断处理函数 */
	cbz	x0, interrupt_exit_\label  /* 函数指针为0，直接退出 */
	mov	x21, x0

	mov	x0, #INTR_ID_UNAVAILABLE /* 参数id */

	/* Set the current security state in the 'flags' parameter */
	mrs	x2, scr_el3
	ubfx	x1, x2, #0, #1    /* 获取Security state，对应中断处理函数的flags参数 */

	/* Restore the reference to the 'handle' i.e. SP_EL3 */
	mov	x2, x20 /* x2指向SP_EL3 用于修改返回值*/

	/* x3 will point to a cookie (not used now) */
	mov	x3, xzr

	/* Call the interrupt type handler */
	blr	x21

interrupt_exit_\label:
	/* Return from exception, possibly in a different security state */
	b	el3_exit

	.endm
```



# BL1分析

## 功能概要

BL1主要负责对BL2进行认证加载并执行，并且提供SMC中断服务，便于执行在EL1-Secure的BL2能给把BL31加载到EL3。

## 主过程

BL1的入口代码位于`bl1/aarch64/bl1_entrypoint.S`文件中，代码注释如下。

```asm
	.globl	bl1_entrypoint
func bl1_entrypoint /* BL1的程序入口 */
	 /* EL3通用的初始化宏 */
	el3_entrypoint_common					\
		_set_endian=1		/* 大小端初始化 */\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	/* 热启动分支执行 */\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU /* 次处理器进入低功耗模式 */\
		_init_memory=1	/* 内存初始化 */\
		_init_c_runtime=1	/* 初始化堆栈（SP）等 */\
		_exception_vectors=bl1_exceptions /* 设置异常向量 */

	/*--------------------------------------------------
	 * bl1_early_platform_setup
	 * 初始化看梦狗、串口（用于输出调试信息）
	 * 更新BL1内存布局到bl1_tzram_layout
	 *--------------------------------------------------
	 */
	bl	bl1_early_platform_setup

	/*--------------------------------------------------
	 * bl1_plat_arch_setup
	 * 根据内存布局设置页表并使能MMU
	 *--------------------------------------------------
	 */
	bl	bl1_plat_arch_setup

	/* --------------------------------------------------
	 * 转到主程序
	 * --------------------------------------------------
	 */
	bl	bl1_main

	/* --------------------------------------------------
	 * 退出异常，执行下一级BL
	 * --------------------------------------------------
	 */
	b	el3_exit
endfunc bl1_entrypoint
```

bl1\_main是bl1的主程序，位于`bl1/bl1_main.c`中

```c
void bl1_main(void)
{
	unsigned int image_id;

	/* 输出一些提示信息 */
	NOTICE(FIRMWARE_WELCOME_STR);
	NOTICE("BL1: %s\n", version_string);
	NOTICE("BL1: %s\n", build_message);

	INFO("BL1: RAM %p - %p\n", (void *)BL1_RAM_BASE,
					(void *)BL1_RAM_LIMIT);

	print_errata_status();

	/*
	 * 异常检测，查看汇编中初始化是否正常
	 */
#if DEBUG
	u_register_t val;
#ifdef AARCH32
	val = read_sctlr();
#else
	val = read_sctlr_el3();
#endif
	assert(val & SCTLR_M_BIT);
	assert(val & SCTLR_C_BIT);
	assert(val & SCTLR_I_BIT);
	val = (read_ctr_el0() >> CTR_CWG_SHIFT) & CTR_CWG_MASK;
	if (val != 0)
		assert(CACHE_WRITEBACK_GRANULE == SIZE_FROM_LOG2_WORDS(val));
	else
		assert(CACHE_WRITEBACK_GRANULE <= MAX_CACHE_LINE_SIZE);
#endif

	/* 设置下一级EL可以运行在64bit */
	bl1_arch_setup();

#if TRUSTED_BOARD_BOOT
	/* 初始化认证模块 */
	auth_mod_init();
#endif /* TRUSTED_BOARD_BOOT */

	/* 具体硬件相关设置（IO外设等） */
	bl1_platform_setup();

	/* 获取下一级要执行的镜像id */
	image_id = bl1_plat_get_next_image_id();

	if (image_id == BL2_IMAGE_ID)
		bl1_load_bl2();/* 加载BL2到内存 */
	else
		NOTICE("BL1-FWU: *******FWU Process Started*******\n");

	/* 执行环境配置，在程序结束后可以执行下一个镜像 */
	bl1_prepare_next_image(image_id);
}
```

## 中断服务

BL1异常服务程序位于`bl1/aarch64/bl1_exceptions.S`中

```assembly
/*
 * BL1的异常向量表
 * plat_report_exception用与指示异常与具体架构有关
 * no_ret 是一个bl的宏，用于指示被调用的函数没有返回值
 * plat_panic_handler是一个死循环，在程序出错时停止程序的运行
 * check_vecror_size用于检查代码有没有越界(一个异常向量只有128Byte)
 * BL1只处理低异常等级的SMC调用，其他异常全部非法
 */
	.globl	bl1_exceptions

vector_base bl1_exceptions

	/* -----------------------------------------------------
	 * Current EL with SP0 : 0x0 - 0x200
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionSP0
	mov	x0, #SYNC_EXCEPTION_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SynchronousExceptionSP0

vector_entry IrqSP0
	mov	x0, #IRQ_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqSP0

vector_entry FiqSP0
	mov	x0, #FIQ_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqSP0

vector_entry SErrorSP0
	mov	x0, #SERROR_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorSP0

	/* -----------------------------------------------------
	 * Current EL with SPx: 0x200 - 0x400
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionSPx
	mov	x0, #SYNC_EXCEPTION_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SynchronousExceptionSPx

vector_entry IrqSPx
	mov	x0, #IRQ_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqSPx

vector_entry FiqSPx
	mov	x0, #FIQ_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqSPx

vector_entry SErrorSPx
	mov	x0, #SERROR_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorSPx

	/* -----------------------------------------------------
	 * Lower EL using AArch64 : 0x400 - 0x600
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionA64
	/* 使能SError中断 */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x30下面需要使用x30作临时寄存器分析异常的原因 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]

	/* Expect only SMC exceptions */
	mrs	x30, esr_el3 /* 获取异常的原因 */
	ubfx	x30, x30, #ESR_EC_SHIFT, #ESR_EC_LENGTH
	cmp	x30, #EC_AARCH64_SMC
	b.ne	unexpected_sync_exception /* 报告非法异常（只处理AArch64 SMC调用） */

	b	smc_handler64 /* 处理低异常等级SMC调用 */
	check_vector_size SynchronousExceptionA64

vector_entry IrqA64
	mov	x0, #IRQ_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqA64

vector_entry FiqA64
	mov	x0, #FIQ_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqA64

vector_entry SErrorA64
	mov	x0, #SERROR_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorA64

	/* -----------------------------------------------------
	 * Lower EL using AArch32 : 0x600 - 0x800
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionA32
	mov	x0, #SYNC_EXCEPTION_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SynchronousExceptionA32

vector_entry IrqA32
	mov	x0, #IRQ_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqA32

vector_entry FiqA32
	mov	x0, #FIQ_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqA32

vector_entry SErrorA32
	mov	x0, #SERROR_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorA32


func smc_handler64

	/* RUN_IMAGE SMC调用，用于运行低异常等级的镜像，并执行在EL3，
	 * 即不返回调用的位置，所以需要特别处理 */
	mov	x30, #BL1_SMC_RUN_IMAGE
	cmp	x30, x0
	b.ne	smc_handler/* 其他SMC调用 */

	/* 确定在secure world执行RUN_IMAGE SMC调用 */
	mrs	x30, scr_el3
	tst	x30, #SCR_NS_BIT
	b.ne	unexpected_sync_exception/* 报告异常 */

	/* 加载出异常退出时保存的SP，切换到SP_EL0 */
	ldr	x30, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	msr	spsel, #0
	mov	sp, x30

	/*
	 * bl1_print_next_bl_ep_info用于打印要执行的镜像文件的地址参数
	 * 这些参数保存在entry_point_info_t结构体中
	 * x1为指针指向entry_point_info_t
	 */
	mov	x20, x1
	mov	x0, x20
	bl	bl1_print_next_bl_ep_info

	ldp	x0, x1, [x20, #ENTRY_POINT_INFO_PC_OFFSET]/* 获取pc、spsr */
	msr	elr_el3, x0/* 修改返回地址，用来在异常退出时执行指定的镜像文件 */
	msr	spsr_el3, x1/* 修改返回的cpsr,可以修改异常等级 */
	ubfx	x0, x1, #MODE_EL_SHIFT, #2
	cmp	x0, #MODE_EL3/* 执行的镜像必须为EL3，为后面的代码放开权限 */
	b.ne	unexpected_sync_exception

	bl	disable_mmu_icache_el3
	tlbi	alle3

#if SPIN_ON_BL1_EXIT
	bl	print_debug_loop_message
debug_loop:
	b	debug_loop
#endif

	mov	x0, x20
	bl	bl1_plat_prepare_exit
	/* 从x1指定的结构体中获取参数，并退出异常 */
	ldp	x6, x7, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x30)]
	ldp	x4, x5, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x20)]
	ldp	x2, x3, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x10)]
	ldp	x0, x1, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x0)]
	eret
endfunc smc_handler64

unexpected_sync_exception:
	mov	x0, #SYNC_EXCEPTION_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler

smc_handler:
	bl	save_gp_registers/* 保存x0-x29，LR(x30)在之前已经保存过了 */
	mov	x5, xzr/* x5作为cookie参数传递给异常函数现在未用 */
	mov	x6, sp /* x6为作为handle传递给异常处理函数，x6为堆栈的指针，便于遗产函数设置返回值 */

	/* 加载出上一次异常退出的SP指针 */
	ldr	x12, [x6, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]

	/* ---------------------------------------------
	 * Switch back to SP_EL0 for the C runtime stack.
	 * ---------------------------------------------
	 */
	msr	spsel, #0
	mov	sp, x12

	/* -----------------------------------------------------
	 * 保存spsr_el3、elr_el3、scr_el3寄存器
	 * 在el3_exit时可以退出到进入遗产的位置
	 * -----------------------------------------------------
	 */
	mrs	x16, spsr_el3
	mrs	x17, elr_el3
	mrs	x18, scr_el3
	stp	x16, x17, [x6, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]
	str	x18, [x6, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]

	/* 传递给异常处理函数的参数flag，标记之前是否在Secure Word */
	bfi	x7, x18, #0, #1

	/* -----------------------------------------------------
	 * 跳转到证实的异常处理函数
	 * -----------------------------------------------------
	 */
	bl	bl1_smc_handler

	/* -----------------------------------------------------
	 * 遗产调用返回，返回到调用位置
	 * -----------------------------------------------------
	 */
	b	el3_exit
```

BL1使用SP_EL0作为其C程序运行的堆栈，在进入异常时默认认为SPSEL=1，使用的是SP_EL3。这样，保存的线程在SP_EL3中。切换到SP_EL0执行程序，并通过SP_EL3来修改堆栈。

# BL2分析

## 功能概要

BL2主要负责对其他所有BL进行认证和加载，并执行EL31。

## 主过程

BL2入口位于`bl2/aarch64/bl2_entrypoint.S`中

```assembly
	.globl	bl2_entrypoint



func bl2_entrypoint
	mov	x20, x1 /* x1保存了内存布局信息 */

    /* 设置异常处理函数 */
	adr	x0, early_exceptions
	msr	vbar_el1, x0
	isb

	/* ---------------------------------------------
	 * 使能异常
	 * ---------------------------------------------
	 */
	msr	daifclr, #DAIF_ABT_BIT

	/* ---------------------------------------------
	 * 使能指令缓存，使能堆栈数据访问对齐检查
	 * ---------------------------------------------
	 */
	mov	x1, #(SCTLR_I_BIT | SCTLR_A_BIT | SCTLR_SA_BIT)
	mrs	x0, sctlr_el1
	orr	x0, x0, x1
	msr	sctlr_el1, x0
	isb

	/* ---------------------------------------------
	 * 失效内存
	 * ---------------------------------------------
	 */
	adr	x0, __RW_START__
	adr	x1, __RW_END__
	sub	x1, x1, x0
	bl	inv_dcache_range

	/* ---------------------------------------------
	 * BSS内存初始化
	 * ---------------------------------------------
	 */
	ldr	x0, =__BSS_START__
	ldr	x1, =__BSS_SIZE__
	bl	zeromem16

#if USE_COHERENT_MEM
	ldr	x0, =__COHERENT_RAM_START__
	ldr	x1, =__COHERENT_RAM_UNALIGNED_SIZE__
	bl	zeromem16
#endif

	/* --------------------------------------------
	 * 设置SP指针
	 * --------------------------------------------
	 */
	bl	plat_set_my_stack

	/* ---------------------------------------------
	 * 串口初始化，更新内存布局信息，并初始化页表使能mmu
	 * ---------------------------------------------
	 */
	mov	x0, x20
	bl	bl2_early_platform_setup
	bl	bl2_plat_arch_setup

	/* ---------------------------------------------
	 * 跳转到主函数（通过SMC执行下一级BL不会返回）
	 * ---------------------------------------------
	 */
	bl	bl2_main

	/* ---------------------------------------------
	 * 下面的代码不会执行
	 * ---------------------------------------------
	 */
	no_ret	plat_panic_handler

endfunc bl2_entrypoint
```

bl2\_main为bl2的主程序，位于bl2/bl2_main.c中

```c
void bl2_main(void)
{
	entry_point_info_t *next_bl_ep_info;
	/* 输出提示信息 */
	NOTICE("BL2: %s\n", version_string);
	NOTICE("BL2: %s\n", build_message);

	/* 初始化，这里开启FP/SIMD的访问权限 */
	bl2_arch_setup();

#if TRUSTED_BOARD_BOOT
	/*
	 * 初始化认证模块
	 * 初始化加密库（crypto_mod），加密库可以用于校验签名和哈希
	 * 初始化镜像解析模块（img_parser_mod），用于校验镜像完整性以及从镜像中提取内容
	 */
	auth_mod_init();
#endif /* TRUSTED_BOARD_BOOT */

	/* 此处不不只是加载BL2，此处会加载校验所有的BL */
	next_bl_ep_info = bl2_load_images();

#ifdef AARCH32
	/*
	 * For AArch32 state BL1 and BL2 share the MMU setup.
	 * Given that BL2 does not map BL1 regions, MMU needs
	 * to be disabled in order to go back to BL1.
	 */
	disable_mmu_icache_secure();
#endif /* AARCH32 */

	 /* 通过SMC系统调用运行下一级BL，下一级BL将跳转到EL3 */
	smc(BL1_SMC_RUN_IMAGE, (unsigned long)next_bl_ep_info, 0, 0, 0, 0, 0, 0);
}
```



## 中断服务

# BL31分析

## 功能概要

BL31主要提供了SMC服务框架，并于SMC服务的添加扩展。并执行下一级BL

## 主过程

BL31入口位于`bl31/aarch64/bl31_entrypoint.S`中

```assembly
func bl31_entrypoint
#if !RESET_TO_BL31
     /* 由前端bootloader引导进入，这时cpu会通过X0/X1传递信息
      * X0指向`bl31_params`结构体
      * X1指向特定架构相关的结构体
      */
	mov	x20, x0
	mov	x21, x1

     /* 因为前端bootloader已经初始化了一部分类容，这里只要初始化当前内核的运行环境 */
	el3_entrypoint_common					\
		_set_endian=0					\
		_warm_boot_mailbox=0				\
		_secondary_cold_boot=0				\
		_init_memory=0					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions

     /* 恢复之前保存的参数 */
	mov	x0, x20
	mov	x1, x21
#else
     /* 由前端cpu复位进入，这时需要重新初始化，非主核cpu不需要初始化 */
	el3_entrypoint_common					\
		_set_endian=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions

     /* 前端没有参数传递进来，给指针赋值0 */
	mov	x0, 0
	mov	x1, 0
#endif /* RESET_TO_BL31 */

	bl	bl31_early_platform_setup/* 初始化cpu内部初始化，初始化执行环境 */
	bl	bl31_plat_arch_setup/* 初始化页表 */

	/* ---------------------------------------------
	 * Jump to main function.
	 * ---------------------------------------------
	 */
	bl	bl31_main
     /* 把BL31的data段同步到内存 */
	adr	x0, __DATA_START__
	adr	x1, __DATA_END__
	sub	x1, x1, x0
	bl	clean_dcache_range

    /* 把BL31的bss段同步到内存 */
	adr	x0, __BSS_START__
	adr	x1, __BSS_END__
	sub	x1, x1, x0
	bl	clean_dcache_range

    /* 退出异常，并执行下一个镜像 */
	b	el3_exit
endfunc bl31_entrypoint
```

BL31主程序位于`bl31/bl31_main.c`中

```c
void bl31_main(void)
{
	NOTICE("BL31: %s\n", version_string);
	NOTICE("BL31: %s\n", build_message);

	/* 系统相关的初始化 */
	bl31_platform_setup();
	bl31_lib_init();

	/* 初始化系统服务，并构造索引rt_svc_descs_indices */
	INFO("BL31: Initializing runtime services\n");
	runtime_svc_init();

  	/* 判对有无bl31，执行对应的初始化 */
	if (bl32_init) {
		INFO("BL31: Initializing BL32\n");
		(*bl32_init)();
	}

	 /* 初始化下一级BL的执行环境，以便异常返回时执行下一级BL */
	bl31_prepare_next_image_entry();

	 /* 从BL31退出前执行与特定平台相关的初始化工作 */
	bl31_plat_runtime_setup();
}
```



## 中断服务

bl31中断服务程序位于`bl31/aarch64/runtime_exceptions.S`中

```assembly
	.globl	runtime_exceptions

	/* ---------------------------------------------------------------------
	 * This macro handles Synchronous exceptions.
	 * Only SMC exceptions are supported.
	 * ---------------------------------------------------------------------
	 */
     /* 同步异常处理只处理smc相关的调用 */
	.macro	handle_sync_exception
	/* Enable the SError interrupt */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x30下面要分析异常来源 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]

#if ENABLE_RUNTIME_INSTRUMENTATION
	/*
	 * Read the timestamp value and store it in per-cpu data. The value
	 * will be extracted from per-cpu data by the C level SMC handler and
	 * saved to the PMF timestamp region.
	 */
	mrs	x30, cntpct_el0/* 时钟 */
	str	x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X29]
	mrs	x29, tpidr_el3 /* tpidr_el3 cpu标示 */
	str	x30, [x29, #CPU_DATA_PMF_TS0_OFFSET]
	ldr	x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X29]
#endif
	/* 获取exceptions code */
	mrs	x30, esr_el3
	ubfx	x30, x30, #ESR_EC_SHIFT, #ESR_EC_LENGTH

	/* Handle SMC exceptions separately from other synchronous exceptions */
	cmp	x30, #EC_AARCH32_SMC
	b.eq	smc_handler32 /* 32bit SMC调用 */

	cmp	x30, #EC_AARCH64_SMC
	b.eq	smc_handler64 /* 64bit SMC调用 */

	/* Other kinds of synchronous exceptions are not handled */
	no_ret	report_unhandled_exception
	.endm


	/* ---------------------------------------------------------------------
	 * This macro handles FIQ or IRQ interrupts i.e. EL3, S-EL1 and NS
	 * interrupts.
	 * ---------------------------------------------------------------------
	 */
	.macro	handle_interrupt_exception label
	/* Enable the SError interrupt */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x0-x30 sp_el1 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]
	bl	save_gp_registers/* 因为此处为函数调用会修改lr(x30),所以lr需要提前保存 */

	/* Save the EL3 system registers needed to return from this exception */
	mrs	x0, spsr_el3 /* 获取进入异常前的程序状态寄存器 */
	mrs	x1, elr_el3  /* 获取程序被中断的异常地址 */
	/* 保存进入异常的程序状态和异常地址 */
	stp	x0, x1, [sp, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]

	/* Switch to the runtime stack i.e. SP_EL0 */
	ldr	x2, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	mov	x20, sp
	msr	spsel, #0
	mov	sp, x2 /* 切换到EL0的SP指针 */

	/*
	 * Find out whether this is a valid interrupt type.
	 * If the interrupt controller reports a spurious interrupt then return
	 * to where we came from.
	 */
	bl	plat_ic_get_pending_interrupt_type
	cmp	x0, #INTR_TYPE_INVAL
	b.eq	interrupt_exit_\label

	/*
	 * Get the registered handler for this interrupt type.
	 * A NULL return value could be 'cause of the following conditions:
	 *
	 * a. An interrupt of a type was routed correctly but a handler for its
	 *    type was not registered.
	 *
	 * b. An interrupt of a type was not routed correctly so a handler for
	 *    its type was not registered.
	 *
	 * c. An interrupt of a type was routed correctly to EL3, but was
	 *    deasserted before its pending state could be read. Another
	 *    interrupt of a different type pended at the same time and its
	 *    type was reported as pending instead. However, a handler for this
	 *    type was not registered.
	 *
	 * a. and b. can only happen due to a programming error. The
	 * occurrence of c. could be beyond the control of Trusted Firmware.
	 * It makes sense to return from this exception instead of reporting an
	 * error.
	 */
	bl	get_interrupt_type_handler
	cbz	x0, interrupt_exit_\label
	mov	x21, x0

	mov	x0, #INTR_ID_UNAVAILABLE

	/* Set the current security state in the 'flags' parameter */
	mrs	x2, scr_el3
	ubfx	x1, x2, #0, #1

	/* Restore the reference to the 'handle' i.e. SP_EL3 */
	mov	x2, x20

	/* x3 will point to a cookie (not used now) */
	mov	x3, xzr

	/* Call the interrupt type handler */
	blr	x21

interrupt_exit_\label:
	/* Return from exception, possibly in a different security state */
	b	el3_exit

	.endm


	.macro save_x18_to_x29_sp_el0
	stp	x18, x19, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X18]
	stp	x20, x21, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X20]
	stp	x22, x23, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X22]
	stp	x24, x25, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X24]
	stp	x26, x27, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X26]
	stp	x28, x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X28]
	mrs	x18, sp_el0
	str	x18, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_SP_EL0]
	.endm


vector_base runtime_exceptions

	/* ---------------------------------------------------------------------
	 * Current EL with SP_EL0 : 0x0 - 0x200
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_sp_el0
	/* We don't expect any synchronous exceptions from EL3 */
	no_ret	report_unhandled_exception
	check_vector_size sync_exception_sp_el0

vector_entry irq_sp_el0
	/*
	 * EL3 code is non-reentrant. Any asynchronous exception is a serious
	 * error. Loop infinitely.
	 */
	no_ret	report_unhandled_interrupt
	check_vector_size irq_sp_el0


vector_entry fiq_sp_el0
	no_ret	report_unhandled_interrupt
	check_vector_size fiq_sp_el0


vector_entry serror_sp_el0
	no_ret	report_unhandled_exception
	check_vector_size serror_sp_el0

	/* ---------------------------------------------------------------------
	 * Current EL with SP_ELx: 0x200 - 0x400
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_sp_elx
	/*
	 * This exception will trigger if anything went wrong during a previous
	 * exception entry or exit or while handling an earlier unexpected
	 * synchronous exception. There is a high probability that SP_EL3 is
	 * corrupted.
	 */
	no_ret	report_unhandled_exception
	check_vector_size sync_exception_sp_elx

vector_entry irq_sp_elx
	no_ret	report_unhandled_interrupt
	check_vector_size irq_sp_elx

vector_entry fiq_sp_elx
	no_ret	report_unhandled_interrupt
	check_vector_size fiq_sp_elx

vector_entry serror_sp_elx
	no_ret	report_unhandled_exception
	check_vector_size serror_sp_elx

	/* ---------------------------------------------------------------------
	 * Lower EL using AArch64 : 0x400 - 0x600
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_aarch64
	/*
	 * This exception vector will be the entry point for SMCs and traps
	 * that are unhandled at lower ELs most commonly. SP_EL3 should point
	 * to a valid cpu context where the general purpose and system register
	 * state can be saved.
	 */
	handle_sync_exception
	check_vector_size sync_exception_aarch64

vector_entry irq_aarch64
	handle_interrupt_exception irq_aarch64
	check_vector_size irq_aarch64

vector_entry fiq_aarch64
	handle_interrupt_exception fiq_aarch64
	check_vector_size fiq_aarch64

vector_entry serror_aarch64
	/*
	 * SError exceptions from lower ELs are not currently supported.
	 * Report their occurrence.
	 */
	no_ret	report_unhandled_exception
	check_vector_size serror_aarch64

	/* ---------------------------------------------------------------------
	 * Lower EL using AArch32 : 0x600 - 0x800
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_aarch32
	/*
	 * This exception vector will be the entry point for SMCs and traps
	 * that are unhandled at lower ELs most commonly. SP_EL3 should point
	 * to a valid cpu context where the general purpose and system register
	 * state can be saved.
	 */
	handle_sync_exception
	check_vector_size sync_exception_aarch32

vector_entry irq_aarch32
	handle_interrupt_exception irq_aarch32
	check_vector_size irq_aarch32

vector_entry fiq_aarch32
	handle_interrupt_exception fiq_aarch32
	check_vector_size fiq_aarch32

vector_entry serror_aarch32
	/*
	 * SError exceptions from lower ELs are not currently supported.
	 * Report their occurrence.
	 */
	no_ret	report_unhandled_exception
	check_vector_size serror_aarch32


	/* ---------------------------------------------------------------------
	 * The following code handles secure monitor calls.
	 * Depending upon the execution state from where the SMC has been
	 * invoked, it frees some general purpose registers to perform the
	 * remaining tasks. They involve finding the runtime service handler
	 * that is the target of the SMC & switching to runtime stacks (SP_EL0)
	 * before calling the handler.
	 *
	 * Note that x30 has been explicitly saved and can be used here
	 * ---------------------------------------------------------------------
	 */
func smc_handler
smc_handler32:
	/* Check whether aarch32 issued an SMC64 */
	tbnz	x0, #FUNCID_CC_SHIFT, smc_prohibited

	/*
	 * Since we're are coming from aarch32, x8-x18 need to be saved as per
	 * SMC32 calling convention. If a lower EL in aarch64 is making an
	 * SMC32 call then it must have saved x8-x17 already therein.
	 */
	stp	x8, x9, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X8]
	stp	x10, x11, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X10]
	stp	x12, x13, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X12]
	stp	x14, x15, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X14]
	stp	x16, x17, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X16]

	/* x4-x7, x18, sp_el0 are saved below */

smc_handler64:
	/*
	 * Populate the parameters for the SMC handler.
	 * We already have x0-x4 in place. x5 will point to a cookie (not used
	 * now). x6 will point to the context structure (SP_EL3) and x7 will
	 * contain flags we need to pass to the handler Hence save x5-x7.
	 *
	 * Note: x4 only needs to be preserved for AArch32 callers but we do it
	 *       for AArch64 callers as well for convenience
	 */
	stp	x4, x5, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X4]
	stp	x6, x7, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X6]

	/* Save rest of the gpregs and sp_el0*/
	save_x18_to_x29_sp_el0

	mov	x5, xzr
	mov	x6, sp

	/* Get the unique owning entity number */
    /* X0用于确定调用的功能
     * b31    调用类型 1->Fast Call 0->Yielding Call
     * b30    标示32bit还是64bit调用
     * b29:24 服务类型
     *           0x0  ARM架构相关
     *           0x1  CPU服务
     *           0x2  SiP服务
     *           0x3  OEM服务
     *           0x4  标准安全服务
     *           0x5  标准Hypervisor服务
     *           0x6  厂商定制的Hypervisor服务
     *     0x07-0x2f  预留
     *     0x30-0x31  Trusted Application Calls
     *     0x32-0x3f  Trusted OS Calls
     * b23:16 在Fast Call时这些比特位必须为0，其他值保留未用
     *        ARMv7传统的Trusted OS在Fast Call时必须为1
     * b15:0  函数号
     */
	ubfx	x16, x0, #FUNCID_OEN_SHIFT, #FUNCID_OEN_WIDTH   /* 服务类型 */
	ubfx	x15, x0, #FUNCID_TYPE_SHIFT, #FUNCID_TYPE_WIDTH /* 调用类型 */
	orr	x16, x16, x15, lsl #FUNCID_OEN_WIDTH

    /* __RT_SVC_DESCS_START__为rt_svc_descs段的起始地址，此段存放服务描述符
     * RT_SVC_DESC_HANDLE为服务句柄在服务描述符结构体中的偏移量
     * 以下代码保存第一个复位句柄的地址到x11中
     */
	adr	x11, (__RT_SVC_DESCS_START__ + RT_SVC_DESC_HANDLE)

	/* Load descriptor index from array of indices */
    /* 因为描述符在内存中布局是乱的通过rt_svc_descs_indices数组来查找描述符在内存中的位置
     * rt_svc_descs_indices为rt_svc_descs的索引
     */
	adr	x14, rt_svc_descs_indices
	ldrb	w15, [x14, x16]

	/*
	 * Restore the saved C runtime stack value which will become the new
	 * SP_EL0 i.e. EL3 runtime stack. It was saved in the 'cpu_context'
	 * structure prior to the last ERET from EL3.
	 */
	ldr	x12, [x6, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]

	/*
	 * Any index greater than 127 is invalid. Check bit 7 for
	 * a valid index
	 */
	tbnz	w15, 7, smc_unknown/* 索引的值不应该大于127 */

	/* Switch to SP_EL0 */
	msr	spsel, #0

	/*
	 * Get the descriptor using the index
	 * x11 = (base + off), x15 = index
	 *
	 * handler = (base + off) + (index << log2(size))
	 */
	lsl	w10, w15, #RT_SVC_SIZE_LOG2/* 计算偏移量 */
	ldr	x15, [x11, w10, uxtw]/* 加载出对应服务的句柄 */

	/*
	 * Save the SPSR_EL3, ELR_EL3, & SCR_EL3 in case there is a world
	 * switch during SMC handling.
	 * TODO: Revisit if all system registers can be saved later.
	 */
	mrs	x16, spsr_el3
	mrs	x17, elr_el3
	mrs	x18, scr_el3
	stp	x16, x17, [x6, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]
	str	x18, [x6, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]

	/* Copy SCR_EL3.NS bit to the flag to indicate caller's security */
	bfi	x7, x18, #0, #1/* x7用于传递给服务函数security state */

	mov	sp, x12

	/*
	 * Call the Secure Monitor Call handler and then drop directly into
	 * el3_exit() which will program any remaining architectural state
	 * prior to issuing the ERET to the desired lower EL.
	 */
#if DEBUG
	cbz	x15, rt_svc_fw_critical_error/* x15等于0说明有错误存在 */
#endif
	blr	x15 /* 调用对应的服务 */

	b	el3_exit/* 异常恢复 */

smc_unknown:
	/*
	 * Here we restore x4-x18 regardless of where we came from. AArch32
	 * callers will find the registers contents unchanged, but AArch64
	 * callers will find the registers modified (with stale earlier NS
	 * content). Either way, we aren't leaking any secure information
	 * through them.
	 */
	mov	w0, #SMC_UNK
	b	restore_gp_registers_callee_eret

smc_prohibited:
	ldr	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]
	mov	w0, #SMC_UNK
	eret

rt_svc_fw_critical_error:
	/* Switch to SP_ELx */
	msr	spsel, #1
	no_ret	report_unhandled_exception
endfunc smc_handler
```

