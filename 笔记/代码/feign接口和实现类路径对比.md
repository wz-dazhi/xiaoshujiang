---
title: feign接口和实现类路径对比
tags: feign
category: /java/feign/2022-01
renderNumberedHeading: true
grammar_cjkRuby: true
---
2022-01-14 13:46

```java
public class CompareTest {
    private int count = 0;

    @Test
    public void test() throws Exception {
        scanPackage("com.xxx.xxx");
        System.out.println("一共" + count + "类.");
    }

    private void scanPackage(String basePackage) throws Exception {
        // 解析包
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX + ClassUtils.convertClassNameToResourcePath(basePackage) + "/**/*.class";
        // 获取资源数组
        Resource[] resources = new PathMatchingResourcePatternResolver().getResources(packageSearchPath);
        for (Resource resource : resources) {
            try (InputStream is = resource.getInputStream()) {
                ClassReader reader = new ClassReader(is);
                if (reader.getClassName().endsWith("ServiceImpl")) {
                    // System.out.println(reader.getClassName());
                    doCompare(Class.forName(reader.getClassName().replace('/', '.')));
                }
            }
        }
    }

    private void doCompare(Class<?> serviceImplClass) {
        Class<?> serviceInterface = findCandidateInterface(serviceImplClass);
        if (serviceInterface == null) {
            return;
        }

        boolean isError = false;
        FeignClient feignClient = serviceInterface.getAnnotation(FeignClient.class);
        HsafRestPath implHsafRestPath = serviceImplClass.getAnnotation(HsafRestPath.class);
        if (null == implHsafRestPath) {
            System.err.println("实现类: " + serviceImplClass.getName() + ", HsafRestPath: null");
            System.err.println("接口: " + serviceInterface.getName() + ", FeignClient: " + feignClient);
            isError = true;
        } else {
            try {
                String implPath = implHsafRestPath.value().startsWith("tpc") ? classPath(implHsafRestPath.value()) : "tpc/" + classPath(implHsafRestPath.value());
                String interfacePath = "".equals(feignClient.path()) ? "" : classPath(feignClient.path());

                Method[] implMethods = serviceImplClass.getMethods();
                for (Method m : implMethods) {
                    HsafRestPath implMethodHsafRestPath = m.getAnnotation(HsafRestPath.class);

                    Method interfaceMethod;
                    try {
                        interfaceMethod = serviceInterface.getMethod(m.getName(), m.getParameterTypes());
                    } catch (NoSuchMethodException | SecurityException ignored) {
                        continue;
                    }
                    RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(interfaceMethod, RequestMapping.class);
                    if (implMethodHsafRestPath == null || requestMapping == null) {
                        System.err.println("实现类: " + serviceImplClass.getName() + ", 方法: " + m.getName() + ", HsafRestPath: " + implMethodHsafRestPath);
                        System.err.println("接口: " + serviceInterface.getName() + ", 方法: " + interfaceMethod.getName() + ", RequestMapping: " + requestMapping);
                        isError = true;
                        continue;
                    }
                    String implMethodPath = implMethodHsafRestPath.value();
                    String implMethodFullPath = methodPath(implPath + methodPath(implMethodPath)).replace("//", "/");

                    String interfaceMethodPath = requestMapping.value()[0];
                    String interfaceMethodFullPath = methodPath(interfacePath + methodPath(interfaceMethodPath)).replace("//", "/");

                    if (!implMethodFullPath.equals(interfaceMethodFullPath)) {
                        System.err.println("实现类: " + serviceImplClass.getName() + ", 方法: " + m.getName() + ", 全路径: " + implMethodFullPath);
                        System.err.println("接口: " + serviceInterface.getName() + ", 方法: " + interfaceMethod.getName() + ", 全路径: " + interfaceMethodFullPath);
                        isError = true;
                    }
                }
            } catch (Exception e) {
                System.err.println("实现类: " + serviceImplClass.getName() + ", path: " + implHsafRestPath.value());
                System.err.println("接口: " + serviceInterface.getName() + ", path: " + feignClient.path());
                throw e;
            }
        }
        if (isError) {
            count++;
            System.out.println();
        }
    }

    private Class<?> findCandidateInterface(Class<?> serviceImplClass) {
        if (serviceImplClass.getInterfaces().length == 0) {
            return null;
        } else if (serviceImplClass.getInterfaces().length == 1) {
            return serviceImplClass.getInterfaces()[0].isAnnotationPresent(FeignClient.class) ? serviceImplClass.getInterfaces()[0] : null;
        } else {
            for (Class<?> anInterface : serviceImplClass.getInterfaces()) {
                if (anInterface.isAnnotationPresent(FeignClient.class)) {
                    return anInterface;
                }
            }
            return null;
        }
    }

    private static String classPath(String value) {
        if ("".equals(value) || null == value) {
            return value;
        }
        if (value.toCharArray().length == 1) {
            return value.toCharArray()[0] == '/' ? value.substring(1) : value;
        } else {
            value = value.toCharArray()[0] == '/' ? value.substring(1) : value;
            return value.toCharArray()[value.length() - 1] == '/' ? value.substring(0, value.length() - 1) : value;
        }
    }

    private static String methodPath(String value) {
        if ("".equals(value) || null == value) {
            return value;
        }
        return value.toCharArray()[0] == '/' ? value : "/" + value;
    }
}

```