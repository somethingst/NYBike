# 遗留问题解决

1. 饼图数据刷新操作，会导致数据一直叠加
   - 问题原因：在updateData方法中，通过xData和yData生成pieData的操作中，没有清空之前pieData中的数据，而是使用pieData.push()方法向之前的数组中继续添加数据，导致用于展示的数组中包含多组记录。
   - 解决方案：在循环向pieData中添加数据之前，使用pieData=[]的方式，将pieData变成一个新的空数组。
2. 存在大量重复代码，进行封装
   - 新建为pannel2.html，方法都封装到pannel2.js里面。
   - 将站点状态数据获取封装成initPart1函数
   - 将yData和pieData的获取封装成函数，并且我们发现yData和pieData不需要定义为全局的变量，只需要作为函数的返回值分别赋给局部变量即可。至此数据初始化完，要对左右表初始化。
   - 定义initLeft和initRight函数，作为对表的初始化的函数。对表的初始化需要参数，分别传参即可。
   - 定义updateData函数，还是要获取最新的状态数据，赋给局部变量。
   - 左右表数据刷新函数，只对series里的数据进行更新。

#  历史数据对比

1. 问题：为什么要分析数据？ 用于支持商业决策，理性的决策需要数据的支持。
   - 纽约市在其官网上共享了共享单车的历史数据：`https://www.citibikenyc.com/system-data`->Citi Bike Trip Histories。
   - 该部分数据的下载地址为`https://s3.amazonaws.com/tripdata/index.html`。
   - 历史数据以每一次骑行作为一条记录，记录中包含如下字段：
     - Trip Duration (seconds)：骑行时长，以秒为单位
     - Start Time and Date：骑行开始的日期和时间
     - Stop Time and Date：骑行结束的日期和时间
     - Start Station Name：骑行开始的站点名称
     - End Station Name：骑行结束的站点名称
     - Station ID：站点的id，实际上是2个字段，分别是开始站点id和结束站点id
     - Station Lat/Long：开始站点和结束站点的经纬度信息
     - Bike ID：车辆id
     - User Type (Customer = 24-hour pass or 3-day pass user; Subscriber = Annual Member)：用户类型(Cutomer代表临时用户，24小时通用或3天通用，Subscriber代表订阅用户，年卡会员)
     - Gender (Zero=unknown; 1=male; 2=female)：用户性别（0-未知；1-男性；2-女性）
     - Year of Birth：用户出生的年份
2. 这么多的数据(上百万条)，用什么平台去分析较为便捷呢？ -> 关系型数据库
3. 基于这些数据，可以分析哪些内容？
   - 计算每一天的总骑行量（数据按日期分组，统计每组的总数据条数）
   - 计算每一天不同性别的用户的骑行数量（数据按日期分组，再按性别分组）
   - 研究天气对骑行数量的影响（将天气数据和骑行次数在一起显示）
   - 计算不同年龄段的骑行总数
   - （起始 、终止）站点热门排行
   - 骑行时间的众数
   - 不同骑行时间的占比
   - 骑行次数 、时长较多的自行车ID
   - 骑行的路程（缺少实际路线，预估）
   - 研究用车高峰和低谷的时间段
   - 热门的骑行线路和拥挤路段
   - 长时间没人用的车和站点
   - 会员和非会员的比例
   - 会员和非会员的用车高峰，骑行时长对比
   - 会员和非会员在热门站点上的区别
   - 不同时间段的热门站点变化

# 2019年4月每一天的总骑行量可视化

1. 需求分析：以图表的形式，显示2019年4月每一天的总骑行数量。

2. 思路分析：
   
   - ![](https://img.99couple.top/20200602152659.png)
   
3. 建库建表，保存原始数据
   - 在nybike项目中，在`src/main/java`下创建`cn.tedu.nybike.util`包，并在该包下创建`nybike.sql`。该文件仅用于保存这个项目涉及到的所有的SQL语句。
   - 该文件仅用于保存项目会所涉及到的SQL语句
   
4. 将表导入数据库中

   - ```mysql
     -- 将csv数据导入数据库表
     load data infile 'D:/201904-citibike-tripdata.csv' -- 文件的绝对路径
     into table tb_trip_1904 -- 表名
     fields terminated by ',' -- 原始文件中的分隔符
     optionally enclosed by '"' -- 去掉原始数据前后的双引号
     ignore 1 lines; -- 导入时忽略第一行-表头行
     ```

# 基础知识：

1. 关系型数据库：
   - 近代数据库分为三种：层次性数据库、网格型数据库、关系型数据库
   - 关系型数据库理念诞生于1970年，由IBM的研究员研发。
   - 1979年，Oracle公司基于关系型数据库模型，开发了商用的关系型数据库模型。
   - 目前，Oracle数据库是全球最流行的商用关系型数据库软件。
   - 瑞典的公司推出了免费的、轻量级的关系型数据库MySQL，得到了行业中的广泛认可。
   - Mariadb：MySQL的社区版，全面支持MySQL，同时永久免费
   - ![](https://img.99couple.top/20200602143456.png)
2. MySQL中的数据类型
   - 问题：为什么要有不同的数据类型？方便进行不同类型的数据的计算，提升效率。
   - 问题：同样是整数，为什么要分byte、short、int、long？避免空间浪费，节省资源。
   - 在MySQL中对应类型为tinyint、smallint、int、bigint。
     - byte：1字节，8位（bit），0000 0000 任意位可以为0或1，可能的组合为2^8，256个数字，用于保存数字，范围 -128 ~ 127
     - short：2字节，16位（bit），2^16,65536个数字，范围 -32768 ~ 32767
     - int：4字节，32位（bit），2^32，范围 -2^31 ~ 2^31-1
     - long：8字节，64位（bit），2^64，范围 -2^63 ~ 2^63-1
   - 具体的数据类型：
     - 数值型：
       - 整型：整数
         - tinyint
         - smallint
         - int
         - bigint
       - 浮点型：小数
         - float 4字节 单精度
         - double 8字节 双精度
         - decimal 8字节  没有精度损失，金额类型必须用此类型
     - 字符型：
       - char 不可变长度字符，声明数据字段为char时必须提供一个char的长度`xxx cahr(20)`，无论数据多长，那么该字段在硬盘上会实际占用20字符的空间。
       - varchar 可变长度字符 ，声明数据字段为varchar时，也要提供长度`xxx varchar(20)`，那么在硬盘上，该字段实践占用的空间由存入该字段的值得长度来决定，但是不能超过给定的数字。
       - 但不能用varchar去取代char，原因是二者的执行机制不同。char是一次性开辟固定长度的空间，然后在需要保存数据时，直接无脑放进去。快！而varchar时先判断数据实际长度，再用该长度去开辟空间，多了一步判断。慢！
     - 大文本-Text（MySQL方言，自己这么叫，其他数据库叫CLOB，Character Large Object）字符大对象
     - 二进制数据-BLOB（Binary Large Object）
     - 日期：
       - date 日期 4个字节 仅能保存年月日信息
       - time 时间 4个自己 仅能保存时分秒
       - datetime 日期时间 8个字节 日期和时间
       - timestamp 时间戳类型 8个字节 保存时间戳数据