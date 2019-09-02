# 获取数据库连接对象java.sql.Connection,AOP切入DataSource的getConnection方法
```java
@Aspect
@Component
public class DataSourceAspect {

    private Logger logger = LoggerFactory.getLogger(DataSourceAspect.class);

    @Autowired
    private ILCNConnection lcnConnection;


    @Around("execution(* javax.sql.DataSource.getConnection(..))")
    public Connection around(ProceedingJoinPoint point)throws Throwable{

        logger.debug("getConnection-start---->");

        Connection connection = lcnConnection.getConnection(point);
        logger.debug("connection-->"+connection);

        logger.debug("getConnection-end---->");

        return connection;
    }
}

```
# 具体获取Connection实现委派到ILCNConnection,ILCNConnection实现类LCNTransactionDataSource
```java

/**
 * 关系型数据库代理连接池对象
 * create by lorne on 2017/7/29
 */

@Component
public class LCNTransactionDataSource extends AbstractResourceProxy<Connection,LCNDBConnection> implements ILCNConnection {


    private org.slf4j.Logger logger = LoggerFactory.getLogger(LCNTransactionDataSource.class);



    @Override
    protected Connection createLcnConnection(Connection connection, TxTransactionLocal txTransactionLocal) {
        nowCount++;
        if(txTransactionLocal.isHasStart()){
            LCNStartConnection lcnStartConnection = new LCNStartConnection(connection,subNowCount);
            logger.debug("get new start connection - > "+txTransactionLocal.getGroupId());
            pools.put(txTransactionLocal.getGroupId(), lcnStartConnection);
            txTransactionLocal.setHasConnection(true);
            return lcnStartConnection;
        }else {
            LCNDBConnection lcn = new LCNDBConnection(connection, dataSourceService, subNowCount);
            logger.debug("get new connection ->" + txTransactionLocal.getGroupId());
            pools.put(txTransactionLocal.getGroupId(), lcn);
            txTransactionLocal.setHasConnection(true);
            return lcn;
        }
    }


    @Override
    protected void initDbType() {
        TxTransactionLocal txTransactionLocal = TxTransactionLocal.current();
        if(txTransactionLocal!=null) {
            //设置db类型
            txTransactionLocal.setType("datasource");
        }
        TxCompensateLocal txCompensateLocal = TxCompensateLocal.current();
        if(txCompensateLocal!=null){
            //设置db类型
            txCompensateLocal.setType("datasource");
        }
    }

    @Override
    public Connection getConnection(ProceedingJoinPoint point) throws Throwable {
        //说明有db操作.
        hasTransaction = true;

        initDbType();

        Connection connection = (Connection)loadConnection();
        if(connection==null) {
            //委派到父类初始化
            connection = initLCNConnection((Connection) point.proceed());
            if(connection==null){
                throw new SQLException("connection was overload");
            }
            return connection;
        }else {
            return connection;
        }
    }
}
// 父类
   protected C initLCNConnection(C connection) {
        logger.debug("initLCNConnection");
        C lcnConnection = connection;
        TxTransactionLocal txTransactionLocal = TxTransactionLocal.current();

        if (txTransactionLocal != null&&!txTransactionLocal.isHasConnection()) {

            logger.debug("lcn datasource transaction control ");

            //补偿的情况的
//            if (TxCompensateLocal.current() != null) {
//                logger.info("rollback transaction ");
//                return getCompensateConnection(connection,TxCompensateLocal.current());
//            }

            if(StringUtils.isNotEmpty(txTransactionLocal.getGroupId())){

                logger.debug("lcn transaction ");
                return createConnection(txTransactionLocal, connection);
            }
        }
        logger.debug("load default connection !");
        return lcnConnection;
    }

    private C createConnection(TxTransactionLocal txTransactionLocal, C connection){
        if (nowCount == maxCount) {
            for (int i = 0; i < maxWaitTime; i++) {
                for(int j=0;j<100;j++){
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    if (nowCount < maxCount) {
                        return createLcnConnection(connection, txTransactionLocal);
                    }
                }
            }
        } else if (nowCount < maxCount) {
            return createLcnConnection(connection, txTransactionLocal);
        } else {
            logger.info("connection was overload");
            return null;
        }
        return connection;
    }

    @Override
    protected Connection createLcnConnection(Connection connection, TxTransactionLocal txTransactionLocal) {
        nowCount++;
        if(txTransactionLocal.isHasStart()){
            LCNStartConnection lcnStartConnection = new LCNStartConnection(connection,subNowCount);
            logger.debug("get new start connection - > "+txTransactionLocal.getGroupId());
            pools.put(txTransactionLocal.getGroupId(), lcnStartConnection);
            txTransactionLocal.setHasConnection(true);
            return lcnStartConnection;
        }else {
            LCNDBConnection lcn = new LCNDBConnection(connection, dataSourceService, subNowCount);
            logger.debug("get new connection ->" + txTransactionLocal.getGroupId());
            pools.put(txTransactionLocal.getGroupId(), lcn);
            txTransactionLocal.setHasConnection(true);
            return lcn;
        }
    }

     public LCNDBConnection(Connection connection, DataSourceService dataSourceService, ICallClose<ILCNResource> runnable) {
            logger.debug("init lcn connection ! ");
            this.connection = connection;
            this.runnable = runnable;
            this.dataSourceService = dataSourceService;
            TxTransactionLocal transactionLocal = TxTransactionLocal.current();
            groupId = transactionLocal.getGroupId();
            maxOutTime = transactionLocal.getMaxTimeOut();
    
            // 创建Task
            TaskGroup taskGroup = TaskGroupManager.getInstance().createTask(transactionLocal.getKid(),transactionLocal.getType());
            waitTask = taskGroup.getCurrent();
        }

```

# Task源码
```java

  protected Task() {
        lock = new ReentrantLock();
        condition = lock.newCondition();
    }
```

# Connection提交事务
```java
    public void transaction() throws SQLException {
        if (waitTask == null) {
            rollbackConnection();
            System.out.println(" waitTask is null");
            return;
        }


        //start 结束就是全部事务的结束表示,考虑start挂掉的情况
        Timer timer = new Timer();
        System.out.println(" maxOutTime : "+maxOutTime);
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("auto execute ,groupId:" + getGroupId());
                dataSourceService.schedule(getGroupId(), waitTask);
            }
        }, maxOutTime);

        System.out.println("transaction is wait for TxManager notify, groupId : " + getGroupId());
        // 挂起当前线程，等待业务处理结果以及补偿信息来判断当前事务提交或者回滚
        waitTask.awaitTask();

        timer.cancel();

        int rs = waitTask.getState();


        try {
            if (rs == 1) {
                connection.commit();
            } else {
                rollbackConnection();
            }

            System.out.println("lcn transaction over, res -> groupId:"+getGroupId()+" and  state is "+(rs==1?"commit":"rollback"));

        }catch (SQLException e){
            System.out.println("lcn transaction over,but connection is closed, res -> groupId:"+getGroupId());

            waitTask.setState(TaskState.connectionError.getCode());
        }


        waitTask.remove();

    }
```
# TxStartTransactionServerImpl 源码
```java
@Service(value = "txStartTransactionServer")
public class TxStartTransactionServerImpl implements TransactionServer {


    private Logger logger = LoggerFactory.getLogger(TxStartTransactionServerImpl.class);


    @Autowired
    protected MQTxManagerService txManagerService;


    @Override
    public Object execute(ProceedingJoinPoint point,final TxTransactionInfo info) throws Throwable {
        //分布式事务开始执行

        logger.debug("--->begin start transaction");

        final long start = System.currentTimeMillis();

        int state = 0;

        final String groupId = TxCompensateLocal.current()==null?KidUtils.generateShortUuid():TxCompensateLocal.current().getGroupId();

        //创建事务组
        txManagerService.createTransactionGroup(groupId);


        TxTransactionLocal txTransactionLocal = new TxTransactionLocal();
        txTransactionLocal.setGroupId(groupId);
        txTransactionLocal.setHasStart(true);
        txTransactionLocal.setMaxTimeOut(Constants.txServer.getCompensateMaxWaitTime());
        TxTransactionLocal.setCurrent(txTransactionLocal);


        try {
            Object obj = point.proceed();
            state = 1;
            return obj;
        } catch (Throwable e) {
            state = rollbackException(info,e);
            throw e;
        } finally {

            final String type = txTransactionLocal.getType();

            int rs = txManagerService.closeTransactionGroup(groupId, state);

            int lastState = rs==-1?0:state;

            int executeConnectionError = 0;

            //控制本地事务的数据提交
            final TxTask waitTask = TaskGroupManager.getInstance().getTask(groupId, type);
            // 获取Task对象，Task对象持有Lock以及Lock的Condition,Condition控制事务线程挂起和继续
            if(waitTask!=null){
                // 业务方法执行完毕，设置执行结果状态
                waitTask.setState(lastState);
                // 通知事务线程，使其继续执行
                waitTask.signalTask();

                while (!waitTask.isRemove()){
                    try {
                        Thread.sleep(1);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }

                if(waitTask.getState()== TaskState.connectionError.getCode()){
                    //本地执行失败.
                    executeConnectionError = 1;

                    lastState = 0;
                }
            }

            final TxCompensateLocal compensateLocal =  TxCompensateLocal.current();

            if (compensateLocal == null) {
                long end = System.currentTimeMillis();
                long time = end - start;
                if ((executeConnectionError == 1&&rs == 1)||(lastState == 1 && rs == 0)) {
                    //记录补偿日志
                    txManagerService.sendCompensateMsg(groupId, time, info,executeConnectionError);
                }
            }else{
                if(rs==1){
                    lastState = 1;
                }else{
                    lastState = 0;
                }
            }

            TxTransactionLocal.setCurrent(null);
            logger.debug("<---end start transaction");
            logger.debug("start transaction over, res -> groupId:" + groupId + ", now state:" + (lastState == 1 ? "commit" : "rollback"));

        }
    }


    private int  rollbackException(TxTransactionInfo info,Throwable throwable){

        //spring 事务机制默认回滚异常.
        if(RuntimeException.class.isAssignableFrom(throwable.getClass())){
            return 0;
        }

        if(Error.class.isAssignableFrom(throwable.getClass())){
            return 0;
        }

        //回滚异常检测.
        for(Class<? extends Throwable> rollbackFor:info.getTransaction().rollbackFor()){

            //存在关系
            if(rollbackFor.isAssignableFrom(throwable.getClass())){
                return 0;
            }

        }

        //不回滚异常检测.
        for(Class<? extends Throwable> rollbackFor:info.getTransaction().noRollbackFor()){

            //存在关系
            if(rollbackFor.isAssignableFrom(throwable.getClass())){
                return 1;
            }
        }
        return 1;
    }
}

```