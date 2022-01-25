---
title: java Agentmain
tags: java,热部署
category: /java/热部署/2022-01
renderNumberedHeading: true
grammar_cjkRuby: true
---
2022-01-25 13:57

> 上一篇讲解了premain的使用, premain只会在JVM启动的时候main方法执行之前会进行替换. 且还需要在目标类启动时候加入参数java -javaagent. 查看premain使用请看之前的文章
> 假如我们想要真正的实现不停机热部署的话, 并且不想加入启动参数, 需要agentmain
> 下面我们聊聊agentmain是如何使用的


# 继续沿用java-agent项目(懒)
* pom.xml不动, 添加两个Agentmain类和AgentTransformer类转换器
* Agentmain.java
```java
public class Agentmain {

    public static void agentmain(String agentArgs, Instrumentation inst) throws ClassNotFoundException, UnmodifiableClassException {
        // 第二个参数设置为true, 表示这个转换器可以重新转换
        inst.addTransformer(new AgentTransformer(), true);
        System.out.println("agentmain ok!");
        // 重新转换的class, 这里可以传数组
        inst.retransformClasses(Class.forName("com.wz.agent.Service"));
    }
}
```
* AgentTransformer.java 这里依然转换Service类
```java
public class AgentTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className,
                            Class<?> classBeingRedefined, ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {
        System.out.println("agentmain load Class  :" + className);
        if (className.equals("com/wz/agent/Service")) {
            try {
                // 我这里写死, 我的测试项目是jvm-test, idea 每次build后的target目录
                String classFile = "/Users/wangzhi/work/project/test/jvm-test/target/classes/com/wz/agent/Service.class";
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
                return os.toByteArray();
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
        return classfileBuffer;
    }

}
```
* 编写resources/META-INF/MANIFEST.MF
```shell
Agent-Class: com.wz.agent.Agentmain
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```
* 打包备用`mvn clean package`

# 测试项目
* Service.java, 跟之前一样, 不改动
```java
public class Service {
    public void print() {
        System.out.println("---原始------------Service print().========");
    }
}
```
* Test.java, 测试类加入死循环调用(模拟热部署)
```java
public class Test {

    public static void main(String[] args) throws InterruptedException {
        new Service().print();
        for (; ; ) {
            new Service().print();
            Thread.sleep(5000);
        }
    }
}
```
* attach api使用, 这里使用attach需要探测Test进程的pid, 得到Test pid进行热替换
* 为了演示, 我这里死循环, 10秒钟去探测一次, 并热替换
```java
public class AttachTest {
    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException, InterruptedException {
        while (true) {
            // 这里63099 是Test运行后的pid, 使用jps命令可以找到
            VirtualMachine virtualMachine = VirtualMachine.attach("63099");
            // 加载上面打包后的agent.jar
            virtualMachine.loadAgent("/Users/wangzhi/work/project/test/java-agent/target/agent.jar");
            System.out.println("ok");
            virtualMachine.detach();
            Thread.sleep(10000);
        }
    }
}
```

# 测试
* 运行Test.java
![run Test](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643095907000.png)
* 使用jps得到Test的pid, 修改AttachTest.java
* 运行AttachTest.java
![run 运行AttachTest](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643095993559.png)
* 手动修改Service.java, 点击idea Rebuild 重新构建class文件(**==Test在运行的过程中, idea不会自动编译, 对于普通的java项目. springboot的项目可以设置Update classes and resources #F44336==**)
![rebuild](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643096064709.png)
* 可以看到在不停止Test的情况下, 成功替换了class
![run result](https://gitee.com/wz-dazhi/pic/raw/master/xiaoshujiang/2022/1/25/1643096108696.png)
