## Gradle的Jar冲突解决方案



#### jar冲突问题

`gradlew -q beifu-udp-api-service:dependencies` 或者在idea中help菜单中执行

强制使用某版本jar的方式

- 使用`compile`强制声明
- 在`allprojects`属性中使用`force`强制声明

```groovy
allprojects { project ->
	apply plugin: 'idea'
	apply plugin: 'eclipse'
	
	ext.releaseBuild = version.endsWith('RELEASE')
	ext.snapshotBuild = version.endsWith('SNAPSHOT')

	repositories {
		maven { url vipShopMavenCentral }
	}
	configurations.all {
		//关键点在这里
		resolutionStrategy {
			force 'com.alibaba:fastjson:1.2.52'
		}

	}
}
```













