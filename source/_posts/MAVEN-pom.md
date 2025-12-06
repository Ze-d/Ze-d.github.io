---
title: MAVEN pom
date: 2025-11-28 17:22:02
tags: java mavena
---

Maven POM.xml 配置完整指南

## 1. 项目坐标标识

### 基础坐标配置

```markdown
<!-- 项目坐标 - 唯一标识 -->
<groupId>com.example</groupId>      <!-- 组织/公司域名反写 -->
<artifactId>demo-project</artifactId> <!-- 项目名称 -->
<version>1.0.0</version>            <!-- 版本号 -->
<packaging>jar</packaging>          <!-- 打包方式: jar, war, pom -->
```

### 项目元数据
```xml
<name>Demo Project</name>           <!-- 项目名称 -->
<description>项目描述信息</description>  <!-- 项目描述 -->
<url>http://www.example.com</url>   <!-- 项目URL -->
```

## 2. 依赖管理配置

### 依赖声明
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
        <version>5.3.0</version>
        <scope>compile</scope>      <!-- 依赖范围 -->
        <optional>false</optional>  <!-- 是否可选 -->
        <exclusions>
            <exclusion>
                <groupId>commons-logging</groupId>
                <artifactId>commons-logging</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>
```

### 依赖范围说明
| Scope      | 编译 | 测试 | 运行 | 打包 | 示例        |
| ---------- | ---- | ---- | ---- | ---- | ----------- |
| `compile`  | ✅    | ✅    | ✅    | ✅    | spring-core |
| `provided` | ✅    | ✅    | ❌    | ❌    | servlet-api |
| `runtime`  | ❌    | ✅    | ✅    | ✅    | JDBC驱动    |
| `test`     | ❌    | ✅    | ❌    | ❌    | JUnit       |
| `system`   | ✅    | ✅    | ❌    | ❌    | 本地jar     |

### 依赖版本管理
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.28</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 3. 构建配置详解

### 资源文件配置
```xml
<build>
    <resources>
        <resource>
            <!-- 资源文件目录 -->
            <directory>src/main/resources</directory>
            
            <!-- 是否启用过滤（占位符替换） -->
            <filtering>true</filtering>
            
            <!-- 包含的文件模式 -->
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            
            <!-- 排除的文件模式 -->
            <excludes>
                <exclude>**/*.tmp</exclude>
            </excludes>
            
            <!-- 目标路径（打包后的位置） -->
            <targetPath>config</targetPath>
        </resource>
    </resources>
</build>
```

### 过滤功能示例
```properties
# 资源文件内容
app.name=@project.name@
app.version=@project.version@

# 打包后实际内容
app.name=MyApplication
app.version=1.0.0
```

### 插件配置详解
```xml
<build>
    <plugins>
        <plugin>
            <!-- 插件坐标 -->
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            
            <!-- 插件配置 -->
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>UTF-8</encoding>
            </configuration>
            
            <!-- 执行配置 -->
            <executions>
                <execution>
                    <id>default-compile</id>
                    <phase>compile</phase>
                    <goals>
                        <goal>compile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 常用核心插件

#### 编译器插件
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.11.0</version>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <encoding>UTF-8</encoding>
        <parameters>true</parameters>
    </configuration>
</plugin>
```

#### 打包插件
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.3.0</version>
    <configuration>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <mainClass>com.example.MainApplication</mainClass>
            </manifest>
        </archive>
    </configuration>
</plugin>
```

#### 依赖插件
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>3.6.0</version>
    <executions>
        <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 4. 项目继承和聚合

### 父项目继承
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.0</version>
    <relativePath/> 
</parent>
```

### 多模块聚合
```xml
<!-- 父项目 packaging 必须为 pom -->
<packaging>pom</packaging>

<modules>
    <module>user-service</module>
    <module>order-service</module>
    <module>gateway</module>
</modules>
```

## 5. 属性配置

### 自定义属性
```xml
<properties>
    <!-- Java版本 -->
    <java.version>1.8</java.version>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
    
    <!-- 项目编码 -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    
    <!-- 依赖版本 -->
    <spring.version>5.3.0</spring.version>
    <junit.version>5.7.0</junit.version>
</properties>
```

## 6. 环境配置

### 仓库配置
```xml
<repositories>
    <repository>
        <id>central</id>
        <name>Central Repository</name>
        <url>https://repo.maven.apache.org/maven2</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

### 部署配置
```xml
<distributionManagement>
    <repository>
        <id>company-releases</id>
        <url>http://nexus.company.com/repository/maven-releases</url>
    </repository>
    <snapshotRepository>
        <id>company-snapshots</id>
        <url>http://nexus.company.com/repository/maven-snapshots</url>
    </snapshotRepository>
</distributionManagement>
```

### 多环境配置
```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <env>dev</env>
            <db.url>jdbc:mysql://localhost:3306/dev</db.url>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
    
    <profile>
        <id>prod</id>
        <properties>
            <env>prod</env>
            <db.url>jdbc:mysql://prod-db:3306/prod</db.url>
        </properties>
    </profile>
</profiles>
```

**激活方式**：
```bash
mvn clean install -P dev
mvn clean install -Denv=prod
```

## 7. 其他重要配置

### 报告生成
```xml
<reporting>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-javadoc-plugin</artifactId>
            <version>3.3.1</version>
        </plugin>
    </plugins>
</reporting>
```

### 开发者信息
```xml
<developers>
    <developer>
        <id>dev001</id>
        <name>张三</name>
        <email>zhangsan@company.com</email>
        <roles>
            <role>architect</role>
        </roles>
    </developer>
</developers>
```

## 8. 实际应用场景

### 多环境资源配置
```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <env>dev</env>
        </properties>
        <build>
            <resources>
                <resource>
                    <directory>src/main/resources</directory>
                    <filtering>true</filtering>
                    <includes>
                        <include>application-${env}.properties</include>
                    </includes>
                </resource>
            </resources>
        </build>
    </profile>
</profiles>
```

### 统一版本管理
```xml
<properties>
    <spring-boot.version>2.7.0</spring-boot.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 完整构建配置示例
```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
        </resource>
    </resources>
    
    <plugins>
        <!-- 编译器配置 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <source>11</source>
                <target>11</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
        
        <!-- 打包配置 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>3.3.0</version>
            <configuration>
                <archive>
                    <manifest>
                        <mainClass>com.example.MainApp</mainClass>
                    </manifest>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 9. 关键总结

### 核心标签作用

| 标签                         | 作用           | 重要性 |
| ---------------------------- | -------------- | ------ |
| `groupId/artifactId/version` | 项目唯一标识   | ★★★★★  |
| `dependencies`               | 管理项目依赖   | ★★★★★  |
| `build/resources`            | 资源文件处理   | ★★★★★  |
| `build/plugins`              | 构建过程控制   | ★★★★★  |
| `properties`                 | 定义属性和变量 | ★★★★☆  |
| `parent`                     | 继承父项目配置 | ★★★★☆  |
| `modules`                    | 多模块项目管理 | ★★★★☆  |
| `profiles`                   | 环境特定配置   | ★★★☆☆  |

### Maven 生命周期阶段

| 阶段       | 描述             |
| ---------- | ---------------- |
| `validate` | 验证项目是否正确 |
| `compile`  | 编译源代码       |
| `test`     | 运行测试         |
| `package`  | 打包编译后的代码 |
| `verify`   | 运行集成测试     |
| `install`  | 安装到本地仓库   |
| `deploy`   | 部署到远程仓库   |

### 最佳实践

1. **统一版本管理**：使用 `dependencyManagement` 管理依赖版本
2. **环境分离**：使用 `profiles` 管理不同环境配置
3. **属性化配置**：在 `properties` 中定义可复用变量
4. **资源过滤**：利用 `filtering` 实现配置文件的动态替换
5. **插件管理**：统一插件版本，避免版本冲突

---

*本文档总结了 Maven POM.xml 文件中各个配置标签的详细作用和用法，可作为日常开发的参考手册。*

**文档版本**: 1.0  
**最后更新**: 2024年1月  
**适用对象**: Java开发人员、Maven使用者

```

## 保存步骤

1. **复制内容**：选中上面的整个代码块并复制
2. **创建文件**：在您的计算机上新建一个文本文件
3. **粘贴内容**：将复制的内容粘贴到文件中
4. **保存文件**：将文件保存为 `Maven-POM-Configuration-Guide.md`

## 替代方案

如果您希望获得格式更好的版本，建议：

1. **使用Markdown编辑器**：如 Typora、VS Code 等
2. **在线转换**：使用 pandoc 或其他工具转换为 PDF
3. **GitHub Gist**：将内容创建为 GitHub Gist 便于分享和版本控制

这个文档包含了所有讨论过的 Maven POM 配置细节，可以作为日常开发的参考资料。
