

```bash

# byte-buddy

```


```java
package com.roy.apm;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

/**
 *
 * @author luoyh(Roy) - May 6, 2025
 * @since 21
 */
public class ApmTestTransformer  implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, 
            String className, 
            Class<?> classBeingRedefined,
            ProtectionDomain protectionDomain, 
            byte[] classfileBuffer) throws IllegalClassFormatException {
        if (!"com/tqbb/test/Apm".equals(className)) {
            return null;
        }   
        System.out.println("className: " + className);
        
        try {
            System.out.println("> log 0 : " + ClassPool.class);
            ClassPool pool = ClassPool.getDefault();
            System.out.println("> log 1");
            CtClass ctClass = pool.get(className.replace("/", "."));
            System.out.println("> log 2");

            CtMethod executeQuery = ctClass.getDeclaredMethod("test");
            System.out.println("> log 3");
//            executeQuery.setBody("""
//                    {
//                    long startTime = System.currentTimeMillis();
//                    Object $_ = executeQuery($$);
//                    long cost = System.currentTimeMillis() - startTime;
//                    System.out.println("jdbc query cost: " + cost + "ms");
//                    return $_;
//                    }
//                    """);
            executeQuery.addLocalVariable("startTime", CtClass.longType);
            executeQuery.insertBefore("""
                    startTime = System.currentTimeMillis();
                    """);
            System.out.println("> log 4");
            executeQuery.insertAfter(
                    """
                    long cost = System.currentTimeMillis() - startTime;
                    System.out.println("jdbc query cost: " + cost + "ms");
                    return $_;
                    """
//                    ,true
            );
            System.out.println("> log 5");
            return ctClass.toBytecode();
        } catch (Throwable ex) {
            ex.printStackTrace();
            System.out.println("error:" + ex.getMessage());
        }
        System.out.println("> log 6");
        return null;
    }
}
```

```java
package com.roy.apm;

import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

import net.bytebuddy.agent.builder.AgentBuilder;
import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.description.type.TypeDescription;
import net.bytebuddy.dynamic.DynamicType.Builder;
import net.bytebuddy.implementation.MethodDelegation;
import net.bytebuddy.matcher.ElementMatchers;
import net.bytebuddy.utility.JavaModule;

/**
 *
 * @author luoyh(Roy) - May 6, 2025
 * @since 21
 */
public class App {
    public static void premain(String args, Instrumentation inst) {
        inst.addTransformer(new DbTransformer());    // 拦截 DB
        inst.addTransformer(new ApmTestTransformer());
        
//        new AgentBuilder.Default()
//        .type(ElementMatchers.nameEndsWith("Service"))
//        .transform((builder, type, loader, module, protectionDomain) -> 
//            builder.method(ElementMatchers.any())
//                .intercept(MethodDelegation.to(TimingInterceptor.class))
//        ).installOn(inst);
        
        new AgentBuilder.Default().type(ElementMatchers.nameStartsWith("com.roy.test"))
          .transform((builder, type, loader, module, protectionDomain) -> 
          builder.method(ElementMatchers.any())
              .intercept(MethodDelegation.to(TimingInterceptor.class))
          ).installOn(inst);
        
        new AgentBuilder.Default()
            .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
            .type(ElementMatchers
                    .hasSuperType(ElementMatchers.named("java.sql.Statement"))
//                    .isSubTypeOf(ElementMatchers.named(""))
//                    .isSubTypeOf (Statement.class)
                                .and(ElementMatchers.not(ElementMatchers.isInterface()))
                                .and(ElementMatchers.not(ElementMatchers.<TypeDescription>isAbstract()))
                                .and(ElementMatchers.not(ElementMatchers.<TypeDescription>nameStartsWith("com.sun"))))
            .transform(new AgentBuilder.Transformer() {
                
                @Override
                public Builder<?> transform(Builder<?> builder, 
                        TypeDescription typeDescription, 
                        ClassLoader classLoader,
                        JavaModule module, 
                        ProtectionDomain protectionDomain) {
                    String className = typeDescription.getName();
                    System.out.println("class-name=%s, plugin-name=%s".formatted(className, "jdbc stmt"));
                    builder = 
                            builder
//                            .visit(Advice.to(String.class)
                            .method(
                                    ElementMatchers.named("executeQuery")
//                                    ElementMatchers.isMethod()
//                                    .and(ElementMatchers.named("executeQuery"))
                                    .and(ElementMatchers.<MethodDescription>isPublic())
                                    .and(ElementMatchers.not(ElementMatchers.<MethodDescription>named("executeInternal"))))
                            .intercept(MethodDelegation.to(TimingInterceptor.class))
                             ;
//                    FieldDefine[] fields = plugin.buildFieldDefine();
//                    if (fields != null && fields.length > 0) {
//                        for (int x = 0; x < fields.length; x++) {
//                            builder = builder.defineField(fields[x].name, fields[x].type, fields[x].modifiers);
//                        }
//                    }
                    return builder;
                }
            }).installOn(inst)
            
        ;
        
        
//        
//        new AgentBuilder.Default().type(ElementMatchers.nameEndsWith("Service"))
//        .transform(new AgentBuilder.Transformer() {
//            
//            @Override
//            public Builder<?> transform(Builder<?> builder, TypeDescription typeDescription, ClassLoader classLoader,
//                    JavaModule module, ProtectionDomain protectionDomain) {
//                // TODO Auto-generated method stub
//                return null;
//            }
//        })
//        ;
    }
}

```

```java
package com.roy.apm;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;

import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtMethod;

/**
 *
 * @author luoyh(Roy) - May 6, 2025
 * @since 21
 */
public class DbTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, 
            String className, 
            Class<?> classBeingRedefined,
            ProtectionDomain protectionDomain, 
            byte[] classfileBuffer) throws IllegalClassFormatException {
        if (!"com/zaxxer/hikari/pool/HikariProxyStatement".equals(className)) {
            return null;
        }   
        System.out.println("className: " + className);
        
        try {
            System.out.println("> log 0 : " + ClassPool.class);
            ClassPool pool = ClassPool.getDefault();
            System.out.println("> log 1");
            CtClass ctClass = pool.get(className.replace("/", "."));
            System.out.println("> log 2");

            CtMethod executeQuery = ctClass.getDeclaredMethod("executeQuery");
            System.out.println("> log 3");
//            executeQuery.setBody("""
//                    {
//                    long startTime = System.currentTimeMillis();
//                    Object $_ = executeQuery($$);
//                    long cost = System.currentTimeMillis() - startTime;
//                    System.out.println("jdbc query cost: " + cost + "ms");
//                    return $_;
//                    }
//                    """);
            executeQuery.addLocalVariable("startTime", CtClass.longType);
            executeQuery.insertBefore("""
                    startTime = System.currentTimeMillis();
                    """);
            System.out.println("> log 4");
            executeQuery.insertAfter(
                    """
                    long cost = System.currentTimeMillis() - startTime;
                    System.out.println("jdbc query cost: " + cost + "ms");
                    """
//                    ,true
            );
            System.out.println("> log 5");
            return ctClass.toBytecode();
        } catch (Throwable ex) {
            ex.printStackTrace();
            System.out.println("error:" + ex.getMessage());
        }
        System.out.println("> log 6");
        return null;
    }
}
```

```java
package com.roy.apm;

import java.util.concurrent.Callable;

import net.bytebuddy.implementation.bind.annotation.RuntimeType;
import net.bytebuddy.implementation.bind.annotation.SuperCall;

public class TimingInterceptor {
    @RuntimeType
    public static Object intercept(@SuperCall Callable<?> callable) throws Exception {
        long start = System.currentTimeMillis();
        try {
            return callable.call();
        } finally {
            System.out.println("方法耗时: " + (System.currentTimeMillis() - start) + "ms");
        }
    }
}

```

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.roy.apm</groupId>
    <artifactId>test-apm</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>test-apm</name>
    <url>http://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>net.bytebuddy</groupId>
            <artifactId>byte-buddy</artifactId>
            <version>1.17.5</version>
        </dependency>
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.30.2-GA</version>
        </dependency>
    </dependencies>



    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>package</phase>
                    </execution>
                </executions>
                <configuration>
                    <archive>
                        <manifestEntries>
                            <Agent-Class>com.roy.apm.App</Agent-Class>
                            <Premain-Class>com.roy.apm.App</Premain-Class>
                            <Can-Redefine-Classes>true</Can-Redefine-Classes>
                            <Can-Retransform-Classes>true</Can-Retransform-Classes>
                            <Pinpoint-Version>${project.version}</Pinpoint-Version>
                            <Boot-Class-Path>${project.build.finalName}.jar</Boot-Class-Path>
                        </manifestEntries>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>

```

```java
package com.roy.test;

import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;

import com.tqbb.test.util.DataSources;
import com.zaxxer.hikari.HikariDataSource;

import lombok.extern.slf4j.Slf4j;

/**
 *
 * @author luoyh(Roy) - May 6, 2025
 * @since 21
 */
@Slf4j
public class Apm {

    public static void main(String[] args) throws Exception {
        try (HikariDataSource ds = DataSources.newMysqlTEST();
                Connection conn = ds.getConnection();
                Statement stmt = conn.createStatement();
                ResultSet rs = stmt.executeQuery("select id,duty_date from ledger_duty")) {
            System.out.println(stmt.getClass());
            while (rs.next()) {
                log.info("id:" + rs.getLong(1) + ",duty_date:" + rs.getDate(2));
                System.out.println("id:" + rs.getLong(1) + ",duty_date:" + rs.getDate(2));
            }
        };
        System.out.println("test:" + test());
    }
    
    
    public static String test() throws InterruptedException {
        log.info("line number 36");
        Thread.sleep(2000);
        log.info("line number 38");
        return "1234";
    }
}

```