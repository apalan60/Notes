# Gradle

## [Use IntelliJ IDEA to create JAR with dependency using Gradle](https://youtu.be/tKdFrVzDLV4?si=xqRgpJ8dC7XDLpli)

### 為什麼需要JAR?
  javac compile .java file 之後，即可透過JVM等Runtime來執行，但當該java file有外部依賴，就會需要把每份依賴的檔案都compile之後才能執行，JAR就是用於此刻打包這些依賴項目，降低複雜度。 類似於.NET的.DLL

### [如何在intelliJ 建立專案時，使用Gradle建置](https://www.jetbrains.com/help/idea/work-with-gradle-projects.html)
    
### 如何透過gradle打包成.jar

  ```bash
  ./gradlew build jar
  ```

  執行後，```projectName/build/libs```會多出一個.jar file
  
  ```bash
  # sample
  GradleBuildJarPractice/build/libs on  master [!] …
  ➜ ls
  GradleBuildJarPratice-1.0-SNAPSHOT.jar
  ```

- 如何執行.jar

  ```bash
  java -jar GradleBuildJarPratice-1.0-SNAPSHOT.jar
  ```

  但此時在沒有調整build.gradle的情況下，容易遇到以下情況

  ```
  no main manifest attribute, in GradleBuildJarPratice-1.0-SNAPSHOT.jar
  ```

  有兩個方式可以解決，原理都是要讓JVM知道要執行的個java class

  1. 調整build.gradle

  ```groovy
  jar{
    manifest {
        attributes(
                'Main-Class': 'com.Init' 
        )
    }
  } 
  ```

  2. commmand明確指定class

  ```bash
  java -cp GradleBuildJarPratice-1.0-SNAPSHOT.jar com.Init
  ```

### 如何新增依賴項
  到build.gradle --> 右鍵 generate --> dependency --> add maven dependency 
  ![image](https://github.com/user-attachments/assets/369af9ac-e501-4f8c-8cf3-8200bfdd6a81)


### 如何把依賴項打包進jar (又稱Uber JAR / fat JAR)

  新增dependency之後直接run jar file， 會發現依賴項找不到
  
  ```bash
GradleBuildJarPractice/build/libs on  master [!] …
➜ java -cp Gra* com.Init
Exception in thread "main" java.lang.NoClassDefFoundError: org/json/JSONObject
        at com.Init.main(Init.java:7)
Caused by: java.lang.ClassNotFoundException: org.json.JSONObject
        at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641)
        at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:188)
        at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:526)
        ... 1 more
```

執行 ```jar --tvf```會發現依賴項並沒有被打包進去

```bash
GradleBuildJarPractice/build/libs on  master [!] via ☕ v21.0.6 …
➜ jar -tvf GradleBuildJarPratice-1.0-SNAPSHOT.jar
     0 Wed Mar 05 00:12:56 CST 2025 META-INF/
    25 Tue Mar 04 23:13:06 CST 2025 META-INF/MANIFEST.MF
     0 Wed Mar 05 00:12:56 CST 2025 com/
   668 Wed Mar 05 00:12:56 CST 2025 com/Init.class
```

> 除了手動在build.gradle明確指定要打包哪些依賴項，目前有個開源專案[Gradle Shadow](https://github.com/GradleUp/shadow)能協助完成這件事

在build.gradle新增該項目plugin
```
plugins {
    id 'java'
    id 'com.github.johnrengelman.shadow' version '8.1.1'  
}
```

接著執行shell 
```bash
./gradlew shadowJar
```

```build/lib```會多一個jar file

e.g.
```
java -jar build/libs/GradleBuildJarPractice-1.0-SNAPSHOT-all.jar
```

試著比較兩份jar，uber jar 明顯大許多，因為裡面包含了依賴項的所有java file
```
GradleBuildJarPractice/build/libs on  master [!] via ☕ v21.0.6 …
➜ ls -l
total 80
-rwxrwxrwx 1 apalan60 apalan60 77312 Mar  5 00:33 GradleBuildJarPratice-1.0-SNAPSHOT-all.jar
-rwxrwxrwx 1 apalan60 apalan60   872 Mar  5 00:12 GradleBuildJarPratice-1.0-SNAPSHOT.jar
```

接著執行 ```jave -jar <your-jar-file>```就能正常執行了

```
GradleBuildJarPractice/build/libs on  master [!] …
➜ java -jar GradleBuildJarPratice-1.0-SNAPSHOT-all.jar
{"car":null,"name":"John","age":30}
```
