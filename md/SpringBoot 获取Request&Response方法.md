1.方法参数直接注入
```java
    @RequestMapping(value = "/{orderId}/attachment/{attachmentId}", method = RequestMethod.GET)
    public void getAttachment(HttpServletRequest request, HttpServletResponse response, 
                              @PathVariable Long orderId, @PathVariable Long attachmentId) throws Exception {
        orderInfoService.getAttachment(response, orderId, attachmentId);
    }
```
2.ServletRequestAttributes获取
```java
@GetMapping(value = "")
public String center() {
    ServletRequestAttributes servletRequestAttributes = (ServletRequestAttributes)RequestContextHolder.getRequestAttributes();
    HttpServletRequest request = servletRequestAttributes.getRequest();
    HttpServletResponse response = servletRequestAttributes.getResponse();
    //...
}
```
3.直接注入
```java
@Autowired
private HttpServletRequest request;

@Autowired
private HttpServletResponse response;
```

# 直接注入实现原理,实际注入的对象如图所示[注入的对象](https://github.com/lucky-xin/Learning/blob/gh-pages/image/elasticsearch-search-pas.png)




