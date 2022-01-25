---
title: java Premain测试
tags: java,热部署
category: /java/热部署/2022-01
renderNumberedHeading: true
grammar_cjkRuby: true
---
2022-01-25 13:55

* [代码测试](#代码测试)
	* [创建premain项目java-agent](#创建premain项目java-agent)
		* [pom.xml](#pomxml)
		* [代码](#代码)
		* [编写resources/META-INF/MANIFEST.MF](#编写resourcesmeta-infmanifestmf)
		* [打包java-agent](#打包java-agent)
	* [创建测试项目](#创建测试项目)
		* [创建普通maven项目](#创建普通maven项目)
	* [测试](#测试)

# 代码测试
## 创建premain项目java-agent
### pom.xml 
* 需要配置一下MANIFEST.MF, 避免maven打包的时候覆盖我们自己创建的MANIFEST.MF (当我们自己手动在resources下创建MANIFEST.MF的时候), 配置好之后maven打包的时候会帮我们自动合并到一个MANIFEST.MF中
```xml
<build>
    <finalName>agent</finalName>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <archive>
                    <manifestFile>src/main/resources/META-INF/MANIFEST.MF</manifestFile>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```
### 代码
* 创建一个Transformer类转换器, 继承ClassFileTransformer
```java
public class Transformer implements ClassFileTransformer {
    @Override
    public byte[] transform(ClassLoader loader, String className,
                            Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {
        System.out.println(className);
        // 只转换我们自己手动写的class, Service类在测试项目中
        if (className.equals("com/wz/agent/Service")) {
            try {
                // 方便演示手动编译好一个class文件, 然后把这个class放到一个跟项目没关系的目录中(Service的包在com.wz.agent.Service).用来premain的测试
                String classFile = "/Users/wangzhi/work/project/test/Service.class";
                System.out.println(classFile);
                FileInputStream fis = new FileInputStream(classFile);
                ByteArrayOutputStream os = new ByteArrayOutputStream();
                byte[] bytes = new byte[1024];
                int len;
                while ((len = fis.read(bytes)) != -1) {
                    os.write(bytes, 0, len);
                }
                fis.close();
                os.flush();
                os.close();
                System.out.println("-------写入完成------------");
                // 将新的class转成子节数组返回
                return os.toByteArray();
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
        // 返回原始的class子节数组
        return classfileBuffer;
    }
}
```
* 创建一个类, 名叫Premain(叫什么都可以)
* Instrumentation 加入我们自己写的转换器类
```java
public class Premain {
    public static void premain(String agentArgs, Instrumentation inst) throws ClassNotFoundException, UnmodifiableClassException {
        inst.addTransformer(new Transformer());
        System.out.println("premain ok!");
    }
}
```

### 编写resources/META-INF/MANIFEST.MF
* 在resources目录下新建META-INF文件夹
* 在META-INF文件夹下新建MANIFEST.MF
* 写入我们上面写的Premain-Class
```mf
Manifest-Version: 1.0
Premain-Class: com.wz.agent.Premain
```
### 打包java-agent
* 打包之后备用, 在target目录下会产生agent.jar(上面我们定义了finalName)
```shell
mvn -U clean package
```

## 创建测试项目
### 创建普通maven项目
* 创建一个Service类用来测试(com.wz.agent.Service)
```java
public class Service {
    public void print() {
        System.out.println("---原始------------Service print().========");
    }
}
```
* 创建一个Test main方法
```java
public class Test {

    public static void main(String[] args) throws InterruptedException {
        new Service().print();
    }
}
```
## 测试
1. 普通测试, 不使用premain. 直接点击idea -> run
![结果](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643091942551.png)
2. Premain测试, 需要配置idea的run
* 重写Service类, 我这里只是打印输出换了一下, 编译完成后, 将class文件放到一个跟项目没关系的目录中 **需要跟agent项目替换的那个class路径保持一致**
![agent class](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643092497430.png)
* 然后再把Service.java的输出还原回来(避免混乱), 我这里输出的还是"原始"
![origin Service](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643092659011.png)
* 打开Edit Configurations...
![Configurations](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643092046645.png)
* 配置我们上面写的agent.jar, `-javaagent:/Users/wangzhi/work/project/test/java-agent/target/agent.jar`, 点击保存
![agent](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643092187928.png)
* 运行Test, 可以看到已经替换成了  ---agent------------Service print().========
![agent run](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643092773465.png)
