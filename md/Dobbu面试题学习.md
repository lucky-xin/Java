```java
 @Reference(
            version = "1.0.0",
            sticky = true,
            interfaceClass = RoleService.class
    )
 sticky 配置，sticky 表示粘滞连接。所谓粘滞连接是指让服务消费者尽可能的
 调用同一个服务提供者，除非该提供者挂了再进行切换
```
# 源码实现如下org.apache.dubbo.rpc.cluster.support.AbstractClusterInvoker#select
```java
    protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation,
                                List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {

        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        String methodName = invocation == null ? StringUtils.EMPTY : invocation.getMethodName();

        boolean sticky = invokers.get(0).getUrl()
                .getMethodParameter(methodName, Constants.CLUSTER_STICKY_KEY, Constants.DEFAULT_CLUSTER_STICKY);

        //ignore overloaded method
        if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
            stickyInvoker = null;
        }
        //ignore concurrency problem
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
            if (availablecheck && stickyInvoker.isAvailable()) {
                return stickyInvoker;
            }
        }

        Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);

        if (sticky) {
            stickyInvoker = invoker;
        }
        return invoker;
    }
```