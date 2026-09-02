 <b>bootdao是基于spring-boot的持久层封装</b><br>
 source：<a href='https://github.com/wjmx1214/bootdao'>https://github.com/wjmx1214/bootdao</a>
 
<pre>

 简介：
  1.函数式封装，适合轻量业务以常见泛型函数方式访问数据层
  2.可与其他持久层框架并存，无依赖式兼容JPA或mybatis-plus的实体注解，也可使用EntityTable注解，仅依赖spring-boot基础包
  3.支持扩展更多函数，支持entity、dto、vo无感知无差别调用(配置好映射路径即可)
  4.支持静默多数据源，若yml或xml按特定名称配置好多数据源后，无需其他配置即可使用多数据源
  5.支持注解式多条件动态查询，单表分页查询全量字段可省略SQL，参考：com.boot.dao.api.Search
  6.支持SQL语句静态常量化：bootdao 启动时扫描 classpath 下 .sql，通过 com.boot.dao.api.SQL 运行时获取；可选 SQL.createSqlClass() 生成 XxxSQL 常量类
  7.目前尚未经过大规模性能和稳定性测试，暂不支持缓存

 使用：
  1.可直接在服务层注入IBaseDAO进行泛型函数式调用, 无需定义任何业务DAO, 注入方式请查看IBaseDAO接口说明
  2.可同时兼并使用mybatis、jpa或JdbcTemplate等框架

 作者：wang.jia.le	2020-12-01	若发现BUG或疑惑请至信	wjmx1214@sina.com

</pre>

 pom：
 
	 <dependency>
        <groupId>com.bootdao</groupId>
        <artifactId>bootdao-spring-boot-starter</artifactId>
        <version>1.2.4</version>
	 </dependency>

 yml配置(选配)：
 
	#关系型数据库持久层函数式封装;
	#注入方式请查看IBaseDAO接口说明;
	#一个业务中只有一个数据源调用时使用: @Transactional(value = "transactionManager或事务名称1", rollbackFor=Exception.class)
	#一个业务中包含多个数据源调用时使用: @TransactionalMore(value = {"transactionManager或事务名称1", "事务名称2"})，可省略rollbackFor，已默认捕获Throwable异常
	#注意: @TransactionalMore 不是 XA 分布式事务，仅依次开启多个本地事务并统一提交/回滚；若前一数据源已 commit 而后一数据源失败，可能出现跨库数据不一致，强一致场景请用 Seata 等方案（详见 TransactionalMore 注解说明）

	bootdao: #关系型数据库持久层函数式封装, 多数据源配置以及更多详细说明请参考IBaseReadme.class
	    #entity-paths: com.xxx.xxx.entity #实体类包路径, 用于entity、dto、vo无差别调用(可指定多个包路径用逗号分隔; 也可不配置, 由@EntityPath注解到Dto上)
	    #auto-createtime: true #当有创建时间字段时, 是否自动生成值(默认false)(根据名称createTime或createDate推理)(mysql5.x无法同时创建时间和更新时间自动配置, mysql8.x无问题)
	    #show-sql: true #是否显示SQL语句(默认=false)
	    #show-param: true #是否显示SQL参数(默认=false)
	    #show-source: true #是否显示数据源相关信息, 主要用于调试(默认=false)
	    #save-batch-size: 50000 #批量执行SQL单次最多提交数量(默认10000行)
	    #different-names: Dto, Vo #实体类与DTO或VO类名不相同的部分, 用于entity、dto、vo无差别调用, 可直接将其作为参数类型(可指定多个名称, 默认Dto,Vo)
	    #snowflake-id-worker: 1, 1 #基于雪花算法的ID生成器, 工作ID (0~31) / 数据中心ID (0~31) (目前自动生成情况下, 仅用于clickhouse库表主键)(默认1, 1); 多实例部署时必须为各实例配置不同的 workerId/datacenterId，否则可能主键冲突
	    #rethrow-query-exception: false #查询异常是否包装为BootDaoQueryException并抛出(默认true; false时兼容旧版：仅记录日志并返回null或空集)


 若yml未配置或类名无法对应，但需要entity、dto、vo无差别调用时，可在Dto类中通过@EntityPath注解配置
 
	@EntityPath("com.xxx.xxx.entity.Student")
	public class StuDto {
		private Long stuId;
		//...
	}

 IBaseDAO使用示例：
```

	@Service
	@Transactional(rollbackFor=Exception.class)
	public class StuService implements IStuService{
	   @Autowired
	   private IBaseDAO baseDAO;
	   
	   @Autowired
	   @Qualifier("mysql") //mysql数据源
	   private IBaseDAO mysql;
	   
	   @Override
	   public void list() throws Exception{
			String sql = "SELECT * FROM stu WHERE age > ?";
			List<StuDto> list = baseDAO.getEntitys(sql, StuDto.class, 15);
			for(StuDto stu : list) {
	            System.out.println(stu);
			}
	   }
	}

 多条件动态查询示例：

	public class StuSearch extends PageSearch{ //BaseSearch
		private Long id;
		@Search(column="stu_name", type=SearchType.like_right)
		private String name;
		@Search(column="stu_name", type=SearchType.like_all, tableAs="s", whereKey="search2")
		private String name2;
	}

 调用示例：

	public Page<StuDto> pageStu(StuSearch search){
		search.SQL = "(select * from stu where 1=1 #{search或任意标识}) union (select * from stu s where s.on_class=1 #{search2})";
		return baseDAO.page(search, StuDto.class);
		or
		//search.appendWhere("select * from stu where 1=1 #{search}"); //单表分页查询全量字段可省略SQL
		return baseDAO.page(search, StuDto.class);
	}
	public List<StuDto> listStu(StuSearch search){
		search.appendWhere("select * from stu where 1=1 #{search}");
		return baseDAO.getEntitys(search.SQL, StuDto.class, search.params);
		or
		search.SQL = "select * from stu where 1=1 #{search}";
		return baseDAO.getEntitys(search.appendWhere(), StuDto.class, search.params);
	}


 SQL运行时示例：
 
	说明：
	 1.bootdao 启动时自动扫描 classpath 下 bootdao 格式 .sql；未扫描到任何合法 .sql 时静默跳过，视为未使用本功能
	 2.运行时通过 SQL.get("stu.sql.page") 获取 SQL 正文；key 格式为 文件名.sql.标签名，文件名须全局唯一（全项目仅允许一份 stu.sql）
	 3.stu.sql 放在 java 源码包目录并打入 classpath，推荐 src/main/java/com/foo/impl/sql/stu.sql
	 4.可选：调用 SQL.createSqlClass() 生成 StuSQL 常量类后，可改用 StuSQL.page

	<sql>
	<queryByName>select * from stu where name = ?</queryByName>
	<page>
	select * from stu where #{search}
	</page>
	</sql>

	@Service
	@Transactional(rollbackFor=Exception.class)
	public class StuService implements IStuService{
	   @Autowired
	   private IBaseDAO baseDAO;
	   
	   @Override
	   public List<StuDto> listByName(String name) throws Exception{
	      return baseDAO.getEntitys(SQL.get("stu.sql.queryByName"), StuDto.class, name);
	   }
	   
	   @Override
	   public Page<StuDto> pageStu(StuSearch search) throws Exception{
	      search.appendWhere(StuSQL.page);
	      return baseDAO.page(search, StuDto.class);
	   }
	}

 手动生成 XxxSQL 常量类（可选）：

	说明：扫描项目内全部 bootdao 格式 .sql，生成 target/generated-sources/annotations 下各包 XxxSQL.java。生成后可用 StuSQL.page 替代 SQL.get("stu.sql.page")。

	public class SqlBuild {
	    public static void main(String[] args) {
	        SQL.createSqlClass();
	    }
	}

 运行时加载说明：

	业务项目需将.sql文件复制到 target/classes或打入jar包，pom 需配置：

	<plugin>
	  <artifactId>maven-resources-plugin</artifactId>
	  <executions>
	    <execution>
	      <id>copy-main-sql-from-java-tree</id>
	      <phase>process-sources</phase>
	      <goals><goal>copy-resources</goal></goals>
	      <configuration>
	        <outputDirectory>${project.build.outputDirectory}</outputDirectory>
	        <resources>
	          <resource>
	            <directory>src/main/java</directory>
	            <includes><include>**/*.sql</include></includes>
	          </resource>
	        </resources>
	      </configuration>
	    </execution>
	  </executions>
	</plugin>
