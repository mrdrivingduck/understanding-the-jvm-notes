# Chapter 9.2 - Tomcat: 正统的类加载器架构

Created by : Mr Dk.

2020 / 01 / 31 16:32 🧨🧧

Ningbo, Zhejiang, China

---

## 9.2 案例分析

### 9.2.1 Tomcat: 正统的类加载器架构

主流的 Java Web 服务器都实现了自己定义的类加载器，且一般不止一个。要解决的问题：

- 部署在同一服务器上的两个 Web 应用使用的 Java 类库可以互相隔离，因为两个不同的应用程序可能会依赖同一个第三方类库的不同版本
- 部署在同一服务器上的两个 Web 应用使用的 Java 类库可以互相共享，否则 JVM 的方法区容易出现过度膨胀的风险
- 服务器需要保证自身安全不受部署的 Web 应用程序影响，服务器使用的类库应该与应用程序类库互相独立
- HotSwap

Tomcat 的类库目录结构：

- `/common` - 类库可被 Tomcat 和所有的 Web 应用程序共同使用
- `/server` - 类库可被 Tomcat 使用，对所有的 Web 应用程序都不可见
- `/shared` - 类库可被所有的 Web 应用程序共同使用，但对 Tomcat 不可见
- `/WebApp/WEB-INF` - 类库仅仅可以被该 Web 应用程序使用

为了支持这套目录结构，并对目录中的类库进行加载和隔离。Tomcat 自定义了多个类加载器，并按照经典的双亲委派模型实现：

- CommonClassLoader 继承 Java 自带的应用程序类加载器
  - 负责加载 Tomcat 和应用程序共用的类库
- CatalinaClassLoader 和 SharedClassLoader 继承 CommonClassLoader
  - CatalinaClassLoader 加载仅被 Tomcat 使用的类库
  - SharedClassLoader 加载应用程序共用的类库
- WebAppClassLoader 继承 SharedClassLoader
  - 每个 Web 应用程序对应一个 WebApp 类加载器，因此有多个实例
  - 每个 WebApp 类加载器之间隔离
- JasperLoader 继承 WebAppClassLoader
  - 每个 JSP 文件对应一个 JasperLoader 类加载器
  - 仅用于加载某个 JSP 文件编译出的 Class 文件
  - 当服务器检测到 JSP 文件被修改时，会替换当前的 JasperLoader 实例，再建立一个新的 JSP 类加载器实现 JSP 的 HotSwap

在 Tomcat 6 中，为了易用性，默认将所有目录整合为一个 `/lib` 目录，默认只启用 CommonClassLoader。

- 简化大多数部署场景
- 可以通过配置文件指定 `server.loader` 和 `share.loader` 重新启用原来完整的加载器架构
