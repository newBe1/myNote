### 一、自定义Realm，实现授权与认证

```java
public class CrmRealm extends AuthorizingRealm {

    @Autowired
    private IUserService employeeService;
    
    @Autowired
    private IPermissionService permissionService;
    
    //获取权限信息,以便框架判断资源是否拥有权限
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        System.out.println("------执行授权--------");

        User user = (User) getAvailablePrincipal(principals);
        //获取用户成功 对此用户进行授权
        if(user != null){
            SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();

            // 角色与权限字符串集合
            Collection<String> rolesCollection = new HashSet<>();
            Collection<String> premissionCollection = new HashSet<>();
            // 读取并赋值用户角色与权限
            Set<RoleBean> roles = user.getRoles();
            for(RoleBean role : roles){
                rolesCollection.add(role.getName());
                Set<PermissionBean> permissions = role.getPermissions();
                for (PermissionBean permission : permissions){
                    
                    //permission.getPerm()作为 用户访问拦截器中拦截路径的凭证 若和拦截器中配置的拦截路径所需的资源相同 则可以访问 
                    premissionCollection.add(permission.getPerm());
                }

                //将权限添加如授权管理器
                simpleAuthorizationInfo.addStringPermissions(premissionCollection);
            }
            //将角色添加如授权管理器
            simpleAuthorizationInfo.addRoles(rolesCollection);
            return simpleAuthorizationInfo;
        }
        throw new UnknownAccountException("用户不存在");
    }

    
    //获取用户身份信息
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        
        //把token转换为UsernamePasswordToken
        UsernamePasswordToken upt = (UsernamePasswordToken) token;
        //获取用户名
        String username = upt.getUsername();
        //根据用户名获取用户
        User employee = employeeService.findByUsername(username);
        //做逻辑判断
        if (employee == null) {
            throw new UnknownAccountException();
        }
        if (employee.getState() == -1) {
            throw new AuthenticationException("员工已离职！");
        }
        //身份信息返回给框架，让框架做比对
        Object principal = employee;//身份不仅可以是用户名，还可以是整个用户
        Object hashedCredentials = employee.getPassword();//必须保证数据库里面的密码是加密加盐
        ByteSource credentialsSalt = ByteSource.Util.bytes(Md5Util.SALT);
        return new SimpleAuthenticationInfo(principal, hashedCredentials , credentialsSalt , getName());
    }


}
```



### 二、Shiro配置类

 

```java
@Configuration
public class ShiroConfig {

    @Autowired
    private IPermissionService permissionService;
    
    @Bean("hashedCredentialsMatcher")
    public HashedCredentialsMatcher hashedCredentialsMatcher() {
        HashedCredentialsMatcher credentialsMatcher =
                new HashedCredentialsMatcher();
        //指定加密方式为MD5
        credentialsMatcher.setHashAlgorithmName("MD5");
        //加密次数
        credentialsMatcher.setHashIterations(1024);
        credentialsMatcher.setStoredCredentialsHexEncoded(true);
        return credentialsMatcher;
    }

    @Bean("customRealm")
    public CustomRealm customRealm(@Qualifier("hashedCredentialsMatcher")
                                       HashedCredentialsMatcher matcher) {

        CustomRealm customRealm = new CustomRealm();
        customRealm.setCredentialsMatcher(matcher);
        return customRealm;
    }


    //设置拦截器  对访问的接口进行拦截和过滤
    @Bean
    public ShiroFilterFactoryBean shirFilter(@Qualifier("securityManager")
                                                     DefaultWebSecurityManager securityManager) {

        //安全管理器
        ShiroFilterFactoryBean bean = new ShiroFilterFactoryBean();
        // 设置 SecurityManager
        bean.setSecurityManager(securityManager);
        // 设置登录成功跳转Url
        bean.setSuccessUrl("/main");
        // 设置登录跳转Url
        bean.setLoginUrl("/toLogin");
        // 设置未授权提示Url
        bean.setUnauthorizedUrl("/error/unAuth");

        /**
         * anon：匿名用户可访问
         * authc：认证用户可访问
         * user：使用rememberMe可访问
         * perms：对应权限可访问
         * role：对应角色权限可访问
         * map集合中以url 为key  shiro对这些url 进行拦截
         **/
        Map<String, String> filterMap = new LinkedHashMap<>();
        filterMap.put("/login","anon");
        filterMap.put("/user/index","authc");
        filterMap.put("/vip/index","roles[vip]");
        filterMap.put("/druid/**", "anon");
        filterMap.put("/static/**","anon");

        filterMap.put("/**","authc");
        filterMap.put("/logout", "logout");
        //对需要权限才能访问的资源地址配置拦截，只要不再这儿配置都任务不需要任何权限，直接放行

        //只要在Permission表的资源都要做权限拦截  以permission.getUrl() 作为拦截路径
        List<Permission> allPermissions = permissionService.getAll();
        for (Permission permission : allPermissions) {
            //reslt.put("/authorized.html", "perms[user:add]");//由于是代码，可以注入Service从数据库中查询
            filterMap.put(permission.getUrl(), "crmPerms["+permission.getPerm()+"]");
        }
        bean.setFilterChainDefinitionMap(filterMap);
        return bean;
    }

    /**
     * 注入 securityManager
     */
    @Bean(name="securityManager")
    public DefaultWebSecurityManager getDefaultWebSecurityManager(
            HashedCredentialsMatcher hashedCredentialsMatcher) {

        DefaultWebSecurityManager securityManager =
                new DefaultWebSecurityManager();
        // 关联realm.
        securityManager.setRealm(customRealm(hashedCredentialsMatcher));
        return securityManager;
    }

    //加入注解的使用，不加入这个注解不生效
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(@Qualifier("securityManager")DefaultWebSecurityManager securityManager) {
        AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
        authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
        return authorizationAttributeSourceAdvisor;
    }
}
```



### 数据库设计

- 用户表

```sql
CREATE TABLE `user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(20) NOT NULL COMMENT '用户id',
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `password` varchar(50) NOT NULL,
  `salt` varchar(128) DEFAULT NULL COMMENT '加密盐值',
  `email` varchar(50) DEFAULT NULL COMMENT '邮箱',
  `phone` varchar(50) DEFAULT NULL COMMENT '联系方式',
  `sex` int(255) DEFAULT NULL COMMENT '年龄：1男2女',
  `age` int(3) DEFAULT NULL COMMENT '年龄',
  `status` int(1) NOT NULL COMMENT '用户状态：1有效; 2删除',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  `last_login_time` datetime DEFAULT NULL COMMENT '最后登录时间',
  PRIMARY KEY (`id`,`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=24 DEFAULT CHARSET=utf8;
```

- 角色表

```sql
CREATE TABLE `role` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `role_id` varchar(20) NOT NULL COMMENT '角色id',
  `name` varchar(50) NOT NULL COMMENT '角色名称',
  `description` varchar(255) DEFAULT NULL COMMENT '角色描述',
  `status` int(1) NOT NULL COMMENT '状态：1有效；2删除',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;
```

- 权限表

```sql
CREATE TABLE `permission` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `permission_id` varchar(20) NOT NULL COMMENT '权限id',
  `name` varchar(100) NOT NULL COMMENT '权限名称',
  `description` varchar(255) DEFAULT NULL COMMENT '权限描述',
  `url` varchar(255) DEFAULT NULL COMMENT '权限访问路径',
  `perm` varchar(255) DEFAULT NULL COMMENT '权限标识',
  `parent_id` int(11) DEFAULT NULL COMMENT '父级权限id',
  `type` int(1) DEFAULT NULL COMMENT '类型   0：目录   1：菜单   2：按钮',
  `order_num` int(3) DEFAULT '0' COMMENT '排序',
  `icon` varchar(50) DEFAULT NULL COMMENT '图标',
  `status` int(1) NOT NULL COMMENT '状态：1有效；2删除',
  `create_time` datetime DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=32 DEFAULT CHARSET=utf8;
```

- 用户角色关联表

```sql
CREATE TABLE `user_role` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` varchar(20) NOT NULL COMMENT '用户id',
  `role_id` varchar(20) NOT NULL COMMENT '角色id',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
```

- 角色用户关联表

```sql
CREATE TABLE `role_permission` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `role_id` varchar(20) NOT NULL COMMENT '角色id',
  `permission_id` varchar(20) NOT NULL COMMENT '权限id',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=869 DEFAULT CHARSET=utf8;
```