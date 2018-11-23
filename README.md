### 今天主要实践一下利用gradle打包并上传aar到私有maven仓库的一些事情

- 第一步，要上传aar到maven仓，起码得有一个maven仓吧，这里我们使用的是Nexus作为私有仓库，如果还没有Nexus仓库，请自行百度；

- 第二步，要有一个带上传的module,例如本项目中的testlib和mytestlib;

- 第三步，编写上传脚本，具体参考maven_upload.gradle;


> ### 脚本编写，先设置maven仓库的地址，上传aar需要授权，所以要填入用户名和密码
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

                println "build type: artifactId = " + artifactId
                pom(_flavorBuildTypeName).groupId = repositoryGroup
                pom(_flavorBuildTypeName).artifactId = artifactId
                pom(_flavorBuildTypeName).version = "1.0.3-SNAPSHOT"
                pom(_flavorBuildTypeName).name = artifactId
                pom(_flavorBuildTypeName).packaging = 'aar'
            }
         }
     }
  }
```

### 多渠道源码上传，未完待续...








