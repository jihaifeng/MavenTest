### 今天主要实践一下利用gradle打包并上传aar到私有maven仓库的一些事情

> ### 准备
- 第一步，要上传aar到maven仓，起码得有一个maven仓吧，这里我们使用的是Nexus作为私有仓库，如果还没有Nexus仓库，请自行百度；

- 第二步，要有一个带上传的module,例如本项目中的testlib和mytestlib;

- 第三步，编写上传脚本，具体参考maven_upload.gradle;


> ### 设置maven仓库的地址，上传aar需要授权，所以要填入用户名和密码
```
apply plugin: 'maven'

uploadArchives {
  repositories {
        mavenDeployer {
            // release仓库
            repository(url: releaseRepositoryUrl) {
                // 仓库的用户名和密码
                authentication(userName: mavenUserName, password: mavenPassword)
            }
             // snapShot仓库
             snapshotRepository(url: snapshotRepositoryUrl) {
                authentication(userName: mavenUserName, password: mavenPassword)
            }
            ...
       }
  }
}
```
## 注意!注意！注意！      
release仓同一个版本号只能上传一次，snapShot仓相同版本号可多次上传。

> ### 设置gradle依赖的pom文件
- 简单pom,不涉及多渠道
```
uploadArchives {
  repositories {
        mavenDeployer {
            ...
            pom.groupId = groupId
            pom.artifactId = artifactId
            pom.version = version
            pom.packaging = 'aar'
       }
  }
}
```

- 涉及多渠道aar的上传
```
uploadArchives {
  repositories {
        mavenDeployer {
            ...
            android.libraryVariants.all { variant ->
                def _flavorBuildTypeName = variant.name

                println "build type: _flavorBuildTypeName = " + _flavorBuildTypeName
                addFilter(_flavorBuildTypeName) { artifact, file ->
                        true
                   }
                pom(_flavorBuildTypeName).groupId = groupId
                pom(_flavorBuildTypeName).artifactId = artifactId
                pom(_flavorBuildTypeName).version = "1.0.3-SNAPSHOT"
                pom(_flavorBuildTypeName).packaging = 'aar'
            }
         }
     }
  }
```
## 注意!注意！注意！
 这里上传时的version有讲究，version后边加上 "-SNAPSHOT" 表示上传到snapShot仓，否则表示上传到Release仓。

> ### 项目中使用
- 普通使用
```
// 使用release仓里面的aar
compile 'groupId:artifactId:version'
// 使用snapShot仓里面的aar
compile 'groupId:artifactId:version-SNAPSHOT'
//例如
compile 'jihf.maven:testlib:1.0.4-SNAPSHOT'
```
- 多渠道的使用
```
 // 多渠道使用需要指定渠道标识flavorName
 compile 'groupId:artifactId:version:flavorName@aar'
 // 例如
  compile 'jihf.maven:testlib:1.0.4-SNAPSHOT:aaDebug@aar'
```

> ### 解决module工程中的远程依赖不可使用的问题
查看gradle源码中的注释
```
* apply plugin: 'java' //so that I can declare 'compile' dependencies
 *
 * dependencies {
 *   compile('org.hibernate:hibernate:3.1') {
 *     //in case of versions conflict '3.1' version of hibernate wins:
 *     force = true
 *
 *     //excluding a particular transitive dependency:
 *     exclude module: 'cglib' //by artifact name
 *     exclude group: 'org.jmock' //by group
 *     exclude group: 'org.unwanted', module: 'iAmBuggy' //by both name and group
 *
 *     //disabling all transitive dependencies of this dependency
 *     transitive = false
 *   }
 * }
```

从源码注释中可以看到源码中设置了transitive ，表示当前依赖是否可以传递，默认是false,讲道理我们在引用时加上这句就可以了，于是我们的引用就变成了这样
```
  compile('groupId:artifactId:version:flavorName@aar') {
        transitive = true
    }
```

这时就会出现新的问题，这样的引用并没什么卵用，于是，我们看了下自己pom文件
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<groupId>jihf.maven</groupId>
<artifactId>testlib</artifactId>
<version>1.0.4-SNAPSHOT</version>
<packaging>aar</packaging>
<name>testlib</name>
</project>
```
这就发现问题了，我们的pom文件里面并没有第三方dependecies,既然发现了问题，就好解决了，修改上传脚本，加上以下代码

```
 pom.withXml {
       def dependenciesNode = asNode().appendNode('dependencies')
       configurations.compile.allDependencies.each {
       def dependencyNode = dependenciesNode.appendNode('dependency')
       dependencyNode.appendNode('groupId', it.group)
       dependencyNode.appendNode('artifactId', it.name)
       dependencyNode.appendNode('version', it.version)
          }
}
```
当然如果涉及多渠道，还是要加上渠道标识的。

最后我们再次打包上传，查看pom文件
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<groupId>jihf.maven</groupId>
<artifactId>testlib</artifactId>
<version>1.0.4-SNAPSHOT</version>
<packaging>aar</packaging>
<name>testlib</name>
<dependencies>
<dependency>
<groupId>com.android.support</groupId>
<artifactId>appcompat-v7</artifactId>
<version>27.1.1</version>
</dependency>
<dependency>
<groupId>com.squareup.picasso</groupId>
<artifactId>picasso</artifactId>
<version>2.4.0</version>
</dependency>
</dependencies>
</project>
```
哇擦，真的有了第三方的依赖，再次编译主工程，就会发现可以使用aar中依赖的三方库了，欣喜之情难以言表，至此泪流满面！！！

### 多渠道源码上传，未完待续...








