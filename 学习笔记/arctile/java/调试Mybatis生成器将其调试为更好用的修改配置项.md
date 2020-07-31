## 优化Mybatis生成器

### 源码下载地址

[github地址](https://github.com/zouzg/mybatis-generator-gui)

### 修改项

只需修改`MySQLLimitPlugin`类覆盖example相关配置即可

#### 去除Example相关引入

``` java
package com.zzg.mybatis.generator.plugins;


import org.mybatis.generator.api.IntrospectedTable;
import org.mybatis.generator.api.PluginAdapter;
import org.mybatis.generator.api.dom.java.*;
import org.mybatis.generator.api.dom.xml.Attribute;
import org.mybatis.generator.api.dom.xml.TextElement;
import org.mybatis.generator.api.dom.xml.XmlElement;

import java.util.List;

public class MySQLLimitPlugin extends PluginAdapter {

    @Override
    public boolean validate(List<String> list) {
        return true;
    }



    /**
     * 为每个Example类添加limit和offset属性已经set、get方法
     */
    @Override
    public boolean modelExampleClassGenerated(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;

    }


    public boolean sqlMapExampleWhereClauseElementGenerated(XmlElement element, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean sqlMapCountByExampleElementGenerated(XmlElement element, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean sqlMapDeleteByExampleElementGenerated(XmlElement element, IntrospectedTable introspectedTable) {
        return false;
    }



    public boolean sqlMapSelectByExampleWithBLOBsElementGenerated(
            XmlElement element, IntrospectedTable introspectedTable) {
        return false;
    }


    public boolean sqlMapUpdateByExampleSelectiveElementGenerated(
            XmlElement element, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean sqlMapUpdateByExampleWithBLOBsElementGenerated(
            XmlElement element, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean sqlMapUpdateByExampleWithoutBLOBsElementGenerated(
            XmlElement element, IntrospectedTable introspectedTable) {
        return false;
    }

    /**
     * 为Mapper.xml的selectByExample添加limit
     */
    @Override
    public boolean sqlMapSelectByExampleWithoutBLOBsElementGenerated(XmlElement element,
                                                                     IntrospectedTable introspectedTable) {
        return false;
    }




    public boolean clientCountByExampleMethodGenerated(Method method,
                                                       Interface interfaze, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientCountByExampleMethodGenerated(Method method,
                                                       TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientDeleteByExampleMethodGenerated(Method method,
                                                        Interface interfaze, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientDeleteByExampleMethodGenerated(Method method,
                                                        TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }


    public boolean clientSelectByExampleWithBLOBsMethodGenerated(Method method,
                                                                 Interface interfaze, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientSelectByExampleWithBLOBsMethodGenerated(Method method,
                                                                 TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientSelectByExampleWithoutBLOBsMethodGenerated(Method method,
                                                                    Interface interfaze, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientSelectByExampleWithoutBLOBsMethodGenerated(Method method,
                                                                    TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientUpdateByExampleSelectiveMethodGenerated(Method method,
                                                                 Interface interfaze, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientUpdateByExampleSelectiveMethodGenerated(Method method,
                                                                 TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientUpdateByExampleWithBLOBsMethodGenerated(Method method,
                                                                 Interface interfaze, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientUpdateByExampleWithBLOBsMethodGenerated(Method method,
                                                                 TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientUpdateByExampleWithoutBLOBsMethodGenerated(Method method,
                                                                    Interface interfaze, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean clientUpdateByExampleWithoutBLOBsMethodGenerated(Method method,
                                                                    TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }


    public boolean providerCountByExampleMethodGenerated(Method method,
                                                         TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean providerDeleteByExampleMethodGenerated(Method method,
                                                          TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        return false;
    }



    public boolean providerSelectByExampleWithBLOBsMethodGenerated(
            Method method, TopLevelClass topLevelClass,
            IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean providerSelectByExampleWithoutBLOBsMethodGenerated(
            Method method, TopLevelClass topLevelClass,
            IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean providerUpdateByExampleSelectiveMethodGenerated(
            Method method, TopLevelClass topLevelClass,
            IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean providerUpdateByExampleWithBLOBsMethodGenerated(
            Method method, TopLevelClass topLevelClass,
            IntrospectedTable introspectedTable) {
        return false;
    }

    public boolean providerUpdateByExampleWithoutBLOBsMethodGenerated(
            Method method, TopLevelClass topLevelClass,
            IntrospectedTable introspectedTable) {
        return false;
    }



}

```

#### 优化tinyint类型只映射为Integer的java类型

修改`MybatisGeneratorBridge`类的execute方法

```java
context.setTargetRuntime("MyBatis3");

//在指定targetRuntime下添加指定javaType即可
JavaTypeResolverConfiguration typeResolverConfiguration = new JavaTypeResolverConfiguration();
typeResolverConfiguration.setConfigurationType(MyJavaTypeResolverDefaultImpl.class.getName());
context.setJavaTypeResolverConfiguration(typeResolverConfiguration);
```

对应的修改类`MyJavaTypeResolverDefaultImpl`

```java
import org.mybatis.generator.internal.types.JavaTypeResolverDefaultImpl;

import java.sql.Types;

public class MyJavaTypeResolverDefaultImpl extends JavaTypeResolverDefaultImpl {

    public MyJavaTypeResolverDefaultImpl() {
        super();
        //把数据库的 TINYINT 映射成 Integer
        super.typeMap.put(Types.TINYINT, new JdbcTypeInformation("TINYINT", new FullyQualifiedJavaType(Integer.class.getName())));
        super.typeMap.put(Types.SMALLINT, new JdbcTypeInformation("TINYINT", new FullyQualifiedJavaType(Integer.class.getName())));
    }
}
```

#### 忽略项

在`MainUIController`类的`generateCode`方法中的bridge字段后添加以下代码段

```java
 //添加特殊列处理
ColumnHandlePlugin columnHandlePlugin = new ColumnHandlePlugin();
if (CollectionUtils.isNotEmpty(columnHandlePlugin.getIgnoredColumnList())){
    ignoredColumns  = new ArrayList<>();
    ignoredColumns.addAll(columnHandlePlugin.getIgnoredColumnList());
}
```

新增插件类

```java
package com.zzg.mybatis.generator.plugins;

import org.mybatis.generator.config.ColumnOverride;
import org.mybatis.generator.config.IgnoredColumn;

import java.util.ArrayList;
import java.util.List;

/**
 * 列处理插件
 */
public class ColumnHandlePlugin {
    /**
     * 忽略的列
     */
    private List<IgnoredColumn> ignoredColumnList = new ArrayList<>();



    /**
     * 在这里定义
     */
    public ColumnHandlePlugin() {
        initIgnored();
    }

    /**
     * 初始化忽略项,默认忽略更新时间
     */
    private void initIgnored(){
        ignoredColumnList.add(new IgnoredColumn("update_time"));
    }


    public List<IgnoredColumn> getIgnoredColumnList() {
        return ignoredColumnList;
    }

}

```

#### Mapper自带注解

插件类

```java
public class RepositoryPlugin extends PluginAdapter {

    private FullyQualifiedJavaType annotationRepository;
    //private String annotation = "@Repository";
    private String annotation = "@Mapper";

    public RepositoryPlugin () {
       // annotationRepository = new FullyQualifiedJavaType("org.springframework.stereotype.Repository"); //$NON-NLS-1$
        annotationRepository = new FullyQualifiedJavaType("org.apache.ibatis.annotations.Mapper"); //$NON-NLS-1$
    }

    @Override
    public boolean validate(List<String> list) {
        return true;
    }

    @Override
    public boolean clientGenerated(Interface interfaze, TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        interfaze.addImportedType(annotationRepository);
        interfaze.addAnnotation(annotation);
        return true;
    }
}
```

使用注入插件 : MybatisGeneratorBridge的generate方法添加以下代码

```java
if(generatorConfig.isAnnotationDAO()) {
    if (DbType.MySQL.name().equals(dbType) || DbType.MySQL_8.name().equals(dbType)
        || DbType.PostgreSQL.name().equals(dbType)) {
        PluginConfiguration pluginConfiguration = new PluginConfiguration();
        pluginConfiguration.addProperty("type", "com.zzg.mybatis.generator.plugins.RepositoryPlugin");
        pluginConfiguration.setConfigurationType("com.zzg.mybatis.generator.plugins.RepositoryPlugin");
        context.addPluginConfiguration(pluginConfiguration);
    }
}
//GeneratorConfig类去添加属性以及MainUIController类的getGeneratorConfigFromUI方法设置默认值
```

#### 自动生成Repository类

插件类

```java
public class RepositoryDaoPlugin extends PluginAdapter {
    private ShellCallback shellCallback = null;
    private FullyQualifiedJavaType annotationRepository;
    private String annotation = "@Repository";

    public RepositoryDaoPlugin() {
        shellCallback = new DefaultShellCallback(false);
        annotationRepository = new FullyQualifiedJavaType("org.springframework.stereotype.Repository");
    }

    @Override
    public List<GeneratedJavaFile> contextGenerateAdditionalJavaFiles(IntrospectedTable introspectedTable) {
        JavaFormatter javaFormatter = context.getJavaFormatter();
        String daoTargetDir = context.getJavaClientGeneratorConfiguration().getTargetProject();
        String daoTargetPackage = context.getJavaClientGeneratorConfiguration().getTargetPackage();
        List<GeneratedJavaFile> mapperJavaFiles = new ArrayList<>();
        String javaFileEncoding = context.getProperty("javaFileEncoding");
        //构建类名
        String tableName = context.getTableConfigurations().get(0).getDomainObjectName();
        String repositoryName = tableName + "Repository";
        TopLevelClass repositoryClass = new TopLevelClass(daoTargetPackage + "." + repositoryName);
        //添加公共属性
        repositoryClass.setVisibility(JavaVisibility.PUBLIC);
        //添加引用类
        repositoryClass.addImportedType(annotationRepository);
        //添加引用注解
        repositoryClass.addAnnotation(annotation);




        //生成文件
        GeneratedJavaFile mapperJavafile = new GeneratedJavaFile(repositoryClass, daoTargetDir, javaFileEncoding, javaFormatter);
        try {
            File mapperDir = shellCallback.getDirectory(daoTargetDir, daoTargetPackage);
            File mapperFile = new File(mapperDir, mapperJavafile.getFileName());
            // 文件存在
            if (mapperFile.exists()) {
                mapperFile.delete();
            }
            mapperJavaFiles.add(mapperJavafile);
        } catch (ShellException e) {
            e.printStackTrace();
        }
        return mapperJavaFiles;
    }


    @Override
    public boolean validate(List<String> list) {
        return true;
    }

}

```

使用注入插件 : MybatisGeneratorBridge的generate方法添加以下代码

```java
//添加自动生成Repository插件,
if (generatorConfig.isGenerateRepository()){
    if (DbType.MySQL.name().equals(dbType) || DbType.MySQL_8.name().equals(dbType)
        || DbType.PostgreSQL.name().equals(dbType)) {
        PluginConfiguration pluginConfiguration = new PluginConfiguration();
        pluginConfiguration.addProperty("type", "com.zzg.mybatis.generator.plugins.RepositoryDaoPlugin");
        pluginConfiguration.setConfigurationType("com.zzg.mybatis.generator.plugins.RepositoryDaoPlugin");
        context.addPluginConfiguration(pluginConfiguration);
    }
}
//GeneratorConfig类去添加属性以及MainUIController类的getGeneratorConfigFromUI方法设置默认值
```



[参考](https://segmentfault.com/a/1190000013037968)