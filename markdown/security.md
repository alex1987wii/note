[TOC]
# 前言
## 源由及目的
在公司为客户提供的服务中，软件服务占据重要地位，而软件服务又以设备端软件服务为主，软件的常年维护更新也花费了公司不少人力财力，因而有必要对不同付费层级的客户提供不同的服务范围，在这里体现为对软件的更新限制，限制条件由业务人员根据不同的客户去指定，限制的规则由业务提出需求，系统Team去实施。
目前业务提出了对MSN和model number进行判断后决定是否更新的需求，系统Team考虑到该需求的完整性，进一步提出了对镜像版本进行限定的需求。
本文用于描述该功能需求的设计与实现。

## 文档结构
本文首先进行需求的整理，除了业务提出的需求外，也对其它需求来源进行了分析展开。然后，针对需求提出设计方案并进行对比。确定方案后，对方案进一步分析确定软件架构及功能模组。最后对软件提供的接口进行了描述说明，并列举测试用例。
按照该逻辑思路，本文将分为以下几个章节：
+ 需求整理
+ 方案设计
+ 软件架构
+ 接口描述
+ 测试用例

# 需求整理
## 功能需求
目前的需求描述是对允许指定MSN序列的设备更新，不允许指定model number的设备更新。由于一次更新事务可以用三元组`（设备，镜像，更新列表）`来表示，其中MSN序列号和model number的序列号都是针对设备的，而更新列表对于一键更新工具而言，可以认为是确定的，为完整的实现该功能，系统Team提出了对镜像版本进行限定的需求。
另外，为了防止用户篡改db文件中的MSN和model number序列号，做好安全工作，db文件中的记录和镜像均需要进行加密处理。
本更新工具可能会由其它UI工具调用，而该UI工具需要实时获取烧录进展状态，因而本更新工具需要将关键事件上报，让UI工具能够及时响应。
最终，新增功能需求整理如下：
+ MSN限定
+ model number限定
+ image版本限定
+ 加解密处理
+ 事件上报功能

## 代码维护需求
除了功能需求外，为了保证后续维护工作的顺利开展，软件结构及代码风格需要进行一定规范，保证各模块功能的独立性与完整性。这里的主要依据是设计模式的6大原则以及编码规范，这里不一一展开。

## 调试及测试需求
为了方便前期的调试和后期的代码维护，各功能模块对外接口函数中应该有关键信息的打印（调试版本中）,便于在出现问题的时候，能够获取关键信息并快速定位问题所在。

# 方案设计
## 分析
之前有提供过一个MSN限定的更新工具，需求基本类似，但没有提供对model number和image版本限定的功能。这个MSN限定更新的功能还需要另外两个工具协助来完成，因而，这三个工具是作为一个功能整体提供的。另外的两个工具分别是镜像加密工具和msn的db生成工具，其作用如下：
1. 镜像加密工具
镜像加密工具用于将镜像文件按预定的加密方式生成加密镜像，主要是为了安全保密。
2. msn的db生成工具
更新工具在更新前需要对msn进行验证，而验证的依据就是msn列表，所以这个列表对msn限定的更新工具是非常关键的，但这个列表在工具编译时是不确定的，而是由业务根据实际客户指定列表，使用该工具生成db文件。msn db文件的生成与解析，db生成工具来生成，更新工具来解析，显然二者应该协定，保证文件生成和解析的一致性，从而引申出msn db文件的格式定义（若有兴趣可查阅msn限定更新工具的概念设计文档）。

从该功能的逻辑流程上来看，可以分以下两大步：
1. 更新包的提供
业务根据特定的客户
a. 选择需要提供的原始镜像，并使用我们提供的工具进行加密处理，并将加密后的镜像加入更新包中。
b. 选择可以进行更新的msn序列号，并将其写入一个txt文件中（一行一个记录）,使用我们提供的msn的db生成工具生成db文件（文件名需要为“msn.db"），一并放入更新包中。
c. 配置文件的配置，主要是在配置文件中指定需要更新的镜像名。
2. MSN限定更新工具的处理流程
用户（也可能是UI工具）在使用更新工具，并点击了更新按钮以后，工具首先从设备端获取设备的msn序列号, 同时解析本地msn.db文件，并进行比对验证处理（中间出错，如网络错误，msn.db文件丢失，或被篡改，都将导致更新失败，也就是拒绝更新）确认通过后，将进行镜像的解密，并使用解密的镜像更新设备，后续的步骤基本和普通的一键更新流程一致（需要注意的是这些临时文件需要在更新完成后及时清除）。

## 方案一：以原来的MSN限定工具为模板新增db文件及更新工具的逻辑
+ 从需求上来看，本次提出的只是新增一、两项数据的校验，其它逻辑保持不变。所以，直接以原来的工具为模块新增db文件，并小幅修改更新工具逻辑即可，工作量相对较小。但如此一来，就需要新增model number和image版本的db文件格式定义（可参考msn的db文件格式修改实现）及其生成工具，业务发布时需要额外生成这两个db文件。
+ 从逻辑层面上来看，无论是数据结构还是使用场景，这几个db文件是有很大共性的，如果使用多份格式定义，多份代码实现，哪一天需要修改其中的处理逻辑时就需要对三处同步进行修改，一不小心就容易出现处理逻辑不一致而导致隐藏的BUG，测试的工作量也加大了。
+ 从更新工具的角度看，如果新增了这3个db，限定更新的逻辑又不允许这3个db文件缺失，如果只要对msn进行限定更新呢？或是只对model number进行限定，只对image版本进行限定？每种情况下这3个db都不可缺少吗？还是说为每个使用场景分出一个分支来生成特定的更新工具，只对限定的db进行检查？事实上，由于各项目的差异原因（MSN长度不一样），db生成工具目前对u3和g4_bba是进行了区分处理的（即两个项目的msn db生成工具不能混用），这已经带来了一些不便和困扰。如果再进一步分支，将对代码的管理和工具的使用带来巨大挑战。

## 方案二：抽象需求并重新实现
为了解决方案一中的缺点，需要：
1. 对限定更新的功能需求进行抽象，并使用一份代码来维护，更新工具中抽象出一个模块来完成该功能，避免了更新工具直接与具体的功能需求打交道，减小了耦合。
2. 各db文件整合为一个文件，对于限定更新工具而言，这个db文件就是限定更新的数据来源，其当然是不可或缺的，但这个文件格式是可以自定义的。从功能需求上讲，这个文件里至少要能包含msn列表，model number列表和image版本列表，为了更灵活的适应各种使用场景，这些列表可以部分甚至全部为空（甚至可以自定义其它列表），当文件中不存在这个列表时，意味着不对该项元素进行限定。

对限定功能需求抽象如下：
1. 从设备或镜像中获取客户数据
2. 从db中读取允许/不允许的客户数据列表，并与获取的客户数据进行匹配（目前是直接比较）,依据当前列表是黑名单还是白名单确定是否允许更新。
3. 不同类型的列表需要全部满足，方可更新。不过并不要求所有类型列表全部存在（事实上，该文件格式是可扩展的，并不规定实际的列表类型），所以是可以只限定部分条件的。

经过抽象后的模型，可以方便的添加其它的限制策略，比如依据日期，时间，PC序列号或激活码等等。

## 两种方案的对比
|   | 工作量 | 可维护性 | 扩展性 | 易用性 |
| ---- | ---- | ---- | ---- | ---- |
| 方案一 | 较小 | 较差 | 差 | 麻烦 |
| 方案二 | 较大 | 较强| 强 | 方便 |

经过比较，最终确定使用方案二来实现该功能。
需要说明的是方案二将会重新定义新的文件格式，与原有的msn的db文件格式是不同的，因而也不存在兼容旧的db文件格式之说。另外，新的db文件生成工具的用法也将会有所变化（后面会进行说明）。

# 架构设计
按照方案二的设计，整个功能的完整应用涉及多个工具的使用及部分代码的重构。现一一说明：
## 软件架构
### db文件格式
按照设计，db文件中需要支持存放多种列表类型，多个列表数据，且支持对列表数据进行加密存放。根据此需求，可以将db文件内容分三部分：
+ 文件头
文件头位于db文件开头，用于描述当前文件类型，标识db文件格式的版本，同时表明db文件中包含的列表数量及入口地址等，其定义如下：
``` c
#define NAME_LEN	16
struct security_header {
	char magic[4];		/* "UNI" */
	char project[NAME_LEN]; /* project name */
	short major_ver;        /* major vertion: 1 */
	short minor_ver;        /* minor vertion: 1 */
	size_t header_size;     /* sizeof this header */
	off_t section_entry;    /* entry of section headers */
	size_t sections;      /* how many sections in this file */
	size_t section_size;    /* section header size */
	unsigned short crc_ploynomial;     /* always to be 0xA001 */
	unsigned short content_crc;        /* crc16 code for the rest using crc_ploynomial above */
};
```
+ 列表头
列表头位于文件头之后（多个列表头连续存储），用于描述当前列表类型（列表名称来标识）,字段使用加密算法（算法名称来标识），字段数量以及字段大小等等。其定义如下：
```c
struct section_header {
	char name[NAME_LEN];
	char alg[NAME_LEN];
#define FLAG_DENY	0x0
#define FLAG_ALLOW	0x1
#define FLAG_RAW	(0x0 << 1)
#define FLAG_GLOB	(0x1 << 1)
#define FLGA_REGULAR (0x2 << 1)
	unsigned int flags;
	size_t entities;
	size_t entity_size;
	off_t entry;
};
```
其中flags用于标识当前名单是黑名单还是白名单，名单是raw数据（直接存储对应的名单数据）还是通配符模式数据，还是正则表达式模式等等[^2]。
[^2]: 目前只支持raw数据存储

+ 列表数据
列表数据位于文件最后，一般按对应列表头的顺序存放（但并不强制约定如此），列表数据的长度及列表之间的分界点由列表头来确定。

### 功能模组定义
根据设计，可以将功能为以下几个模块：
1. 抽象容器
抽象容器用于存放同类型的数据，将它们串在一起，并提供增删查改的功能，这里实际上就是链表。
2. 安全检查的核心模块
该模块是本功能的核心，它本身并不定义实际的安全检查策略，而是提供一个抽象框架，为实际的安全检查模块服务。主要有注册安全检查模块，注册加解密模块，执行抽象的安全检查过程以及db文件的生成与解析的功能。
3. 加解密模块
主要提供加解密算法，用于db文件中列表数据的加解密，通过向核心模块注册来实现其功能。
4. msn检查模块
提供MSN序列号的定义，获取办法，及匹配规则，向核心模块注册完成其功能。
5. model number检查模块
提供model number的定义，获取办法，及匹配规则，向核心模块注册完成其功能。
6. image版本检查模块
提供image版本信息的定义，获取办法，及匹配规则，向核心模块注册完成其功能。
7. db文件格式生成与解析
该模块主要用于db生成工具和核心模块的解析过程中。

其结构图如下：
``` mermaid
graph BT
container[抽象容器]--提供数据存储及管理服务-->core[核心模块]
encrypt[加解密模块]--注册模块-->core
msn[msn检查模块]--注册模块-->core
model[model number检查模块]--注册模块-->core
image[镜像检查模块]--注册模块-->core
core--提供安全检查服务-->更新工具
core--提供接口-->db生成工具
```

### 镜像更新流程
```  flow
st=>start: 开始
click=>operation: 用户点击更新按钮
read_db=>condition: 读取并解析db文件ok?
error=>end: 报错并退出更新
read_data=>condition: 读取当前用户数据成功?
check_data=>condition: 检查当前数据是否满足更新条件?
next_data=>operation: 进行下一项用户数据的检查
data_check_complete=>condition: 所有数据检查完毕
uncompress_image=>operation: 解压镜像
update=>operation: 更新
update_success=>condition: 更新成功？
complete=>end: 更新完成

st->click->read_db
read_db(no)->error
read_db(yes)->read_data(yes)->check_data
check_data(no)->error
check_data(yes)->data_check_complete
data_check_complete(yes)->update->complete
data_check_complete(no)->next_data(top)->read_data

```
其中，对镜像版本的验证，需要从镜像文件中获取镜像版本信息，这就需要将加密的镜像进行解密操作后进行，为防止不必要的风险，也从效率上考虑，应该将镜像版本的验证放在最后进行（实现的办法初步定为定义一个对各数据列表的排序函数，在写入或解析db文件时进行排序处理）。

## 镜像加密工具的使用
与之前MSN限定工具保持一致。

## db生成工具的使用
### 工具名称
由于文件格式重新定义了，db生成工具也与原来有所区别，名称定为“UniDBGenerator.exe"。

### 工具特性
+ 支持自定义数据列表类型
db文件格式支持存储多种数据类型列表，目前是MSN，model number和image版本三种，在验证核心模块中使用的是类型名来区分的，其分别为`msn`、`model_number`、`image_version`。但db生成工具本身是不限定数据类型及其名称的，所支持的列表类型是由各限定模块自己定义的，因而需要使用者在加入对应列表数据时指定各列表名称（也就是说这里不会对列表名称进行检查，由使用者根据实际情况指定）。

+ 支持命令行直接指定数据或通过特定格式文件指定
通过文件指定列表时，文件格式要求记录应该是定长的字符串，且一行一个记录。

+ 同时支持黑名单和白名单
在黑名单列表中，意味着不允许更新；在白名单列表中，意味着允许更新。如果同时存在白名单和黑名单，则需要同时满足两个条件才允许更新（即在白名单中，不在黑名单中）[^白名单说明]。
[^白名单说明]: 这里的白名单的要求较为严格，若设置了白名单，且白名单为空，则所有的设备都无法更新。如果只是想对部分名单拉黑，请只设置黑名单，不要设置白名单。

+ 支持生成空的db文件
db文件中没有任何有效数据，但是满足db文件格式（只有一个文件头），主要用于更新工具需要这个db文件而又不想添加任何限定条件的情况下使用。

### 参数说明
+ -h, --help
该参数用于显示简短的使用说明
+ --project proj
该参数用于指名db文件中的项目名称
+ --empty
该参数用于明确指定生成空的db文件
+ --allow
位于该选项后的列表，被认为是白名单，该选项为默认选项。
+ --deny
位于该选项后的列表，被认为是黑名单。
+ --xxx-list list_file
用于指定列表文件为list_file，文件格式为一行一个数据记录；其中xxx是列表名，当前可用列表名为`msn`、`model_number`、`image_version`三种[^3]。
[^3]: 生成工具可以任意指定列表名来生成db文件，但更新工具目前只能识别上面的三种列表，如果db中含有当前工具不能识别的列表类型，则认为有限定条件无法判断是否满足，从而报错退出。
+ --xxx data
用于在命令行上直接指定列表数据，xxx是列表名，指示列表类型。
+ --image image_file
用于通过指定镜像文件，从镜像文件中获取版本信息。[^4]
[^4]: 这里的镜像是没有加密过的镜像。
+ -o filename
用于指定输出的db文件名
+ 示例：
```sh
UniDBGenerator.exe --project g4_bba --allow --msn-list msn.txt --image G4_BBA_SS_v02p48.bin --deny --model_number-list model_number.txt -o UniAuth.db
```
上述命令用于生成g4项目的db文件，其中包含msn和image版本的白名单和model_number的黑名单。

注： 更新工具要求的db文件名变更为*UniAuth.db*

## 更新工具的打包
除了db生成工具名称变更以外，其它与之前MSN限定工具保持一致

# 接口描述
## 提供给扩展限定功能模块开发者的接口
### 核心数据结构
```c
struct list_head {
	struct list_head *next, *prev;
};

struct section_desc {
	const char *name;
	struct list_head list;
	size_t entity_size;
	struct section_ops *ops;
	struct record_ops *rops;
	void *data;
};

struct encrypt_algorithm_desc {
	const char *name;
	const size_t key_len;
	const unsigned char *key;
	int (*encode)(void *key, size_t key_len, const unsigned char *in, size_t len, unsigned char *out);
	int (*decode)(void *key, size_t key_len, const unsigned char *in, size_t len, unsigned char *out);
};
```
### 常用API接口描述

```c
int register_section(struct section_desc *desc);
int register_encrypt_algorithm(struct encrypt_algorithm_desc *desc);
void unregister_section(struct section_desc *desc);
void unregister_encrypt_algorithm(struct encrypt_algorithm_desc *desc);
```
其中`register_sectin`用于注册限定模块，该模块通过一个结构体`struct section_desc`（静态分配）来描述。
`register_encrypt_algorithm`用于注册加解密模块，通过一个结构体`struct encrypt_algorithm_desc`（静态分配）来描述。
其它两个接口用于注销对应的模块（这两个接口用不上，可能不会实现）。

### 示例代码
TODO

## 提供给上层应用的接口
### 核心数据结构
```c
struct security_desc {
	struct list_head section_desc_head;
	struct list_head encrypt_desc_head;
};

struct security_info {
	struct list_head section_head;
	struct security_desc *se_desc;
};
```
### 常用API接口描述
```c
int security_init(void);
int security_db_read(struct security_info *se_info, const char *filename);
int security_db_write(struct security_info *se_info, const char *filename);
int security_db_add_section(struct security_info *se_info, struct section *section);
int security_db_add_record(struct section *section, struct record *record);
```
`security_init`用于初始化安全检查模块，该函数会调用各子模块的初始化函数，完成子模块的注册；从而，后续就可以使用相应的接口了。
`security_db_read`用于将UniAuth.db读入内存，信息放入`struct security_info`中，供程序使用。
`security_db_write`用于将内存中的db数据写入文件，
`security_db_add_section`用于在现有的限定列表中添加新的列表。
`security_db_add_record`用于在现在有的限定列表中添加新的数据记录。

### 示例代码
TODO

## 事件上报
### 事件上报列表
| Event Code | MACRO              | Description | 备注 |
| :----      | :----              | :---- | |
| 0x3001     | EV_START           | 初始状态，程序启动尚未点击upgrade按钮| |
| 0x3002     | EV_UPGRADE         | 用户点击upgrade按钮 | |
| 0x3003     | EV_NETWORK_ERROR   | 网络错误事件 | |
| 0x3004     | EV_VERIFY_MSN      | 验证MSN的事件，有附加信息msn序号 | 已弃用 |
| 0x3005     | EV_MSN_DB_LOST     | 检测到msn.db文件打开失败后抛出的事件| 已弃用 |
| 0x3006     | EV_MSN_NOT_MATCH   | 对比设备与msn.db中的MSN序号，不匹配后抛出| 已弃用 |
| 0x3007     | EV_MSN_MATCH       | 对比设备与msn.db中的MSN序号，匹配后抛出| 已弃用 |
| 0x3008     | EV_INIT_SUCCESS    | 初始化（烧录前的准备）成功后抛出 | |
| 0x3009     | EV_INIT_ERROR      | 初始化（烧录前的准备）失败后抛出| |
| 0x300a     | EV_BURNPARTITION   | 强制烧录分区前抛出 | |
| 0x300b     | EV_BURNIMAGE       | 更新系统前抛出 | |
| 0x300c     | EV_UPGRADE_FAILED  | 更新userdata分区失败后抛出(后续在点击对话框后强制烧录) | |
| 0x300d     | EV_ERROR           | 烧录或更新出错| |
| 0x300e     | EV_COMPLETE        | 烧录或更新（成功）完成| |
| 0x300f     | EV_LOW_POWER       | 电池电量过低 | |
| 0x3010     | EV_USERDATA_ERROR_SKIP   | 在userdata更新失败时，用户选择了不进行强制更新 | |
| 0x3011     | EV_USERDATA_ERROR_MANDATORY | 在userdata更新失败时，用户选择了强制更新 | |
| 0x4001 | EV_AUTH_DB_LOST | db文件缺失时抛出，要求名为*UniAuth.db* | 新增 |
| 0x4002 | EV_AUTH_DB_ERROR | db文件解析出错时抛出（通常是文件被篡改） | 新增 |
| 0x4003 | EV_VERIFY_DATA | 验证数据的事件，有附加信息验证的类型（列表类型名）和当前的数据/序列号 | 新增 |
| 0x4004 | EV_DATA_NOT_MATCH | 数据验证失败的事件，有附加验证类型和当前的数据 | 新增 |
| 0x4005 | EV_AUTH_PASS | 所有验证通过的事件 | 新增 |

# 测试用例
## 测试路径