# MyBatis 源码学习

## 使用
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
