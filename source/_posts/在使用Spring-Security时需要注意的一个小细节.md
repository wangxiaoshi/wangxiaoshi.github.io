---
title: 在使用Spring Security时踩到的一个小坑
date: 2020-01-16 10:51:40
tags:
- springboot
- JWT Token
categories: 开发笔记
cover: https://raw.githubusercontent.com/wangxiaoshi/image-host-xiaoshi/master/blog_files/img/spring-boot-security.png
---
## JwtAuthenticationFilter
关于什么是JWT Token在这篇文章里就不介绍了，主要是想记一个之前困扰了团队很长时间的小bug。项目采用Springboot，同事参照[这篇文章](https://www.devglan.com/spring-security/spring-boot-jwt-auth)设置了以下几个类: 
- `JwtAuthenticationFilter.java` 
- `JwtTokenUtil.java`
- `WebSecurityConfig.java`
- `AuthenticationController.java`

并且在`WebSecurityConfig`中做了如下configure：
```
    @Override
	protected void configure(HttpSecurity http) throws Exception {
		  http
          .csrf().disable()
          .headers().disable()
          .cors().and()
          .authorizeRequests()
              .antMatchers(HttpMethod.POST, "/token/generate-token").permitAll()
              .antMatchers(HttpMethod.POST, "/users/signup").permitAll()
              .antMatchers(HttpMethod.GET, "/entries").hasAuthority("USER")
              .anyRequest().denyAll()
              .and()
          .sessionManagement()
              .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
              .and()
          .exceptionHandling()
          .authenticationEntryPoint(unauthorizedHandler);
		http.addFilterBefore(authenticationTokenFilterBean(), UsernamePasswordAuthenticationFilter.class);
	}
```

测试时发现，用户在登陆后正常获取到 token，但是调用接口 /entries 时本应验证已经赋予每个用户的权限`USER`，却总是返回 HTTP 401，也就意味着用户并没有这个权限。于是便开始了漫长的寻找，最开始我坚定的认为前段获取的 token 里肯定是带着我们需要的 Authority USER 的，一上午无果。于是我才想到有可能是这个 authority 根本就没有被赋予用户，回到 `JwtAuthenticationFilter` 里面仔细查看，终于找到了问题所在：

## UsernamePasswordAuthenticationToken 的构造器
原来，我们的 `UsernamePasswordAuthenticationToken` 有多种构造器。在上面的那篇文章中，是这样用的：
```
UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails, null, Arrays.asList(new SimpleGrantedAuthority("USER")));
```
可以看到这里有三个参数。查找[Spring的源码](https://docs.spring.io/spring-security/site/docs/4.2.13.RELEASE/apidocs/org/springframework/security/authentication/UsernamePasswordAuthenticationToken.html)得知，调用了第二种Constructor

> `UsernamePasswordAuthenticationToken(Object principal, Object credentials)`<br> This constructor can be safely used by any code that wishes to create a UsernamePasswordAuthenticationToken, as the AbstractAuthenticationToken.isAuthenticated() will return false.

> `UsernamePasswordAuthenticationToken(Object principal, Object credentials, Collection<? extends GrantedAuthority> authorities)`<br>
This constructor should only be used by AuthenticationManager or AuthenticationProvider implementations that are satisfied with producing a trusted

第一个参数就是我们的 UserDetails, 第二个 credentials 代表了用来证明 Principle 的凭证（比如Password），第三个就是困扰我们的 authorities。而在我们的 `JwtAuthenticationFilter.java` 中，同事少些了一个参数：
```
UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
						userDetails, Arrays.asList(new SimpleGrantedAuthority("USER")));
```

这也就因为多态导致调用了第一种构造器，并错误得将 authority 放进了 每个 token 的 credentials 里面。糟糕的是这并不会报出任何错误，导致我们会很难发现问题所在。解决方法就是在两个参数间加一个 null 参数。

## 查看当前 Token 所拥有的权限
可以使用下面这句查看：
```
System.out.println(SecurityContextHolder.getContext().getAuthentication().getAuthorities());
```