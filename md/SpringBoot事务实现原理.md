# SpringBoot事务实现原理

## 基于Spring AOP实现 [AOP解析](https://github.com/lucky-xin/Learning/blob/gh-pages/md/Spring%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E4%B9%8BAOP.md)

## 基于AOP实现，把有Transactional注解的方法封装成Advisor，提交事务之前先保存提交事务之前状态（TransactionStatus），然后提交事务。如果事务执行失败，则回滚到保存状态点TransactionStatus。成功就不回滚。具体实现见源码分析。

## 1.通过注解`@EnableTransactionManagement`打开事务配置，通过ProxyTransactionManagementConfiguration注册AdvisorBeanFactoryTransactionAttributeSourceAdvisor
```java

```
