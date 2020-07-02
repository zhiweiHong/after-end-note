# SkyEngine 学习笔记

## 1. 目录结构

| 目录结构          | 作用                                |
| ----------------- | ----------------------------------- |
| xx-module-def     | API定义模块，接口模块，不负责实现。 |
| xx-module-impl    | API具体的实现模块。                 |
| xx-module-web     | 负责提供与web交互的接口             |
| xx-module-openapi | 负责提供接口，包括rest，mq，thrift  |
| xx-module-deploy  | 负责整合项目。                      |

### 1.1 目录划分

```bash
base-aaa-module-impl-{子模块名}
  |-src/main/java       // 组件实现工程的Java源码，包括API的实现，service层、DAO层和实体bean
  |-src/main/resources  // 组件实现工程的资源，会包含少量spring配置，国际化文件
    |-i18N              // 国际化文件的目录
	  |-messages_zh_CN.properties   // 国际化文件
	|-sql-script        // 数据库脚本目录
	  |-default         // default数据库中的初始化脚本，具体参见数据库脚本规范
	  |-log             // log数据库中的初始化脚本
	  |-statistics      // 统计或时序数据的数据库中的初始化脚本
  |-src/test/java       // 组件实现工程的单元测试，包含 src/main/java 工程内所有代码的单元测试
  |-src/test/resources  // 单元测试相关的资源，通常这个目录为空
```

### 1.2 包结构

#### 1.2.1 def

```bash
/src/com/ruijie/rcos/rco/module/def
    |-- api
        |-- request
        |-- response
    |-- spi
    	|-- request
        |-- response
    |-- dto
    |-- enums
    |-- msg 
    |-- interceptor
    |-- constant
    |-- wrapper
    |-- annotation
```

#### 1.2.2 impl

```bash
/src/com/ruijie/rcos/rco/module/impl
    |-- api
    |-- dao
    |-- entity
    |-- dto
    |-- service 
        |-- impl
    |-- msg
    |-- quartz
    |-- task
    |-- tx
    |-- constants
    |-- util
```

#### 1.2.3 web

```bash
/src/com/ruijie/rcos/rco/module/web
    |-- ctrl
    |-- request
    |-- validation
    |-- vo
```

## 2. 规范相关

### 2.1 通用命名规范

- 驼峰法命名
- 禁止使用英文和拼音混合使用方式。禁止使用全拼音方式。
- 抽象类以`Abstract`开头
- 接口不加`I`前缀，实现类命名规则为`接口名称+impl`
- 内部用的复杂业务数据模型以DTO结尾
- Restful接口相应嵌套实体以VO结尾
- 以相应功能名作为结尾，例如：`Interceptor`、`Task`、`Quartz`、`Exception`、`Test`、`Util`

### 2.2 组件相关规范

#### 2.2.1 def

- 定义接口以API结尾
- 组件定义接口请求参数以Request结尾，且实现`com.ruijie.rcos.sk.modulekit.api.comm.Request` 接口
  - 规定每一个方法定义一个`Request`实体接受
- 组件定义的接口响应参数以`Reponse`结尾，并且继承`com.ruijie.rcos.sk.modulekit.api.comm.DefaultResponse` 类
  - 规定每一个方法定义一个`Reponse`返回

#### 2.2.2 impl

- 持久层以DAO结尾，并且继承`SkyEngineJpaRepository` 接口

- 数据库实体以Entity结尾

- 业务逻辑实体以Service结尾

- 包含本地事务的业务接口以Tx结尾

- 常量定义类以Constants结尾

- 国际化key接口文件统一命名为 `BusinessKey`，存放在对应根包下

  - 禁止用硬编码写死异常信息
  - key命名改规则`[工程名前缀]_[模块名]_[业务]`

  ```java
  public interface BusinessKey {
  
      /** 管理员不存在 */
      String BASE_AAA_ADMIN_NOT_EXIST = "base-aaa_admin_admin_not_exist";
      
      /** 管理员不存在或密码错误 */
      String BASE_AAA_LOGIN_ADMIN_NOT_EXIST_OR_PWD_ERROR = "base_aaa_login_admin_not_exist_or_pwd_error";
  }
  ```

#### 2.2.3 组件WEB工程相关

- Controller层类以Controller结尾
- Controller层参数验证以Validation结尾

   ```java
@Controller
@RequestMapping("/rco/admin")
@EnableCustomValidate(validateClass = AdminCustomValidation.class)
public class AdminManageController {
	@RequestMapping(value = "loginAdmin", method = RequestMethod.POST)
    @NoAuthUrl
    @EnableCustomValidate(validateMethod = "loginAdminValidate")
    public DefaultWebResponse loginAdmin(LoginAdminWebRequest request, SessionContext sessionContext,
            ProgrammaticOptLogRecorder optLogRecorder) throws BusinessException {
    	// do some business
        return null;
    }    
}
   ```

### 2.3 方法约定

- 方法名首字母小写，驼峰命名
- 布尔返回值以`is`,`has`,`can`,`should`或者能明确表达返回值的。例如`needTranslate`
  - 最好不使用`is`作为前缀
- 获取单个对象建议以`get`作为前缀
- 获取多个对象建议以`list`作为前缀
- 进行统计值的建议以`count`做前缀
- 插入方法建议用`save`或者`insert`做前缀
- 修改方法建议用`update`做前缀
- 删除方法建议用`remove`或者`delete`做前缀
- 事务补偿方法以`rollback`做前缀，命名规则为`Rollback`+具体业务方法

### 2.4 参数规定

- API 和 SPI 接口定义的参数需实现 `com.ruijie.rcos.sk.modulekit.api.comm.Request` 接口
- API 接口定义的 **分页查询** 请求参数需继承 `DefaultPageRequest`
- API 接口定义的 **返回值** 需继承  `com.ruijie.rcos.sk.modulekit.api.comm.DefaultResponse` 类
- API 接口定义的 **分页列表** 返回类型使用 `DefaultPageResponse<dto>`
- API 接口定义使用 `@Rollback`、`@NoRollback` 标注失败后是否需要事务补偿
- Controller 层接口请求参数必须实现 `com.ruijie.rcos.sk.webmvc.api.request.WebRequest` 接口
- Controller 层接口的列表 **分页查询** 参数需继承 `PageWebRequest`
- Controller 层接口的请求类型为 `@RequestMapping(method=POST)`
- Controller 层接口的返回值为 `DefaultWebResponse`

### 2.5 JPA使用规定

- 业务表必须带有`version int` 字段，用于实现乐观锁
- 表实体统一以Entity结尾。并加上`@Table`注解对应具体业务表
- 主键使用UUID类型
- 使用`@Version` 标注`version`字段，JPA支持乐观锁实现
- DAO为接口类型，需要实现接口`SkyEngineJpaRepository<entity, idType>`
- DAO 中更新记录时，使用 `version` 进行乐观锁控制。例如：`@Query("update entity set c1=?2, version=version+1 where id=?1 and version=?3")`

## 3. 线程池

- 禁止直接使用java的线程池。
- 禁直接使用Thread.start()方法创建线程。
- 建议交给spring进行托管。

### 3.1 简单案例

```java
ThreadExecutors.execute("简单任务", ()->{
    // DO SOMETHING
});
```

### 3.2 执行一个待回执的简单任务

```java
final Future<String> future = ThreadExecutors.submit("待回执的简单任务", () -> {
	// do something
	return "<执行结果>";
});

// do other thing

// 获取执行结果
final String result = future.get();

// 根据执行结果进行其他业务处理
```

### 3.3 执行一个延迟任务

```java
ThreadExecutors.scheduleAtFixedRate("以固定的间隔运行任务", ()->{}, 1000, 5000);
ThreadExecutors.scheduleWithFixedDelay("以固定的延迟运行任务", ()->{}, 1000, 5000);
ThreadExecutors.scheduleWithCron("以Cron表达式运行任务", ()->{}, "0 0 0 1 * ? ");
```

### 3.4 创建自定义线程池

```java
final ThreadExecutor executor = ThreadExecutors.newBuilder("自定义线程池") //
		.maxThreadNum(10) //
		.queueSize(100) //
		.addStartupLineNumberToThreadName() //
		.build();

executor.execute(()->{});
```

## 4. 依赖管理规范

- 第三方jar包不允许随便引入和升级，专业组内惩罚措施很恐怖；
- 业务子系统引入第三方包时需要自行进行谨慎评审；
- 业务子系统升级第三方包的流程很繁琐。