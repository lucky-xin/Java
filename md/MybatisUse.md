# MyBatis 源码学习

## MyBatis使用。MySqlSessionFactory 封装管理SqlSessionFactory。MySqlSessionFactory为单例

```java
public class MySqlSessionFactory {

    private static SqlSessionFactory sqlSessionFactory = null;

    private static Object lock = new Object();

    public static SqlSessionFactory getSqlSessionFactory() {
        if (sqlSessionFactory == null) {
            synchronized (lock) {
                if (sqlSessionFactory == null) {
                    init();
                }
            }
        }
        return sqlSessionFactory;
    }

    static void init() {
        // 1.获取配置文件
        String resource = "SqlMapConfig.xml";
        InputStream in;
        try {
            in = Resources.getResourceAsStream(resource);
            // 2.创建会话工厂
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(in);
        } catch (IOException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    public static SqlSession openSqlSession() {
        if (sqlSessionFactory == null) {
            init();
        }
        return sqlSessionFactory.openSession();
    }
}
```
## 配置文件SqlMapConfig.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

	<properties resource="jdbcconfig.properties"></properties>

    <settings>
        <setting name="lazyLoadingEnabled" value="true"/>
        <setting name="aggressiveLazyLoading" value="false"/>
    </settings>

    <!-- 定义别名 批量扫描-->
	<typeAliases>
		<!-- <typeAlias type="com.mybites.domain.User" alias="user" /> -->
		<package name="com.ssm.po" />
	</typeAliases>

	<environments default="development">
		<environment id="development">
			<!-- 使用jdbc事务管理 -->
			<transactionManager type="JDBC" />
			<!-- 数据库连接池 -->
			<dataSource type="POOLED">
				<property name="driver" value="${jdbcconfig.driverClassName}" />
				<property name="url" value="${jdbcconfig.url}" />
				<property name="username" value="${jdbcconfig.username}" />
				<property name="password" value="${jdbcconfig.password}" />
			</dataSource>
		</environment>
	</environments>
	<!-- 加载映射文件 -->
	<mappers>
		<!-- <mapper resource="mapper/UserMapper.xml" /> -->
		<!--<mapper resource="sqlmap/User.xml" />-->
		<!--<mapper class="com.mybatis.mapper.UserMapper" />-->
		<!--批量加载 -->
		<package name="com.mybatis.mapper" />
	</mappers>
</configuration>
```

## 用户Mapper定义 UserMapper
```java
public interface UserMapper {

	// 用户综合查询
	 List<UserCustom> findUserList(UserQueryVo queryVo) throws Exception;

	// 用户综合查询总数
	 int findUserCount(UserQueryVo queryVo) throws Exception;

	 User findUserByResultMap(int id) throws Exception;

	 User findUserById(int id) throws Exception;

	 List<User> findUserByName(String name) throws Exception;

	 void insertUser(User user) throws Exception;

	 void deleteUser(int id) throws Exception;

	 void updateUser(User user) throws Exception;

	List<User> findUserByHashMap(HashMap<String, Object> map)throws Exception;

}

public class UserQueryVo {

    private List<Integer> ids;

    private UserCustom userCustom;

    public UserCustom getUserCustom() {
        return userCustom;
    }

    public void setUserCustom(UserCustom userCustom) {
        this.userCustom = userCustom;
    }

    public List<Integer> getIds() {
        return ids;
    }

    public void setIds(List<Integer> ids) {
        this.ids = ids;
    }
}

public class UserCustom extends User {

    private static final long serialVersionUID = 1L;
    public UserCustom(){}

    public UserCustom(String email, String name, String sex, Date birth, String address, String password) {
        super(email, name, sex, birth, address, password);
    }
}

public class User implements Serializable{
    private static final long serialVersionUID = 1L;
    private int id;
    private String email;
    private String name;
    private String sex;
    private Date birth;
    private String address;
    private byte[] image;
    private String password;
    private List<Orders> ordersList;

    public List<Orders> getOrdersList() {
        return ordersList;
    }

    public void setOrdersList(List<Orders> ordersList) {
        this.ordersList = ordersList;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getSex() {
        return sex;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }

    public Date getBirth() {
        return birth;
    }

    public void setBirth(Date birth) {
        this.birth = birth;
    }

    public String getAddress() {
        return address;
    }

    public void setAddress(String address) {
        this.address = address;
    }

    public byte[] getImage() {
        return image;
    }

    public void setImage(byte[] image) {
        this.image = image;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public User()
    {
    }

    public User(String email, String name, String sex, Date birth,
                String address, String password) {
        super();
        this.email = email;
        this.name = name;
        this.sex = sex;
        this.birth = birth;
        this.address = address;
        this.password = password;
    }

    @Override
    public String toString() {
        return "User [id=" + id + ", name=" + name + ", password=" + password
                + ", email=" + email + ", sex=" + sex + ", birth=" + birth
                + ", address=" + address + "]";
    }


}

```

## 调用Mapper服务
```java
public class UserMapperTest {

    static SqlSession sqlSession = null;
    static UserMapper userMapper = null;

    static {
        sqlSession = MySqlSessionFactory.openSqlSession();
        userMapper = sqlSession.getMapper(UserMapper.class);
    }

    @Test
    public void test1(){
        String str="";
        String resource = str.getClass().getName().replace('.', '/') + ".java (best guess)";
        System.out.println(resource);
    }
    @Test
    public void testFindUserById() throws Exception {
//        userMapper = sqlSession.getMapper(UserMapper.class);
        System.out.println(userMapper);
        User user = userMapper.findUserById(42);
        //User [id=42, name=Ivan, password=123, email=1314@qq.com, sex=男, birth=Thu Jan 01 00:00:00 CST 1970, address=长沙]
        System.out.println(user);
    }
    
  }
  
```



