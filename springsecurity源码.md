FilterComparator  指定过滤器顺序

在WebSecurity的build方法中调用HttpSecurity的build来将追加到HttpSecurity中filter构建成SecurityFilterChain，再把所有的SecurityFilterChain追加到FilterChainProxy中，最后通过DelegatingFilterProxy注册到ServletContext中的



HttpSecurity#performBuild

```java
protected DefaultSecurityFilterChain performBuild() {
   filters.sort(comparator);
   return new DefaultSecurityFilterChain(requestMatcher, filters);
}
```



