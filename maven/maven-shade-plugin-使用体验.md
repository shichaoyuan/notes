## 0.

使用 maven 管理项目一开始方便，但是随着依赖越来越多就会越来越麻烦。

最头痛的问题就是依赖冲突，做为最终使用方可以通过 <exclusions> 解决，但是作为 sdk 的提供方就更麻烦了，搞不好就会被使用方 diss。

最近发现了一个神奇的 maven 插件 —— maven-shade-plugin —— 非常适合解决这类问题。

## 1. maven-shade-plugin 使用

直接上一个示例 pom：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>scyuan.maven</groupId>
    <artifactId>shade-example</artifactId>
    <version>1.0-SNAPSHOT</version>

    <name>shade-example</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.8.0</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.1.1</version>
                <configuration>
                    <artifactSet>
                        <includes>
                            <include>redis.clients:jedis</include>
                            <include>org.apache.commons:commons-pool2</include>
                        </includes>
                    </artifactSet>
                    <filters>
                        <filter>
                            <artifact>*:*</artifact>
                            <excludes>
                                <exclude>META-INF/*.SF</exclude>
                                <exclude>META-INF/*.DSA</exclude>
                                <exclude>META-INF/*.RSA</exclude>
                            </excludes>
                        </filter>
                    </filters>
                    <relocations>
                        <relocation>
                            <pattern>redis</pattern>
                            <shadedPattern>scyuan.maven.shaded.redis</shadedPattern>
                        </relocation>
                        <relocation>
                            <pattern>org.apache.commons</pattern>
                            <shadedPattern>scyuan.maven.shaded.org.apache.commons</shadedPattern>
                        </relocation>
                    </relocations>
                </configuration>
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

在这个示例中将[臭名昭著](https://stackoverflow.com/questions/22704518/jedispoolconfig-is-not-assignable-to-genericobjectpoolconfig)的 jedis 打进了 jar 包中，并且将包名前缀改为 `scyuan.maven.shaded.redis`，如此 sdk 的使用者即便显式依赖了 2.2.0 版本的 jedis 也不会对 sdk 造成毁灭性影响了。

我们可以简单写个主类，打个包试一下。

```java
package scyuan.maven;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

public class App {
    public static void main(String[] args) {
        String key = "hello";
        String value = "redis";

        JedisPoolConfig config = new JedisPoolConfig();
        JedisPool jedisPool = new JedisPool(config, "127.0.0.1", 6379);

        try (Jedis jedis = jedisPool.getResource()) {
            jedis.set(key, value);
            System.out.println(jedis.get(key));
        } catch (Exception e) {
            System.out.println(e.toString());
        }

        jedisPool.close();
    }
}
```

将打好的 `shade-example-1.0-SNAPSHOT.jar` 包解压缩看一下：

```
➜  target tree -L 5 shade-example-1.0-SNAPSHOT
shade-example-1.0-SNAPSHOT
├── META-INF
│   ├── LICENSE.txt
│   ├── MANIFEST.MF
│   ├── NOTICE.txt
│   └── maven
│       ├── org.apache.commons
│       │   └── commons-pool2
│       │       ├── pom.properties
│       │       └── pom.xml
│       ├── redis.clients
│       │   └── jedis
│       │       ├── pom.properties
│       │       └── pom.xml
│       └── scyuan.maven
│           └── shade-example
│               ├── pom.properties
│               └── pom.xml
└── scyuan
    └── maven
        ├── App.class
        └── shaded
            ├── org
            │   └── apache
            └── redis
                └── clients

15 directories, 10 files

```

`jedis` 和 `commons-pool2` 按照我们的意愿出现在了对应的位置。

我们执行一下看看：

```
➜  target java -cp shade-example-1.0-SNAPSHOT.jar:jedis-2.2.0.jar:commons-pool-1.6.jar scyuan.maven.App
scyuan.maven.shaded.redis
```

*除了输出与预期不符之外，其他都正常。。。*

## 2. maven-shade-plugin 对字符串的处理

看起来好像是将代码 `String value = "redis";` 中的字符串 `redis` 也替换为了 `scyuan.maven.shaded.redis`。

怎么办？看来需要把 relocation 的 pattern 限定的更精确一些，比如 `<pattern>redis.clients</pattern>`。

再试一下，问题确实解决了。但是如果代码变为 `String value = "redis.clients";`，那么还是会重复之前的问题，所以这是一个注意事项：**使用 maven-shade-plugin 时需要小心处理字符串**。

我们需要看一下 [maven-shade-plugin 的源码](https://github.com/apache/maven-shade-plugin)。

总的来说就是使用[asm](https://asm.ow2.io/)，我们找到相关的 `RelocatorRemapper`：

```
// org/apache/maven/plugins/shade/DefaultShader.java#L564

    static class RelocatorRemapper
        extends Remapper
    {

        private final Pattern classPattern = Pattern.compile( "(\\[*)?L(.+);" );

        List<Relocator> relocators;

        RelocatorRemapper( List<Relocator> relocators )
        {
            this.relocators = relocators;
        }

        public boolean hasRelocators()
        {
            return !relocators.isEmpty();
        }

        public Object mapValue( Object object )
        {
            if ( object instanceof String )
            {
                String name = (String) object;
                String value = name;

                String prefix = "";
                String suffix = "";

                Matcher m = classPattern.matcher( name );
                if ( m.matches() )
                {
                    prefix = m.group( 1 ) + "L";
                    suffix = ";";
                    name = m.group( 2 );
                }

                for ( Relocator r : relocators )
                {
                    if ( r.canRelocateClass( name ) )
                    {
                        value = prefix + r.relocateClass( name ) + suffix;
                        break;
                    }
                    else if ( r.canRelocatePath( name ) )
                    {
                        value = prefix + r.relocatePath( name ) + suffix;
                        break;
                    }
                }

                return value;
            }

            return super.mapValue( object );
        }

        public String map( String name )
        {
            String value = name;

            String prefix = "";
            String suffix = "";

            Matcher m = classPattern.matcher( name );
            if ( m.matches() )
            {
                prefix = m.group( 1 ) + "L";
                suffix = ";";
                name = m.group( 2 );
            }

            for ( Relocator r : relocators )
            {
                if ( r.canRelocatePath( name ) )
                {
                    value = prefix + r.relocatePath( name ) + suffix;
                    break;
                }
            }

            return value;
        }

    }
```

注意其中的 `mapValue` 方法对 `String` 进行了特殊处理，这些处理也是很有必要的，比如说：硬编码的类名。


## 3. 其他资料

1. [Apache Maven Shade Plugin](https://maven.apache.org/plugins/maven-shade-plugin/index.html)
