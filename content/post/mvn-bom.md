---
title: "更科学地使用 maven 管理项目版本"
date: 2021-02-26T20:21:48+08:00
lastMod: 2021-02-26T20:21:48+08:00
tags: ['maven', 'mvn', '模块管理', '分享']
comments: true
---

假设一个 `demo` 项目，按功能划分为组件模块和服务模块，组件模块包含 `component-a` 和 `component-b`，服务模块包含 `service-1` 和 `service-2`。结构如下：

```sh
╰─demo$ tree -L 4 -P *.xml --dirsfirst
.
├── component-bom
│   ├── component-container
│   │   ├── component-a
│   │   │   ├── src
│   │   │   └── pom.xml
│   │   ├── component-b
│   │   │   ├── src
│   │   │   └── pom.xml
│   │   ├── component-c
│   │   │   ├── src
│   │   │   └── pom.xml
│   │   ├── src
│   │   │   ├── main
│   │   │   └── test
│   │   └── pom.xml
│   └── pom.xml
├── service-base
│   ├── service-1
│   │   ├── src
│   │   └── pom.xml
│   ├── service-2
│   │   ├── src
│   │   └── pom.xml
│   ├── src
│   └── pom.xml
└── pom.xml
```

现在想要实现这样的效果：组件可以单独升级，而不会影响其他组件的版本，升级后也不用到每个服务里去修改组件的版本号。

这里就会用到 maven 提供的 **BOM(Bills Of Material)** 功能：在 component-bom 中统一定义所有组件的版本号，并且作为组件模块的根 pom。

为方便讨论会省略 component，如 component-a 会简写为 a：

要开辟一个地方专门用于存放组件模块版本号（也就是使用 BOM）的意义，是降低管理版本号的复杂度：

- bom 中定义组件版本号，当升级组件 a 时，只要 bom 和 a 的版本号升级即可，b、c 组件不受影响
- 服务模块在根 pom 中引入组件的 bom，某个服务需要引入某个组件依赖时，不再需要指定版本，由 bom 统一管理
- 当服务模块需要升级依赖的组件时，只需要修改 bom 版本号即可，不需要在每个服务中单独修改版本号

Spring cloud 的伞项目管理方式就是通过 bom 的方式实现的。

当然也有一定的不方便之处，比如 bom 的中定义的某个组件的版本号不是你想要的，这时候只要在依赖中显式指定你需要的版本号就会覆盖 bom 中给出的版本号。但是确定 bom 中每个组件模块的版本号还是有点点麻烦。

原本作为所有组件模块根的是 container，由于需要在 bom 中定义所有组件模块的版本号，因此在 contaienr 上再引入一层就是比较自然的做法了，这样 bom 就代替 container 成了组件模块的根。

再往上引入一层，而不是在 container 模块的文件夹中新建 bom 的原因在于，maven 会默认父子模块在文件夹中的结构也是父子文件夹，(即 <relativePath> 属性的默认取值是 `../`)。不过如果不愿意修改 container 在文件系统中的位置（移入 bom 之下），也可以将 bom 放在 container 的文件夹内，通过修改 <relativePath> 等属性达到文件夹结构和 maven 项目模块结构颠倒的效果：在文件夹上看 container 是 bom 的外层，在模块结构上看，container 是 bom 的子模块。

之所以可以达到单个组件升级而不影响其他组件的效果，是由于各个组件中声明的 <parent> 指向 container，而组件的版本信息声明在 bom 中，这样实现了解耦，组件 a 升级时，只需要修改组件 a 本身的版本定义，bom 中组件 a 的版本声明以及 bom 的版本，container 则不受影响，其他组件也就不会受影响。

bom 中声明组件版本的定义如下：

```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-a</artifactId>
        <version>1.0</version>
      </dependency>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-b</artifactId>
        <version>1.0</version>
      </dependency>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-c</artifactId>
        <version>1.0</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
```

由于 bom 是组件 a, b, c 的祖父 pom，也就是顺着 parent 继承树就可以找到 <dependencyManagement> 的定义，因此这些组件可以免费享受到 bom 用法带来的便利。

在 service 服务中，则需要在 service-base 的 <dependencyManagement> 中添加如下声明：

```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-bom</artifactId>
        <version>1.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
```

这样，在 service-1, service-2 中就可以直接添加依赖而不需操心版本，版本信息会顺着 service-base 的 <dependencyManagement> 中 component-bom 的声明到 component-bom 的 <dependencyManagement> 中读取。

基本说明告一段落，下面来看一个组件模块升级的例子：如果组件 c 依赖 b，a 是单独的组件，service-1 依赖了 a，service-2 依赖 c，这时 b 有更新，会发生什么。

首先，更新 b 不应该影响 a，service-1 不应该受到影响；

其次，由于 bom 中有着 b 的版本声明，b 升级后，版本声明处也应当随之更新，bom 内容发生变化，bom 版本也随着更新

最后，由于 c 依赖 b，所以 b 升级以后，c 也需要升级；相应的，需要决定 service-2 是否依赖升级后的 c，如果不升级，service-base 中的 bom 声明维持不变即可；如果需要升级，只改一个地方即可全部生效。因为例子里只有一个服务依赖了 c，bom 的优势不是很明显，实际项目中，可能有 7、8 个服务都依赖一个组件，尤其是基础组件，这种情况，还请脑补一下。

修改后的 bom 定义：

```xml
  <parent>
    <artifactId>demo</artifactId>
    <groupId>life.kyon</groupId>
    <version>1.0</version>
  </parent>
  <modelVersion>4.0.0</modelVersion>

  <artifactId>component-bom</artifactId>
  <packaging>pom</packaging>
  <version>2.0</version>

  <modules>
    <module>component-container</module>
  </modules>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-a</artifactId>
        <version>1.0</version>
      </dependency>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-b</artifactId>
        <version>2.0</version>
      </dependency>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-c</artifactId>
        <version>2.0</version>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

组件 b 升级后，service-base 中 bom 声明修改为，即升级 bom 版本号，service-2 就可以直接感知到组件 c 的升级：

```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>life.kyon</groupId>
        <artifactId>component-bom</artifactId>
        <version>2.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
```

可以通过这个 [commit](https://github.com/kyon0304/maven-pom-demo/commit/a25e0149016ae851b45e638942180a23306744fb) 看到更具体的变更。

最后，示例项目已上传至 [github](https://github.com/kyon0304/maven-pom-demo)，master 分支中所有组件版本都是 1.0，update-b 分支中，b 组件升级至 2.0 版本。欢迎参考～

成文仓促，如果有更好的解决方案，或者没说明白的地方，欢迎留言讨论。✨