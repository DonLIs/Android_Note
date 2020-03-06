
```
// 引入maven工具
apply plugin: 'maven'
// 生成Javadoc文档
task androidJavadocs(type: Javadoc) {
    source = android.sourceSets.main.java.srcDirs
    classpath += project.files(android.getBootClasspath().join(File.pathSeparator))
}
// 生成Javadoc文档jar包
task androidJavadocsJar(type: Jar, dependsOn: androidJavadocs) {
    classifier = 'javadoc'
    from androidJavadocs.destinationDir
}

// 防止编码问题
tasks.withType(Javadoc) {
    options.addStringOption('Xdoclint:none', '-quiet')
    options.addStringOption('encoding', 'UTF-8')
    options.addStringOption('charSet', 'UTF-8')
}
// 生成源码文件jar包
task androidSourcesJar(type: Jar) {
    classifier = 'sources'
    from android.sourceSets.main.java.srcDirs
}

artifacts {
    // 源码文件jar包执行命令
    archives androidSourcesJar
    // Javadoc文档jar包执行命令
    archives androidJavadocsJar
}
// 上传命令
uploadArchives {
    repositories {
        mavenDeployer {
            // url 私服仓库地址
            repository(url: "") {
                // userName 私服仓库账号 password 私服仓库账号密码
                authentication(userName: "admin", password: "admin123")
            }

            pom.project {
                name 'test'//项目名称
                version '1.1.0'//版本号
                artifactId 'test'//最后下载的aar包名称就是这个
                groupId 'com.app.test'// 建议使用包命
                packaging 'aar'//打包类型
                description ''// 描述信息
            }
        }
    }
}
```
