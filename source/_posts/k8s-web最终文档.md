---
title: K8S-WEB最终文档
category: K8S
tag: 
 - K8S
---

# K8S-WEB项目文档

## 需要了解的前置术语

* 前端：进行数据渲染和展示的部分。通常使用HTML CSS JavaScript实现
* 后端：对数据进行增删改查，提供服务的部分
* VUE：前端框架，将HTML CSS JavaScript 整合到一个页面，便于开发
* VueRouter：Vue的组件，用来快速配置页面跳转
* Axios：Vue的组件，用来发起和配置Http请求。
* NPM：包管理工具，用来管理Vue项目的依赖包
* ElementPlus：Vue3的组件库，用来快速开发，美化页面。
* tomcat：web服务器，监听端口。SpringBoot使用tomcat来开启服务。
* SpringBoot：后端框架，用来快速开发服务端
* Maven：包管理工具，用来管理后端项目
* MyBatisPlus：用来快速开发服务端对数据库增删改查操作的框架
* MySQL：数据库
* SpringCloud：SpringBoot的微服务生态，包含多种框架
* SpringCloudKubernetes：SpringCloud的k8s组件。由官方维护
* JWT：JavaWebToken，是SpringBoot的组件，用来进行鉴权
* Lombok：是SpringBoot的组件，用来一键设置类的get，set方法
* HttpClient：是SpringBoot的组件，用来发起http请求。
* Swagger：是SpringBoot组件，用来统一api。
* token：用来鉴权的序列
* Cookie：是浏览器中暂时存储数据的一个数据结构。项目中在这里存储token和username
* Interceptor：拦截器，在后端处理请求前拦截请求，并进行操作
* Nginx：用来分发http请求，将请求根据端口分配到不同资源
* Docker
* Kubernetes
* Kubevirt

## 后端

### 整体代码包类型解读

<img src="http://gohoy.top/i/2023/08/04/p6aqbg-1.png" alt="image-20230804152221174" style="zoom:80%;" />

**可以很粗略的认为：所有其他的类都是为controller文件服务的**

* `com.example.home.gohoy.k8s_backend`：此目录下的是所有代码
  * assets：静态文件
  * config：配置文件
  * controller：核心提供服务的文件，这里一般只调用service接口实现需求
  * dao：数据库交互的核心文件，这里都实现了MyBatisPlus
  * dto：用于前后端传输的类
  * entities：实体类（实体对象）
  * service：业务实现层
  * utils：工具类
  * K8sWebMainApplication.class：springBoot的启动类，整个项目由此进入
* resource：存放springBoot配置文件
  * application.yml：spring boot配置文件。
  * k8s-user.sql：sql表文件，与项目逻辑无关。
* pom.xml：图中没有展示，这个文件是maven管理工具的配置文件，使用maven来控制依赖版本

### 后端代码的api框架

<img src="http://gohoy.top/i/2023/08/01/qoznsh-1.png" alt="系统结构功能图"  />

分别对应了代码的controller部分

![image-20230804163908243](http://gohoy.top/i/2023/08/04/r3vqkm-1.png)

### 数据库表

数据库名称为：k8s

只有一张表：users，文件见resource/k8s-users.mysql

#### 表user字段说明

| ColumName    | id   | username         | password | ctr_occupied            | ctr_name                  | ctr_max                     | vm_occupied            | vm_name                | vm_max                     | is_admin     | token               | last_login   |
| ------------ | ---- | ---------------- | -------- | ----------------------- | ------------------------- | --------------------------- | ---------------------- | ---------------------- | -------------------------- | ------------ | ------------------- | ------------ |
| describe     | 主键 | 用户名，不可重复 | 密码     | 已经申请的container数量 | 已经申请的container的名称 | 最多可以申请的container数量 | 已经申请的虚拟机的数量 | 已经申请的虚拟机的名称 | 最多可以申请的虚拟机的数量 | 是否是管理员 | 用于验证身份的token | 上次登录时间 |
| notNull      | yes  | yes              | yes      |                         |                           |                             |                        |                        |                            | yes          | yes                 |              |
| default      |      |                  |          | 0                       | null                      | 1                           | 0                      | null                   | 1                          | 0            |                     | null         |
| autoIncrease | yes  |                  |          |                         |                           |                             |                        |                        |                            |              |                     |              |

### 处理请求的逻辑

<img src="http://gohoy.top/i/2023/08/04/r5tcvh-1.png" alt="image-20230804152329594" style="zoom: 67%;" />

#### tomcat服务器如何把请求转发到对应的controller方法？

接下来以用户登录方法为例

我们假设后端服务器的BASE_URL是http://localhost:8080/

* 前端使用POST请求访问了http://localhost:8080/user/login，并且携带了用户的基本信息

* tomcat检测到请求，将请求转发给Springboot框架

* SpringBoot框架通过你的注解找到对应这个url对应的方法

  * 这个方法位于controller/user/UserController.class下

  * UserController.class这个文件需要注解

    * <img src="http://gohoy.top/i/2023/08/04/r5tald-1.png" alt="image-20230804152933250" style="zoom: 67%;" />

    * `@CrossOrigin("*")`

      允许跨域访问到这个方法

    * `@ApiResponses`

      插件Swagger的注解，用来整合所有api的信息，对于这个服务，可以在http://localhost:8080/swagger-ui.html查看所有注册的api，并测试

    * `@RestController()`

      这个注解将这个类作为Restful风格的controller注册到SpringBoot框架中，

    * `@RequestMapping("/user/")`

      这个注解表示这个类接收url为http://localhost:8080/user/的任意方法请求

  * 然后查看login的方法

    * <img src="http://gohoy.top/i/2023/08/04/r5t8ma-1.png" alt="image-20230804153323097" style="zoom:80%;" />

    * `@PostMapping("/login")`

      表示这个方法处理http://localhost:8080/user/login的请求

    * `@ApiResponse(description = "用户登录")`

      注册这个方法到swagger管理页面中，这个api的描述为“用户登录”

    * `@RequestBody User user`

      这是需要传入这个函数的参数，对应前端携带的用户信息。并把这个用户信息生成一个User类的对象

    * 接下来就是对user对象进行操作。

#### controller具体逻辑的实现流程

以根据用户名获取用户数据这个api为例

![image-20230804164347717](http://gohoy.top/i/2023/08/04/r6hzsn-1.png)

* 它的返回值是CommonResult，这是我自定义的类（utils/CommonResult.class），用于规范像前端返回的数据格式。
* 调用了UserService的getUserByName方法
  * **![image-20230804164546919](http://gohoy.top/i/2023/08/04/r7oqsr-1.png)**
  * UserService这里只定义了一个接口，没有进行实现
* UserServiceImpl.class文件实现了UserService接口的方法
  * <img src="http://gohoy.top/i/2023/08/04/r8urkd-1.png" style="zoom:67%;" />
* UserServiceImpl中又调用了UserDao中的方法来从数据库查询用户
  * ![](http://gohoy.top/i/2023/08/04/r9dp3f-1.png)
  * 这里面什么都没写，因为它继承了BaseMapper类，BaseMapper是MyBatisPlus框架实现的类，其中已经有最基本的对数据库进行增删改查的方法。

##### 为什么要有这么多流程呢？直接把在controller文件中调用UserDao实现查询可以吗

分出controller service dao三层是后端开发的规范，在比较复杂的后端逻辑时可以理清思路，便于实现。

所以简单的业务逻辑是可以直接在controller实现的。

###  controller文件中实现api

####  一般的api，简单的增删改查

##### `AdminPodController`：管理员Pod管理api

* `/getAllPods/{type})`：获取所有pod信息
  * 使用kubernetesClient获取所有pod，将pod进行筛选，除去系统pod
  * 将剩下的pod每一个都构建成PodInfo的对象，返回List<PodInfo>
* `/setCtrDefaultResource/`：设置默认的pod资源，需要传入PodResourceDTO对象
  * 这个api实现的方法是：将podResourceDTO对象的属性存放在k8s集群中的configMap中。
* `/setVMDefaultResource/`：与`/setCtrDefaultResource/`同理
* `/getDefaultConfig/{type}`：根据字符串type获取不同的configMap

##### `AdminUserController`：管理员用户管理api：

* `/getUserByName/{username}`：通过用户名从数据库查询该用户的信息
* `/getUsersByPage/{pageNum}/{pageSize}`：分页获取全部用户，使用了MyBatisPlus内置的分页方法。
* `/updateUser`：更新用户信息
* `/deleteUser/{id}`：通过id删除用户

##### `PodController`：用户对pod的功能需求

* `/selectPodByUserName/{username}`：通过用户名查询该用户所拥有的pod。因为pod创建的时候的名称都是username-type-job-random的形式，这里通过把所有的pod导出，遍历筛选的方法获得目标。
* `/deletePod/{podName}`：通过pod名称来删除pod

##### `UserController`：用户登录注册和其他对自己信息的管理

* `/register`：传入用户名，密码，只要数据库中没有这个用户就进行插入。
* `/getUserDTO/{username}`：通过用户名，获取除了密码以外的所有信息。
* `/getIndex`：前端主页显示的内容，通过此处获取index.md进行展示

####  较为复杂的api逻辑分析

#####  `PodController`：用户对pod的功能需求

* `/createCtr/{username}`：用户开启一个pod
  * 使用username获取用户信息，判断用户是否还有可用的pod数量。
  * 从configmap中获取该类型pod的资源数量
  * 按照资源限制依次创建PV，PVC，Service，Job。PV和PVC用来持久化用户信息，Service用来暴露pod的ssh端口，Job用来控制pod的属性（比如存活时间等）
  * 创建成功之后将clusterIp，sshPort，rootPassword返回。
* `/createVm/{username}`：TODO

##### `UserController`：用户登录注册和其他对自己信息的管理

* `/login`：用户登录，传入username和password
  * 首先验证用户名密码是否正确
  * 然后按照用户的username，isAdmin属性来使用jwt生成一个token。过期时间默认一周
  * 然后纠正用户可用的pod数量，因为当pod的job自然结束后，没有钩子函数能够通知我们去修改用户可用pod数量。所以这里在登录的时候进行扫描纠正。
  * 返回给前端token，前端在访问其他api的时候都需要在cookie中携带username和token

###  kubevirt相关api实现的步骤

#### ~~ 走通使用kubevirt进行创建虚拟机的过程，并选取一个最终实现的方案~~

##### ~~将虚拟机镜像传给pvc，然后使用这个pvc创建一个vm（已完成）~~

##### ~~使用数据类型datavolume来创建虚拟机~~

~~优势：可以通过一个模板来clone虚拟机。通过快照来保存虚拟机的数据~~

~~但是从来没有用过这种方法。可行性待验证~~

#### ~~ 通过kubevirt提供的api，实现这个方案~~

##### ~~ 首先要对请求进行鉴权配置（已完成）~~

~~k8s默认api需要进行证书验证。所以如果直接访问api地址会被拦截。~~

~~我仿照kubernetesClient进行了证书验证的操作。此项已完成~~

##### ~~ 验证将pod配置序列化为请求体正常发送请求的方法~~

~~因为在api中，pod的配置全部都被放在请求体中，而Java存储这些配置的方法是生成一个类。~~

~~这里需要去验证如何把类中的数据正确传递给k8s~~

#### kubevirt最终方案

使用脚本监听文件修改，通过监听文件修改来执行kubectl命令来进行虚拟机操作（当前使用的方案）

```sh
#!/bin/bash

# 目标目录

target_directory="/data/upload_pvc_commands"

# 启动监听

inotifywait -m -e create -e moved_to "$target_directory" |
    while read path action file; do
        if [[ "$file" == *.vm ]]; then
            vm_name="${file%.vm}"
            echo $vm_name
            IMAGE_PATH="/home/gohoy/iso/ubuntu-20.04.qcow2"
            PV_NAME="$vm_name-pv"
            PVC_NAME="$vm_name-pvc"
            PVC_SIZE="45Gi"
            UPLOADPROXY_URL=$(kubectl -n cdi get svc -l cdi.kubevirt.io=cdi-uploadproxy | awk '/cdi-uploadproxy/ {print $3}')
            WAIT_SECS="240"
            echo "upload images to $PVC_NAME"
            sed -e "4s/pv/$PV_NAME/; 14s/pv/$PV_NAME/; 19s/pv/$PV_NAME/; 29s/pv/$PV_NAME/" /data/upload_pvc_commands/pv.yaml >/data/upload_pvc_commands/tmppv.yaml
            kubectl apply -f /data/upload_pvc_commands/tmppv.yaml
            # sed -e "4s/pvc/$PVC_NAME/; 11s/45Gi/$PVC_SIZE/ " /data/upload_pvc_commands/pvc.yaml >/data/upload_pvc_commands/tmppvc.yaml
            chmod -R 777 /data/*
            virtctl image-upload --image-path="$IMAGE_PATH" --pvc-name="$PVC_NAME" \
                --pvc-size="$PVC_SIZE" --uploadproxy-url="$UPLOADPROXY_URL" \
                --insecure --wait-secs="$WAIT_SECS"

             chmod -R 777 /data/*
             virtctl image-upload --image-path="$IMAGE_PATH" --pvc-name="$PVC_NAME" \
                --pvc-size="$PVC_SIZE" --uploadproxy-url="$UPLOADPROXY_URL" \
                --insecure --wait-secs="$WAIT_SECS"

            echo "upload images done"
            sed -e "4s/vm/$vm_name/; 11s/vm/$vm_name/; 33s/pvc/$vm_name-pvc/" /data/upload_pvc_commands/vm.yaml >/data/upload_pvc_commands/tmpvm.yaml

            kubectl apply -f /data/upload_pvc_commands/tmpvm.yaml
            rm -f /data/upload_pvc_commands/$vm_name.vm

        elif [[ "$file" == *.delete ]]; then
            vm_name="${file%.delete}"
            echo "Deleting VM: $vm_name"
            kubectl delete vm "$vm_name"
            kubectl delete pvc "$vm_name-pvc"
            kubectl delete pv "$vm_name-pv"

            rm -f "$target_directory/$vm_name.delete"

        fi

    done
```



## 前端

### 前端代码包

![image-20230814160921521](http://gohoy.top/i/2023/08/14/qm6mnh-1.png)

* node_modules：所有前端组件的目录。由npm管理
* public：放一些公共文件，比如网站的头像，index.html
* src：主要代码的存放处
  * assets：静态资源
  * components：组件，需要重复使用的部分
  * router：路由，管理页面的跳转
  * views：主要的页面存放处
  * App.vue：整个项目的入口
  * main.js：整个项目的配置文件

### 前端文件的架构

<img src="http://gohoy.top/i/2023/08/14/qqtpor-1.png" alt="image-20230814161708391" style="zoom:67%;" />

页面的入口是App.vue

App.vue中的`   <router-view></router-view>`是这个页面被渲染的地方。

## 框架的使用说明

### NPM

NodeJs提供的包管理工具

npm install <packageName> 安装某个包

npm install 按照npm的配置文件package.json来安装所有依赖

在前端项目中，需要Node注意版本。

npm run <option> 将项目运行，一般会有 serve 和build 选项。前者用于开发。后者将项目打包，用于部署

### Vue

vue ui：开启一个webCli来进行vue项目管理

vue init <packageName> 创建一个vue项目（可以在web页面进行）

### VueRouter

```sh
npm install vue-router
//在创建vue项目的时候可以选择直接安装
```

用于页面跳转，整合在整个vue项目中。

```js
//   router/index.js示例
import { createRouter, createWebHashHistory } from 'vue-router'
import IndexView from '@/views/indexView.vue'
const routes = [
  {
    path: '/index',
    name: 'home',
    component: IndexView
  }
]

const router = createRouter({
  history: createWebHashHistory(),
  routes
})

export default router
```

```js
// main.js中需要配置如下
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

const app = createApp(App);
app.use(router).mount('#app')

```

这样之后，就可以访问项目地址/index进入页面IndexView

### axios

```sh
npm insatll axios
```

用于发送http请求

```js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'
import axios from 'axios'
axios.defaults.baseURL = "http://localhost:8080/"
//设置基本路径，在之后的请求中不需要重复输入这段url
axios.interceptors.request.use((config) => {
    const token = Cookies.get('token');
    const username = Cookies.get('username');
    if (token) {
        config.headers['Authorization'] = token;
    }

    if (username) {
        config.headers['X-Username'] = username;
    }

    config.headers['Referrer-Policy'] = 'no-referrer'
    return config;
});
//这个interceptor用来解决跨域问题
const app = createApp(App);
app.config.globalProperties.$axios = axios
app.config.globalProperties.$http = axios
// 将axios的别名设置为$axios或者$http
app.use(router).mount('#app')

```

示例使用

```js
        async handleLogin() {
            const response = await this.$axios.post('/user/login', this.userData).then(response => {
                if (response.data.code == "200") {
                    Cookies.set('token', response.data.data);
                    Cookies.set('username', this.userData.username);
                    ElMessage.info(response.data.message)
                    location.reload();
                } else {
                    ElMessage.error(response.data.message)
                }
            })

        },
```

这里使用post请求访问http://localhost:8080/user/login，并在requestBody中携带了用户登录信息。并将得到的返回值放在response中。之后进行操作

### ElementPlus

[查看官方使用文档](https://element-plus.org/zh-CN/guide/quickstart.html#%E7%94%A8%E6%B3%95)

### Maven

包配置文件在项目根目录的pom.xml中

基本示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.1.2</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>home-gohoy-k8s_backend</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
			<version>3.0.3</version>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>

```

主要需要添加依赖的是dependency标签。

修改完成后，使用mvn install 进行安装依赖

mvn package 按照build标签把项目进行打包，用于部署

配置文件：一般是根目录的.m2/setting.xml

maven换源：

```xml
	<mirror>
  	    <id>nexus-aliyun</id>
  		<name>Nexus aliyun</name>
        <mirrorOf>external:*</mirrorOf>
  		<url>http://maven.aliyun.com/nexus/content/groups/public</url>
  	</mirror>
```

在mirrors标签中加入这些即可

### SpringBoot

Maven包设置：

```xml
	<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
			<version>3.1.2</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
			<version>3.1.0</version>
			<scope>compile</scope>
		</dependency>
```

配置：

```yaml
spring:
  application:
    name:k8s-web
server:
  port: 8080
```



主启动类的格式：

```java
@SpringBootApplication
public class K8sWebMainApplication {
	public static void main(String[] args) {
		SpringApplication.run(K8sWebMainApplication.class, args);
	}
}
```

这些是最基本的，且不变的。

可能会需要其他配置。在类名前加上不同的注解。

运行这个类，SpringBoot就会在指定端口开启一个tomcat服务器用来接收http请求。（默认8080）

配置文件默认在项目根目录/resources/application.yaml（或者application.application）

### MyBatisPlus

Maven包引用

```xml
	<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>mybatis-plus-boot-starter</artifactId>
			<version>3.5.3.1</version>
		</dependency>
```



用来简化SpringBoot和数据库的操作

配置：

```yaml
# applicaiton.yaml
mybatis-plus:
  mapper-locations: classpath:mapper/*.xml
```

这里配置了mapper的路径，让SpringBoot能够找到Mybatis的系列配置，但是基础的crud用不到这些mapper文件

基础使用：

```java
// dao文件需要继承BaseMapper<entity>，里面实现的基础的crud
@Table(name = "users")
public interface UserDao extends BaseMapper<User> {
}
```

```java
// service文件需要继承Iservice<entity>，里面定义了基本的crud接口
public interface UserService extends IService<User> {
}
```

```java
// 在serviceImpl文件中继承 ServiceImpl<Dao文件 , entity> ，里面实现了Iservice<entity>的接口
@Service
public class UserServiceImpl extends ServiceImpl<UserDao , User> implements UserService {
}

```

### MySQL

这里准备使用docker来启动

``docker run --name mysql  --restart=always -p 3306:3306 \
-e "MYSQL_ROOT_PASSWORD=040424" mysql ``

修改远程连接：``update user set host='*' where user='root'&& host='localhost';
flush privileges;``

然后使用可视化工具比如MySQLWorkBrench来进行表的设计和操作。

Maven包配置

```xml
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>8.0.33</version>
</dependency>
```

SpringBoot配置

```yam
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/k8s?user=root&password=040424&useUnicode=true&characterEncoding=utf-8&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=Asia/Shanghai
    driver-class-name: com.mysql.cj.jdbc.Driver
```

这里主要要配置url。

在这里配置之后，MyBatisPlus的curd操作就能够找到数据源

### SpringCloudKubernetes

Maven 配置

```xml
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-kubernetes-fabric8-all</artifactId>
			<version>3.0.3</version>
		</dependency>
```

SpringBoot 配置

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
```

基本使用：首先要把k8s的配置文件默认在/etc/kubernetes/admin.conf  复制到用户根目录的 .kube/config

然后开始编码：

```java
package com.example.home.gohoy.k8s_backend.config;

import io.fabric8.kubernetes.client.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class KubernetesConfig {

    @Bean
    public KubernetesClient kubernetesClient() {
        Config config = new ConfigBuilder().withMasterUrl("https://192.168.111.140:6443").build();
        return new KubernetesClientBuilder().withConfig(config).build();
    }
}

```

这段代码将kubernetesClient注入到SpringBoot容器中，生成一个实例。

然后使用的时候直接调用它提供的api

```java
   config = new Namespace();
   ObjectMeta metadata = new ObjectMeta();
   metadata.setName("default");
   config.setMetadata(metadata);
   kubernetesClient.resource(config).createOrReplace();
```

这段代码使用kubernetesClient创建了一个namespace config

### JWT

用来生成token，来进行鉴权

Maven配置

```xml
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-api</artifactId>
			<version>0.11.5</version> <!-- Replace with the latest version -->
		</dependency>
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-impl</artifactId>
			<version>0.11.5</version> <!-- Replace with the latest version -->
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-jackson</artifactId>
			<version>0.11.5</version> <!-- Replace with the latest version -->
			<scope>runtime</scope>
		</dependency>
```



自定义一个JWTUtil.java

```java
package com.example.home.gohoy.k8s_backend.utils;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.security.Key;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;

public class JWTUtils {
    private static final String SECRET_KEY = "TheFurthestDistanceInTheWorldIsNotBetweenLifeAndDeathButWhenIStandInFrontOfYouYetYouDonNotKnowThatILoveYou";
    // 生成JWT令牌
    public static String generateToken(String username,byte isAdmin, long expirationMillis) {
        Date now = new Date();
        Date expiration = new Date(now.getTime() + expirationMillis);
        Map<String, Object> claims = new HashMap<>();
        // 在claims中添加用户信息或其他声明
        claims.put("username", username);
        claims.put("isAdmin",isAdmin);
        // 可以添加更多的声明，比如用户角色等
        byte[] apiKeySecretBytes = SECRET_KEY.getBytes(StandardCharsets.UTF_8);
        Key signingKey = new SecretKeySpec(apiKeySecretBytes, SignatureAlgorithm.HS256.getJcaName());
        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(expiration)
                .signWith(signingKey)
                .compact();
    }
    // 验证JWT令牌，如果验证失败将抛出异常，否则返回声明（claims）
    public static Claims verifyToken(String token) {
        byte[] apiKeySecretBytes = SECRET_KEY.getBytes(StandardCharsets.UTF_8);
        Key signingKey = new SecretKeySpec(apiKeySecretBytes, SignatureAlgorithm.HS256.getJcaName());
        return Jwts.parserBuilder()
                .setSigningKey(signingKey)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
}

```

提供两个方法

* generateToken：使用用户的username和isAdmin字段和过期时间和自定义的Sercert_Key来混淆生成一个token
* verifyToken：将token进行解码，并返回

基本使用：

在登录成功后生成一个token，在接收到前端的请求时，使用token进行鉴权。

### Lombok

用来快速生成类的get set方法，快速标记链式编程等

Maven配置：

```xml
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<version>1.18.26</version>
			<scope>compile</scope>
		</dependency>
```

基本使用：

```java
package com.example.home.gohoy.k8s_backend.entities;
import com.baomidou.mybatisplus.annotation.TableName;
import com.example.home.gohoy.k8s_backend.dto.UserDTO;
import jakarta.persistence.*;
import lombok.Data;
import java.sql.Timestamp;
import java.util.Objects;
@Data
@Entity
@TableName("users")
@Table(name = "users", schema = "k8s")
public class User  extends UserDTO{
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Id
    @Column(name = "id")
    private int id;
    @Basic
    @Column(name = "password")
    private String password;
}
```

直接注解@Data

自动为下面的属性id 和password生成get set方法

### Swagger

Maven配置

```xm
		<dependency>
			<groupId>org.springdoc</groupId>
			<artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
			<version>2.0.4</version>
		</dependency>
```

基本使用：

```java
package com.example.home.gohoy.k8s_backend.controller.admin;

import com.example.home.gohoy.k8s_backend.dao.UserDao;
import com.example.home.gohoy.k8s_backend.dto.UserDTO;
import com.example.home.gohoy.k8s_backend.entities.User;
import com.example.home.gohoy.k8s_backend.service.user.UserService;
import com.example.home.gohoy.k8s_backend.utils.CommonResult;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import jakarta.annotation.Resource;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@CrossOrigin("*")
@ApiResponse
@RequestMapping("/admin/")
public class AdminUserController {
    @Resource
    private UserService userService;
    @Resource
    private UserDao userDao;
    @GetMapping("/getUserByName/{username}")
    @ApiResponse(description = "通过参数userName获取user数据")
    private CommonResult getUserByName(@PathVariable("username") String userName){
  }
    @GetMapping("/getUsersByPage/{pageNum}/{pageSize}")
    @ApiResponse(description = "分页获取所有用户")
    private CommonResult getUsers(@PathVariable("pageNum") int pageNum,@PathVariable("pageSize")int pageSize){
    }
    @PostMapping("/updateUser")
    @ApiResponse(description = "更新用户信息")
    public CommonResult updateUser(@RequestBody User user ){

    }
}

```

@ApiResponses：标记这个类是定义了很多api

@ApiResponse（description="这里是api的描述"）：标记这个方法是一个api

然后在项目运行的url，例如：localhost:8080/swagger-ui.html可以查看这些api，并且可以进行测试

### Interceptor

Maven配置

```xml
<!--		servlet-->
		<dependency>
			<groupId>javax.servlet</groupId>
			<artifactId>javax.servlet-api</artifactId>
			<version>4.0.1</version> <!-- Replace with the appropriate version -->
			<scope>provided</scope>
		</dependency>
```



Interceptor主要实现接口HandlerInterceptor的方法preHandle

这个方法是在controller处理请求前执行的。

```java
package com.example.home.gohoy.k8s_backend.utils.interceptors;

import com.example.home.gohoy.k8s_backend.utils.JWTUtils;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import javax.servlet.http.HttpServletResponse;
@Component
public class AdminInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(jakarta.servlet.http.HttpServletRequest request, jakarta.servlet.http.HttpServletResponse response, Object handler) throws Exception {
        if(request.getMethod().equals("OPTIONS")){
            return true;
        }
        System.out.println("AdminInterceptor");
        // 从请求头部获取 Authorization Cookie
        String token = request.getHeader("Authorization");

        if  (JWTUtils.verifyToken(token).get("isAdmin").equals(1)){
            return true;
        }else {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            return false;
        }
    }
}

```

这段代码的逻辑，就在拦截请求，通过token来获取用户的username，然后从数据库查询是否存在admin权限。

拦截器SpringBoot注入文件

```java
package com.example.home.gohoy.k8s_backend.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class InterceptorConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AdminInterceptor())
                .addPathPatterns("/admin/**");
        registry.addInterceptor(new LoginInterceptor())
                .addPathPatterns("/**")
                .excludePathPatterns("/user/login", "/user/register", "/swagger-ui.html", "/swagger-ui/**",
                        "/webjars/swagger-ui/**", "/v3/api-docs/**");
    }
}

```

这里exludePathPatterns是排除的路径，防止登录注册和swagger页面不能进入。

### HttpClient（当前项目弃用）

用来发送http请求。

使用的原因：Kubevirt的Api没有整合好的Java包，需要自己进行http请求

Maven配置：

```xml
		<dependency>
			<groupId>org.apache.httpcomponents</groupId>
			<artifactId>httpclient</artifactId>
			<version>4.5.14</version>
		</dependency>
```

进行kubevirt编程：

1.鉴权：（是从kubernetesClient调试的来的）

```java
package com.example.home.gohoy.k8s_backend.utils;

import com.example.home.gohoy.k8s_backend.entities.kubevirt.*;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.fabric8.kubernetes.client.KubernetesClientException;
import jakarta.validation.constraints.NotNull;
import okhttp3.*;
import okhttp3.HttpUrl;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import org.springframework.stereotype.Component;

import javax.net.ssl.*;
import java.io.*;
import java.security.*;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.RSAPrivateCrtKeySpec;
import java.util.ArrayList;
import java.util.Base64;
import java.util.Collection;
import java.util.HashMap;
import java.util.stream.Collectors;

@Component
public class KubevirtUtil {


    private final String clientCertResource = "";
    private String caCertResource = "";
private TrustManager[] trustManagers;

    public SSLContext preLoad() throws Exception {
//        System.out.println(caCertResource);
//        System.out.println(clientCertResource);
        String clientKeyResource = "";
//        System.out.println(clientKeyResource);
        InputStream caCert = createInputStreamFromBase64EncodedString(caCertResource);
        InputStream clientCert = createInputStreamFromBase64EncodedString(clientCertResource);
        InputStream clientKey = createInputStreamFromBase64EncodedString(clientKeyResource);
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        KeyStore trustStore = KeyStore.getInstance("pkcs12");
        char[] trustStorePassphrase = "changeit".toCharArray();
        trustStore.load(null);
        while (caCert.available() > 0) {
            CertificateFactory certFactory = CertificateFactory.getInstance("X509");
            X509Certificate cert = (X509Certificate) certFactory.generateCertificate(caCert);
            String alias = cert.getSubjectX500Principal().getName() + "_" + cert.getSerialNumber().toString(16);
            trustStore.setCertificateEntry(alias, cert);
        }
        tmf.init(trustStore);
         trustManagers = tmf.getTrustManagers();
      // clientKey clientCrt
        CertificateFactory certFactory = CertificateFactory.getInstance("X509");
        Collection<? extends Certificate> certificates = certFactory.generateCertificates(clientCert);
        PrivateKey privateKey ;
        byte[] keyBytes = decodePem(clientKey);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        try {
            // First let's try PKCS8
            privateKey = keyFactory.generatePrivate(new PKCS8EncodedKeySpec(keyBytes));
        } catch (InvalidKeySpecException e) {
            // Otherwise try PKCS8
            RSAPrivateCrtKeySpec keySpec = PKCS1Util.decodePKCS1(keyBytes);
            privateKey= keyFactory.generatePrivate(keySpec);
        }
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        keyStore.load(null);
        String alias = certificates.stream().map(cert->((X509Certificate)cert).getIssuerX500Principal().getName()).collect(Collectors.joining("_"));
        keyStore.setKeyEntry(alias, privateKey, trustStorePassphrase, certificates.toArray(new Certificate[0]));
        KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        kmf.init(keyStore, trustStorePassphrase);
        KeyManager[] keyManagers = kmf.getKeyManagers();

        //sslContext
        SSLContext sslContext ;
        try {
             sslContext = SSLContext.getInstance("TLSv1.2");
            sslContext.init(keyManagers, trustManagers, new SecureRandom());
        } catch (KeyManagementException | NoSuchAlgorithmException e) {
            throw KubernetesClientException.launderThrowable(e);
        }
        return sslContext;
    }

    public String sendHttpRequest(@NotNull String method,@NotNull String path, Headers headers, RequestBody requestBody) throws Exception {
        SSLContext sslContext = preLoad();

        OkHttpClient httpClient = new OkHttpClient.Builder()
                .sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustManagers[0])
                .hostnameVerifier((hostname, session) -> true)
                .build();

        HttpUrl url = HttpUrl.parse("https://192.168.111.140:6443" + path);
        Request.Builder requestBuilder = new Request.Builder()
                .url(url)
                .method(method, requestBody);
        System.out.println(url);

        if (headers != null) {
            requestBuilder.headers(headers);
        }

        Request request = requestBuilder.build();

        try (Response response = httpClient.newCall(request).execute()) {
            ResponseBody responseBody = response.body();
            if (responseBody != null) {
                String result = responseBody.string();
                System.out.println(result);
                return result;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        return null;
    }



    private static ByteArrayInputStream createInputStreamFromBase64EncodedString(String data) {
        byte[] bytes;
        try {
            bytes = Base64.getDecoder().decode(data);
        } catch (IllegalArgumentException illegalArgumentException) {
            bytes = data.getBytes();
        }

        return new ByteArrayInputStream(bytes);
    }

    private static byte[] decodePem(InputStream keyInputStream) throws IOException {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(keyInputStream))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.contains("-----BEGIN ")) {
                    return readBytes(reader, line.trim().replace("BEGIN", "END"));
                }
            }
            throw new IOException("PEM is invalid: no begin marker");
        }
    }
    private static byte[] readBytes(BufferedReader reader, String endMarker) throws IOException {
        String line;
        StringBuilder buf = new StringBuilder();

        while ((line = reader.readLine()) != null) {
            if (line.contains(endMarker)) {
                return Base64.getDecoder().decode(buf.toString());
            }
            buf.append(line.trim());
        }
        throw new IOException("PEM is invalid : No end marker");
    }



    public String getRequestBody() throws JsonProcessingException {
        HashMap<String, String> labels = new HashMap<>();
        labels.put("slt", "vm-test-1");

        ArrayList<Disk> disks = new ArrayList<>();
        disks.add(new Disk().setName("containerinit").setBootOrder(1)
                .setDisk(new DiskTarget().setBus("virtio"))
        );
        ArrayList<Interface> interfaces = new ArrayList<>();
        interfaces.add(new Interface().setName("default").setMasquerade("{}"));

        ArrayList<Network> networks = new ArrayList<>();
        networks.add(new Network().setName("default").setPod(new PodNetwork()));

        ArrayList<Volume> volumes = new ArrayList<>();
        volumes.add(new Volume()
                .setName("containerinit")
                .setPersistentVolumeClaim(
                        new PersistentVolumeClaimVolumeSource()
                                .setClaimName("euler-test-1")
                )
        );

        VirtualMachine virtualMachine = new VirtualMachine();
        virtualMachine.setApiVersion("kubevirt.io/v1")
                .setKind("VirtualMachine")
                .setMetadata(new ObjectMeta().setName("vm-test-1"))
                .setSpec(new VirtualMachineSpec().setRunning(true)
                        .setTemplate(new VirtualMachineInstanceTemplateSpec()
                                .setMetadata(new ObjectMeta()
                                        .setNamespace("default")
                                        .setLabels(labels)
                                ).setSpec(
                                        new VirtualMachineInstanceSpec()
                                                .setDomain(
                                                        new DomainSpec().setDevices(new Devices()
                                                                .setAutoattachGraphicsDevice(true)
                                                                .setDisks(disks)
                                                                .setInterfaces(interfaces)
                                                        ).setResources(new ResourceRequirements()
                                                                .setRequests(new Requests()
                                                                        .setCpu("2")
                                                                        .setMemory("1G")
                                                                )
                                                        )
                                                ).setNetworks(networks)
                                                .setVolumes(volumes)
                                )
                        )
                );
        ObjectMapper objectMapper = new ObjectMapper();
        String requestBody = objectMapper.writeValueAsString(virtualMachine);
        System.out.println(requestBody);
        return  requestBody;
    }

    public String sendHttpRequest(String method, String path) throws Exception {
        return sendHttpRequest(method,path,null,null);
    }
}
```

这里的密钥是从k8s的config中复制得到

鉴权操作主要是进行了证书的配置。

2.调用kubevirt Api（弃用）

遇到的困难：缺少一个完整kubevirt环境进行测试。

3.实现kubevirt的方案：使用脚本监听文件修改来执行sh命令。

### Nginx

在部署的环境使用，可以查看[这篇教程](https://www.runoob.com/w3cnote/nginx-setup-intro.html)

### docker

Docker用于作为k8s的container runtime

以及最终项目的打包部署

## 部署

可以参考一些文章。[vue+springboot docker部署](https://blog.csdn.net/qq_52030824/article/details/127982206)

基本步骤：

前端vue 使用npm run build，得到打包后的文件dist，将这个文件放在docker内部nginx的指定目录下。

后端springboot需要使用maven打包成jar包。打包过程参考[教程](https://www.cnblogs.com/sanjay/p/11828081.html)

得到这些包之后，编写dockerfile 文件，将这些容器整合运行。

* 打包得到前端和后端的最终文件
* docker Java17环境
  * 将jar包放入这个镜像，使用Java -jar xxx.jar来运行后台包
* docker MySQL8.0.32环境
  * 将数据库文件写入容器中
* docker Nginx环境
  * 将前端打包文件放入此容器，然后编写nginx配置文件。

### 实操步骤

1. 后端配置好端口后使用maven的package命令打包，得到jar包

   1. 修改端口，MySQL的url：把localhost修改成服务器ip

2. 前端配置好端口后，使用npm build得到dist文件夹

   1. 修改端口和后端的base_url：把base_url修改成服务器ip

3. 然后docker拉取镜像openjdk:17  mysql:8.0.32  nginx:latest

4. 然后写dockerfile文件，将jar包引入为镜像

   1. ```dockerfile
      # 使用合适的 JDK 17 基础镜像
      FROM openjdk:17
      
      # 复制 jar 包到容器
      COPY home-gohoy-k8s_backend-0.0.1-SNAPSHOT.jar k8s_web.jar
      
      # 运行 jar 包
      CMD ["java", "-jar", "/k8s_web.jar"]
      
      # 暴露端口
      EXPOSE 8088
      ```

   2. ``docker build -t k8s-web-server:1.0 .``

   3. 使用docker compose 配置项目需要的镜像的环境

      1. ```yaml
         version: "3"
         services:
           k8s-web:
             image: k8s-web-server:1.0
             ports:
               - "8088:8088"
             volumes:
               - /data/upload_pvc_commands/:/data/upload_pvc_commands/
               - /root/.kube/:/root/.kube/
               - /home/gohoy/k8s_web_docker/assets/:/data/assets/
           mysql:
             image: mysql:8.0.32
             container_name: mysql
             environment:
               MYSQL_ROOT_PASSWORD: '040424'
               MYSQL_ALLOW_EMPTY_PASSWORD: 'no'
               MYSQL_DATABASE: 'k8s'
             ports:
                - "3306:3306"
             networks:
               - k8s_web_network
         
         
           nginx:
             image: nginx:latest
             container_name: nginx
             ports:
               - "80:80"
             volumes:
               - /home/gohoy/k8s_web_docker/nginx/html:/usr/share/nginx/html
               - /home/gohoy/k8s_web_docker/nginx/logs:/var/log/nginx
               - /home/gohoy/k8s_web_docker/nginx/conf:/etc/nginx
             networks:
               - k8s_web_network
         
         # 创建自定义网络
         networks:
            k8s_web_network:
         ```

      2. docker-compose up -d启动环境

   4. 配置nginx

      1. 在/home/gohoy/k8s_web_docker/nginx/conf下配置nginx 的配置文件，nginx.conf 和 mime.types 是从容器里面copy出来的默认配置，如何在conf.d中放入 k8s_web.conf文件

      2. ```conf
         server {
             listen 80;
             location / {
                 root /usr/share/nginx/html/dist; # 根据您的目录配置
                 index index.html;
             }
             access_log /var/log/nginx/access.log;
             error_log /var/log/nginx/error.log;
         }
         ```

      3. 把前端vue打包好的dist文件放在/usr/share/nginx/html/下面

   5. 配置MySQL

      1. 直接在自己电脑使用MySQL work branch（或者同类工具如navicat）连接
      2. 刷入sql脚本文件

   6. 启动主机上 `/home/gohoy/k8s_web_docker/watch_and_execute.sh`，这个脚本来监听/data/upload_pvc_commands/文件夹，通过监听文件修改来执行kubevirt相关的命令

      1. ```sh
         #!/bin/bash
         
         # 目标目录
         
         target_directory="/data/upload_pvc_commands"
         
         # 启动监听
         
         inotifywait -m -e create -e moved_to "$target_directory" |
             while read path action file; do
                 if [[ "$file" == *.vm ]]; then
                     vm_name="${file%.vm}"
                     echo $vm_name
                     IMAGE_PATH="/home/gohoy/iso/ubuntu-20.04.qcow2"
                     PV_NAME="$vm_name-pv"
                     PVC_NAME="$vm_name-pvc"
                     PVC_SIZE="45Gi"
                     UPLOADPROXY_URL=$(kubectl -n cdi get svc -l cdi.kubevirt.io=cdi-uploadproxy | awk '/cdi-uploadproxy/ {print $3}')
                     WAIT_SECS="240"
                     echo "upload images to $PVC_NAME"
                     sed -e "4s/pv/$PV_NAME/; 14s/pv/$PV_NAME/; 19s/pv/$PV_NAME/; 29s/pv/$PV_NAME/" /data/upload_pvc_commands/pv.yaml >/data/upload_pvc_commands/tmppv.yaml
                     kubectl apply -f /data/upload_pvc_commands/tmppv.yaml
                     # sed -e "4s/pvc/$PVC_NAME/; 11s/45Gi/$PVC_SIZE/ " /data/upload_pvc_commands/pvc.yaml >/data/upload_pvc_commands/tmppvc.yaml
                     chmod -R 777 /data/*
                     virtctl image-upload --image-path="$IMAGE_PATH" --pvc-name="$PVC_NAME" \
                         --pvc-size="$PVC_SIZE" --uploadproxy-url="$UPLOADPROXY_URL" \
                         --insecure --wait-secs="$WAIT_SECS"
         
                      chmod -R 777 /data/*
                      virtctl image-upload --image-path="$IMAGE_PATH" --pvc-name="$PVC_NAME" \
                         --pvc-size="$PVC_SIZE" --uploadproxy-url="$UPLOADPROXY_URL" \
                         --insecure --wait-secs="$WAIT_SECS"
         
                     echo "upload images done"
                     sed -e "4s/vm/$vm_name/; 11s/vm/$vm_name/; 33s/pvc/$vm_name-pvc/" /data/upload_pvc_commands/vm.yaml >/data/upload_pvc_commands/tmpvm.yaml
         
                     kubectl apply -f /data/upload_pvc_commands/tmpvm.yaml
                     rm -f /data/upload_pvc_commands/$vm_name.vm
         
                 elif [[ "$file" == *.delete ]]; then
                     vm_name="${file%.delete}"
                     echo "Deleting VM: $vm_name"
                     kubectl delete vm "$vm_name"
                     kubectl delete pvc "$vm_name-pvc"
                     kubectl delete pv "$vm_name-pv"
         
                     rm -f "$target_directory/$vm_name.delete"
         
                 fi
         
             done
         ```

         
