# GraalVM native image メモ

## native-image 対応モジュールの構造

例えば、helidon-webserver-1.1.2.jar の中身

META-INF/native-image/native-image.properties

```properties
Args=--report-unsupported-elements-at-runtime \
     --allow-incomplete-classpath \
     -H:ReflectionConfigurationResources=${.}/helidon-webserver-reflection-config.json \
     --no-fallback \
     --initialize-at-run-time=io.netty.buffer.UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf,io.netty.buffer.UnreleasableByteBuf,io.netty.handler.codec.http2.Http2ConnectionHandler,io.netty.handler.codec.http2.Http2CodecUtil,io.netty.handler.codec.http.HttpObjectEncoder,io.netty.handler.codec.http2.CleartextHttp2ServerUpgradeHandler,io.netty.handler.codec.http2.DefaultHttp2FrameWriter,io.netty.handler.codec.http2.Http2ServerUpgradeCodec,io.netty.handler.ssl.JdkNpnApplicationProtocolNegotiator,io.netty.handler.ssl.JettyNpnSslEngine,io.netty.handler.ssl.ReferenceCountedOpenSslEngine,io.netty.handler.ssl.ConscryptAlpnSslEngine
```

META-INF/native-image/helidon-webserver-reflection-config.json

```json
[
    {
        "name": "io.netty.channel.socket.nio.NioServerSocketChannel",
        "allPublicConstructors": true,
        "allPublicMethods": true
    }
]
```

## --initialize-at-run-time / --initialize-at-build-time について

https://medium.com/graalvm/understanding-class-initialization-in-graalvm-native-image-generation-d765b7e4d6ed

Executing class initializers during image generation is in most cases a good idea. But there are a few use cases where class initializers do things that must not run during image generation, for example

+ start application threads that continue to run in the background of the application,
+ load native libraries using java.lang.Runtime.load(String),
+ open files or sockets, or
+ allocate C memory, e.g., create direct byte buffers using java.nio.ByteBuffer.allocateDirect(int).

→ 記事が古いのでオプションが 19.1.0 では違う  

```
$ cat HelloStartupTime.java
import java.util.Date;

class HelloStartupTime {
  public static void main(String args[]) {
    System.out.println("Startup: " + Startup.TIME);
    System.out.println("Now:     " + new Date());
  }
}

class Startup {
  static final Date TIME = new Date();
}

$ $JAVA_HOME/bin/javac HelloStartupTime.java

$ $JAVA_HOME/bin/java HelloStartupTime
Startup: Thu Jul 11 01:56:34 GMT 2019
Now:     Thu Jul 11 01:56:34 GMT 2019
```

native-imageを作って実行してみる

```
$ $GRAALVM_HOME/bin/native-image HelloStartupTime
Build on Server(pid: 27502, port: 45623)*
(...)

$ ./hellostartuptime  
Startup: Thu Jul 11 01:58:37 GMT 2019
Now:     Thu Jul 11 01:58:37 GMT 2019
```

デフォルトでは --initialize-at-run-time が効いていそう
--initialize-at-build-time を指定してみる

```
$ ls
HelloStartupTime.class  HelloStartupTime.java  Startup.class

$ $GRAALVM_HOME/bin/native-image --initialize-at-build-time=Startup HelloStartupTime
Build on Server(pid: 27502, port: 45623)
(...)

$ ./hellostartuptime  
Startup: Thu Jul 11 02:05:43 GMT 2019
Now:     Thu Jul 11 02:06:10 GMT 2019
```

ビルド時にクラスの初期化がされた

## helidon-cowweb のポイント

### META-INF/native-image/native-image.properties

普通に nataive-image を実行すると、以下のWARNINGが出るが、これは io.helidon.webserver.ByteBufRequestChunk$OneTimeLoggerHolder クラスの初期化処理がビルド時に実行されているため。内容からしてこの初期化はランタイムの方が良いのではないかと考えられるので、オプションを追加した。

```
[WARNING] Jul 11, 2019 1:35:45 AM io.helidon.webserver.ByteBufRequestChunk$OneTimeLoggerHolder <clinit>
[WARNING] WARNING: LEAK: RequestChunk.release() was not called before it was garbage collected. While the Reactive WebServer is designed to automatically release all the RequestChunks, it still comes with a considerable performance penalty and a demand for a large memory space (depending on expected throughput, it might require even more than 2GB). As such the users are strongly advised to release all the RequestChunk instances explicitly when they're not needed.
```

あとは verboseを入れて実行の詳細ログを出す、ということで以下の通り。

```properties
Args= \
    -H:ReflectionConfigurationResources=${.}/helidon-example-reflection-config.json \
    --initialize-at-run-time=io.helidon.webserver.ByteBufRequestChunk$OneTimeLoggerHolder \
    --verbose
```


### pom.xml

cowsayに入っているリソース（ascii art, resource bundle）を native-image tool に処理させないといけないので、native-image プロファイルに記述を追加

リソースディレクトリの追加

```xml
    <resources>
        <resource>
            <directory>src/main/resources</directory>
        </resource>
        <resource>
            <directory>${project.build.directory}/resources</directory>
        </resource>
    </resources>
```

プラグインの追加

```xml
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <executions>
            <!-- extract cowsay resources from dependency jar -->
            <execution>
                <id>unpack-cowsay</id>
                <phase>generate-sources</phase>
                <goals>
                    <goal>unpack</goal>
                </goals>
                <configuration>
                    <artifactItems>
                        <artifactItem>
                            <groupId>com.github.ricksbrown</groupId>
                            <artifactId>cowsay</artifactId>
                            <version>1.0.3</version>
                            <outputDirectory>${project.build.directory}/resources</outputDirectory>
                            <includes>**/*.cow, **/*.csv, **/*.properties</includes>
                            <excludes>META-INF/*.*</excludes>
                        </artifactItem>
                    </artifactItems>
                </configuration>
            </execution>
        </executions>
    </plugin>
```

実行結果

```
$ mvn package -Pnative-image
[INFO] Scanning for projects...
[INFO] 
[INFO] -----------------< io.helidon.examples:helidon-cowweb >-----------------
[INFO] Building helidon-cowweb 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-dependency-plugin:2.9:unpack (unpack-cowsay) @ helidon-cowweb ---
[INFO] Configured Artifact: com.github.ricksbrown:cowsay:1.0.3:jar
[INFO] cowsay-1.0.3.jar already unpacked.
[INFO] 
[INFO] --- maven-resources-plugin:3.0.2:resources (default-resources) @ helidon-cowweb ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 4 resources
[INFO] Copying 60 resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ helidon-cowweb ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 4 source files to /home/opc/work/helidon/helidon-cowweb/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.0.2:testResources (default-testResources) @ helidon-cowweb ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ helidon-cowweb ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- maven-surefire-plugin:2.19.1:test (default-test) @ helidon-cowweb ---

-------------------------------------------------------
 T E S T S
-------------------------------------------------------

Results :

Tests run: 0, Failures: 0, Errors: 0, Skipped: 0

[INFO] 
[INFO] --- maven-dependency-plugin:2.9:copy-dependencies (copy-dependencies) @ helidon-cowweb ---
[INFO] io.helidon.media.jsonp:helidon-media-jsonp-server:jar:1.1.2 already exists in destination.
[INFO] io.helidon.media.jsonp:helidon-media-jsonp-common:jar:1.1.2 already exists in destination.
[INFO] org.apache.maven:maven-model:jar:3.0 already exists in destination.
[INFO] io.helidon.metrics:helidon-metrics:jar:1.1.2 already exists in destination.
[INFO] org.apache.maven:maven-artifact:jar:3.0 already exists in destination.
[INFO] org.yaml:snakeyaml:jar:1.24 already exists in destination.
[INFO] org.osgi:org.osgi.annotation.versioning:jar:1.0.0 already exists in destination.
[INFO] io.netty:netty-transport:jar:4.1.34.Final already exists in destination.
[INFO] io.helidon.health:helidon-health:jar:1.1.2 already exists in destination.
[INFO] io.opentracing:opentracing-noop:jar:0.31.0 already exists in destination.
[INFO] org.codehaus.plexus:plexus-utils:jar:2.0.4 already exists in destination.
[INFO] io.projectreactor:reactor-core:jar:3.1.5.RELEASE already exists in destination.
[INFO] commons-cli:commons-cli:jar:1.3 already exists in destination.
[INFO] org.reactivestreams:reactive-streams:jar:1.0.2 already exists in destination.
[INFO] io.netty:netty-resolver:jar:4.1.34.Final already exists in destination.
[INFO] org.sonatype.sisu:sisu-guice:jar:noaop:2.1.7 already exists in destination.
[INFO] org.eclipse.microprofile.health:microprofile-health-api:jar:1.0 already exists in destination.
[INFO] io.netty:netty-codec:jar:4.1.34.Final already exists in destination.
[INFO] org.apache.commons:commons-lang3:jar:3.3.2 already exists in destination.
[INFO] org.eclipse.microprofile.metrics:microprofile-metrics-api:jar:1.1 already exists in destination.
[INFO] io.helidon.common:helidon-common-key-util:jar:1.1.2 already exists in destination.
[INFO] io.helidon.health:helidon-health-checks:jar:1.1.2 already exists in destination.
[INFO] org.codehaus.plexus:plexus-component-annotations:jar:1.5.4 already exists in destination.
[INFO] javax.annotation:javax.annotation-api:jar:1.3.1 already exists in destination.
[INFO] org.apache.maven:maven-plugin-api:jar:3.0 already exists in destination.
[INFO] org.sonatype.sisu:sisu-inject-bean:jar:1.4.2 already exists in destination.
[INFO] io.helidon.common:helidon-common-http:jar:1.1.2 already exists in destination.
[INFO] io.opentracing:opentracing-api:jar:0.31.0 already exists in destination.
[INFO] io.helidon.config:helidon-config:jar:1.1.2 already exists in destination.
[INFO] io.helidon.bundles:helidon-bundles-webserver:jar:1.1.2 already exists in destination.
[INFO] org.sonatype.sisu:sisu-inject-plexus:jar:1.4.2 already exists in destination.
[INFO] io.netty:netty-handler:jar:4.1.34.Final already exists in destination.
[INFO] io.helidon.common:helidon-common:jar:1.1.2 already exists in destination.
[INFO] io.helidon.config:helidon-config-yaml:jar:1.1.2 already exists in destination.
[INFO] org.codehaus.plexus:plexus-classworlds:jar:2.2.3 already exists in destination.
[INFO] io.netty:netty-codec-http:jar:4.1.34.Final already exists in destination.
[INFO] io.netty:netty-buffer:jar:4.1.34.Final already exists in destination.
[INFO] io.helidon.common:helidon-common-configurable:jar:1.1.2 already exists in destination.
[INFO] com.github.ricksbrown:cowsay:jar:1.0.3 already exists in destination.
[INFO] io.netty:netty-common:jar:4.1.34.Final already exists in destination.
[INFO] io.helidon.common:helidon-common-reactive:jar:1.1.2 already exists in destination.
[INFO] io.helidon.media:helidon-media-common:jar:1.1.2 already exists in destination.
[INFO] javax.json:javax.json-api:jar:1.1.2 already exists in destination.
[INFO] io.netty:netty-codec-http2:jar:4.1.34.Final already exists in destination.
[INFO] org.glassfish:javax.json:jar:1.1.2 already exists in destination.
[INFO] io.opentracing:opentracing-util:jar:0.31.0 already exists in destination.
[INFO] io.helidon.integrations.graal:helidon-graal-native-image-extension:jar:1.1.2 already exists in destination.
[INFO] io.helidon.common:helidon-common-context:jar:1.1.2 already exists in destination.
[INFO] io.helidon.webserver:helidon-webserver:jar:1.1.2 already exists in destination.
[INFO] 
[INFO] --- maven-jar-plugin:2.5:jar (default-jar) @ helidon-cowweb ---
[INFO] Building jar: /home/opc/work/helidon/helidon-cowweb/target/helidon-cowweb.jar
[INFO] 
[INFO] --- helidon-maven-plugin:1.0.10:native-image (default) @ helidon-cowweb ---
[INFO] Building native image :/home/opc/work/helidon/helidon-cowweb/target/helidon-cowweb
[INFO] Apply jar:file:///home/opc/.m2/repository/io/helidon/webserver/helidon-webserver/1.1.2/helidon-webserver-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/.m2/repository/io/helidon/media/jsonp/helidon-media-jsonp-common/1.1.2/helidon-media-jsonp-common-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/.m2/repository/io/helidon/config/helidon-config-yaml/1.1.2/helidon-config-yaml-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/.m2/repository/io/helidon/integrations/graal/helidon-graal-native-image-extension/1.1.2/helidon-graal-native-image-extension-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/work/helidon/helidon-cowweb/target/libs/helidon-webserver-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/work/helidon/helidon-cowweb/target/libs/helidon-media-jsonp-common-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/work/helidon/helidon-cowweb/target/libs/helidon-config-yaml-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/work/helidon/helidon-cowweb/target/libs/helidon-graal-native-image-extension-1.1.2.jar!/META-INF/native-image/native-image.properties
[INFO] Apply jar:file:///home/opc/work/helidon/helidon-cowweb/target/helidon-cowweb.jar!/META-INF/native-image/native-image.properties
[INFO] Executing [
[INFO] /home/opc/opt/graalvm-ee-19.1.0/jre/bin/java \
[INFO] -XX:+UnlockExperimentalVMOptions \
[INFO] -XX:+EnableJVMCI \
[INFO] -Dtruffle.TrustAllTruffleRuntimeProviders=true \
[INFO] -Dtruffle.TruffleRuntime=com.oracle.truffle.api.impl.DefaultTruffleRuntime \
[INFO] -Dgraalvm.ForcePolyglotInvalid=true \
[INFO] -Dgraalvm.locatorDisabled=true \
[INFO] -d64 \
[INFO] -XX:-UseJVMCIClassLoader \
[INFO] -XX:+UseJVMCINativeLibrary \
[INFO] -Xss10m \
[INFO] -Xms1g \
[INFO] -Xmx12384249440 \
[INFO] -Duser.country=US \
[INFO] -Duser.language=en \
[INFO] -Dorg.graalvm.version=19.1.0 \
[INFO] -Dorg.graalvm.config=EE \
[INFO] -Dcom.oracle.graalvm.isaot=true \
[INFO] -Djvmci.class.path.append=/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/enterprise-graal.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/graal.jar \
[INFO] -Xbootclasspath/a:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/boot/graal-sdk.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/boot/graaljs-scriptengine.jar \
[INFO] -cp \
[INFO] /home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm-enterprise-llvm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm-llvm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/graal-llvm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/llvm-wrapper.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/javacpp.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/llvm-platform-specific.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/objectfile.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/pointsto.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm-enterprise.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/enterprise-graal.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/graal.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/graal-management.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/jvmci-hotspot.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/jvmci-api.jar \
[INFO] com.oracle.svm.hosted.NativeImageGeneratorRunner \
[INFO] -watchpid \
[INFO] 28407 \
[INFO] -imagecp \
[INFO] /home/opc/opt/graalvm-ee-19.1.0/jre/lib/boot/graal-sdk.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/boot/graaljs-scriptengine.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm-enterprise-llvm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm-llvm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/graal-llvm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/llvm-wrapper.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/javacpp.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/llvm-platform-specific.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/objectfile.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/pointsto.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/builder/svm-enterprise.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/enterprise-graal.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/graal.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/graal-management.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/jvmci-hotspot.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/jvmci/jvmci-api.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/library-support.jar:/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/library-support-enterprise.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-bundles-webserver-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-webserver-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-common-reactive-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-common-http-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-common-context-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-media-common-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-common-key-util-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-common-configurable-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/opentracing-util-0.31.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/opentracing-api-0.31.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/opentracing-noop-0.31.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-codec-http-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-common-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-buffer-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-transport-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-resolver-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-codec-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-codec-http2-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/netty-handler-4.1.34.Final.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-media-jsonp-server-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-media-jsonp-common-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/javax.json-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/javax.json-api-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-config-yaml-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-config-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/reactor-core-3.1.5.RELEASE.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/reactive-streams-1.0.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-common-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/snakeyaml-1.24.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/javax.annotation-api-1.3.1.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-health-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/microprofile-health-api-1.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-health-checks-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-metrics-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/microprofile-metrics-api-1.1.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/org.osgi.annotation.versioning-1.0.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/cowsay-1.0.3.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/commons-lang3-3.3.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/commons-cli-1.3.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/maven-plugin-api-3.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/maven-model-3.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/plexus-utils-2.0.4.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/maven-artifact-3.0.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/sisu-inject-plexus-1.4.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/plexus-component-annotations-1.5.4.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/plexus-classworlds-2.2.3.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/sisu-inject-bean-1.4.2.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/sisu-guice-2.1.7-noaop.jar:/home/opc/work/helidon/helidon-cowweb/target/libs/helidon-graal-native-image-extension-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/classes:/home/opc/.m2/repository/io/helidon/bundles/helidon-bundles-webserver/1.1.2/helidon-bundles-webserver-1.1.2.jar:/home/opc/.m2/repository/io/helidon/webserver/helidon-webserver/1.1.2/helidon-webserver-1.1.2.jar:/home/opc/.m2/repository/io/helidon/common/helidon-common-reactive/1.1.2/helidon-common-reactive-1.1.2.jar:/home/opc/.m2/repository/io/helidon/common/helidon-common-http/1.1.2/helidon-common-http-1.1.2.jar:/home/opc/.m2/repository/io/helidon/common/helidon-common-context/1.1.2/helidon-common-context-1.1.2.jar:/home/opc/.m2/repository/io/helidon/media/helidon-media-common/1.1.2/helidon-media-common-1.1.2.jar:/home/opc/.m2/repository/io/helidon/common/helidon-common-key-util/1.1.2/helidon-common-key-util-1.1.2.jar:/home/opc/.m2/repository/io/helidon/common/helidon-common-configurable/1.1.2/helidon-common-configurable-1.1.2.jar:/home/opc/.m2/repository/io/opentracing/opentracing-util/0.31.0/opentracing-util-0.31.0.jar:/home/opc/.m2/repository/io/opentracing/opentracing-api/0.31.0/opentracing-api-0.31.0.jar:/home/opc/.m2/repository/io/opentracing/opentracing-noop/0.31.0/opentracing-noop-0.31.0.jar:/home/opc/.m2/repository/io/netty/netty-codec-http/4.1.34.Final/netty-codec-http-4.1.34.Final.jar:/home/opc/.m2/repository/io/netty/netty-common/4.1.34.Final/netty-common-4.1.34.Final.jar:/home/opc/.m2/repository/io/netty/netty-buffer/4.1.34.Final/netty-buffer-4.1.34.Final.jar:/home/opc/.m2/repository/io/netty/netty-transport/4.1.34.Final/netty-transport-4.1.34.Final.jar:/home/opc/.m2/repository/io/netty/netty-resolver/4.1.34.Final/netty-resolver-4.1.34.Final.jar:/home/opc/.m2/repository/io/netty/netty-codec/4.1.34.Final/netty-codec-4.1.34.Final.jar:/home/opc/.m2/repository/io/netty/netty-codec-http2/4.1.34.Final/netty-codec-http2-4.1.34.Final.jar:/home/opc/.m2/repository/io/netty/netty-handler/4.1.34.Final/netty-handler-4.1.34.Final.jar:/home/opc/.m2/repository/io/helidon/media/jsonp/helidon-media-jsonp-server/1.1.2/helidon-media-jsonp-server-1.1.2.jar:/home/opc/.m2/repository/io/helidon/media/jsonp/helidon-media-jsonp-common/1.1.2/helidon-media-jsonp-common-1.1.2.jar:/home/opc/.m2/repository/org/glassfish/javax.json/1.1.2/javax.json-1.1.2.jar:/home/opc/.m2/repository/javax/json/javax.json-api/1.1.2/javax.json-api-1.1.2.jar:/home/opc/.m2/repository/io/helidon/config/helidon-config-yaml/1.1.2/helidon-config-yaml-1.1.2.jar:/home/opc/.m2/repository/io/helidon/config/helidon-config/1.1.2/helidon-config-1.1.2.jar:/home/opc/.m2/repository/io/projectreactor/reactor-core/3.1.5.RELEASE/reactor-core-3.1.5.RELEASE.jar:/home/opc/.m2/repository/org/reactivestreams/reactive-streams/1.0.2/reactive-streams-1.0.2.jar:/home/opc/.m2/repository/io/helidon/common/helidon-common/1.1.2/helidon-common-1.1.2.jar:/home/opc/.m2/repository/org/yaml/snakeyaml/1.24/snakeyaml-1.24.jar:/home/opc/.m2/repository/javax/annotation/javax.annotation-api/1.3.1/javax.annotation-api-1.3.1.jar:/home/opc/.m2/repository/io/helidon/health/helidon-health/1.1.2/helidon-health-1.1.2.jar:/home/opc/.m2/repository/org/eclipse/microprofile/health/microprofile-health-api/1.0/microprofile-health-api-1.0.jar:/home/opc/.m2/repository/io/helidon/health/helidon-health-checks/1.1.2/helidon-health-checks-1.1.2.jar:/home/opc/.m2/repository/io/helidon/metrics/helidon-metrics/1.1.2/helidon-metrics-1.1.2.jar:/home/opc/.m2/repository/org/eclipse/microprofile/metrics/microprofile-metrics-api/1.1/microprofile-metrics-api-1.1.jar:/home/opc/.m2/repository/org/osgi/org.osgi.annotation.versioning/1.0.0/org.osgi.annotation.versioning-1.0.0.jar:/home/opc/.m2/repository/com/github/ricksbrown/cowsay/1.0.3/cowsay-1.0.3.jar:/home/opc/.m2/repository/org/apache/commons/commons-lang3/3.3.2/commons-lang3-3.3.2.jar:/home/opc/.m2/repository/commons-cli/commons-cli/1.3/commons-cli-1.3.jar:/home/opc/.m2/repository/org/apache/maven/maven-plugin-api/3.0/maven-plugin-api-3.0.jar:/home/opc/.m2/repository/org/apache/maven/maven-model/3.0/maven-model-3.0.jar:/home/opc/.m2/repository/org/codehaus/plexus/plexus-utils/2.0.4/plexus-utils-2.0.4.jar:/home/opc/.m2/repository/org/apache/maven/maven-artifact/3.0/maven-artifact-3.0.jar:/home/opc/.m2/repository/org/sonatype/sisu/sisu-inject-plexus/1.4.2/sisu-inject-plexus-1.4.2.jar:/home/opc/.m2/repository/org/codehaus/plexus/plexus-component-annotations/1.5.4/plexus-component-annotations-1.5.4.jar:/home/opc/.m2/repository/org/codehaus/plexus/plexus-classworlds/2.2.3/plexus-classworlds-2.2.3.jar:/home/opc/.m2/repository/org/sonatype/sisu/sisu-inject-bean/1.4.2/sisu-inject-bean-1.4.2.jar:/home/opc/.m2/repository/org/sonatype/sisu/sisu-guice/2.1.7/sisu-guice-2.1.7-noaop.jar:/home/opc/.m2/repository/io/helidon/integrations/graal/helidon-graal-native-image-extension/1.1.2/helidon-graal-native-image-extension-1.1.2.jar:/home/opc/work/helidon/helidon-cowweb/target/helidon-cowweb.jar \
[INFO] -H:Path=/home/opc/work/helidon/helidon-cowweb/target \
[INFO] -H:IncludeResources=application.yaml|META-INF/native-image/helidon-example-reflection-config.json|META-INF/native-image/native-image.properties|logging.properties|MessagesBundle_de.properties|MessagesBundle.properties|cows/www.cow|cows/vader.cow|cows/vader-koala.cow|cows/udder.cow|cows/tux.cow|cows/turtle.cow|cows/turkey.cow|cows/three-eyes.cow|cows/telebears.cow|cows/surgery.cow|cows/supermilker.cow|cows/stimpy.cow|cows/stegosaurus.cow|cows/squirrel.cow|cows/sodomized.cow|cows/small.cow|cows/skeleton.cow|cows/sheep.cow|cows/satanic.cow|cows/ren.cow|cows/mutilated.cow|cows/moose.cow|cows/moofasa.cow|cows/milk.cow|cows/meow.cow|cows/luke-koala.cow|cows/kosh.cow|cows/koala.cow|cows/kitty.cow|cows/kiss.cow|cows/hellokitty.cow|cows/head-in.cow|cows/ghostbusters.cow|cows/flaming-sheep.cow|cows/eyes.cow|cows/elephant.cow|cows/elephant-in-snake.cow|cows/dragon.cow|cows/dragon-and-cow.cow|cows/default.cow|cows/daemon.cow|cows/cower.cow|cows/cheese.cow|cows/bunny.cow|cows/bud-frogs.cow|cows/bong.cow|cows/beavis.zen.cow|cowfile-list.csv|META-INF/maven/org.sonatype.sisu/sisu-inject-plexus/pom.properties|META-INF/maven/org.sonatype.sisu/sisu-inject-bean/pom.properties|META-INF/maven/org.apache.maven/maven-artifact/pom.properties|META-INF/maven/org.apache.maven/maven-model/pom.properties|META-INF/maven/org.apache.maven/maven-plugin-api/pom.properties|META-INF/maven/org.codehaus.plexus/plexus-utils/pom.properties|META-INF/maven/org.codehaus.plexus/plexus-component-annotations/pom.properties|META-INF/maven/org.codehaus.plexus/plexus-classworlds/pom.properties|META-INF/maven/org.apache.commons/commons-lang3/pom.properties|META-INF/maven/commons-cli/commons-cli/pom.properties \
[INFO] -H:+ReportExceptionStackTraces \
[INFO] -H:Class=io.helidon.examples.quickstart.Main \
[INFO] -H:+ReportUnsupportedElementsAtRuntime \
[INFO] -H:+AllowIncompleteClasspath \
[INFO] -H:ReflectionConfigurationResources=META-INF/native-image/helidon-webserver-reflection-config.json \
[INFO] -H:ClassInitialization=io.netty.buffer.UnpooledByteBufAllocator$InstrumentedUnpooledUnsafeNoCleanerDirectByteBuf:run_time,io.netty.buffer.UnreleasableByteBuf:run_time,io.netty.handler.codec.http2.Http2ConnectionHandler:run_time,io.netty.handler.codec.http2.Http2CodecUtil:run_time,io.netty.handler.codec.http.HttpObjectEncoder:run_time,io.netty.handler.codec.http2.CleartextHttp2ServerUpgradeHandler:run_time,io.netty.handler.codec.http2.DefaultHttp2FrameWriter:run_time,io.netty.handler.codec.http2.Http2ServerUpgradeCodec:run_time,io.netty.handler.ssl.JdkNpnApplicationProtocolNegotiator:run_time,io.netty.handler.ssl.JettyNpnSslEngine:run_time,io.netty.handler.ssl.ReferenceCountedOpenSslEngine:run_time,io.netty.handler.ssl.ConscryptAlpnSslEngine:run_time \
[INFO] -H:ReflectionConfigurationResources=META-INF/native-image/helidon-media-jsonp-reflection-config.json \
[INFO] -H:ReflectionConfigurationResources=META-INF/native-image/helidon-config-yaml-reflection-config.json \
[INFO] -H:ReflectionConfigurationResources=META-INF/native-image/helidon-native-image-extension-reflection-config.json \
[INFO] -H:FallbackThreshold=0 \
[INFO] -H:ClassInitialization=io.helidon:build_time,io.netty:build_time,reactor.core:build_time,reactor.util:build_time,org.yaml.snakeyaml:build_time,org.reactivestreams:build_time,org.glassfish.json:build_time,org.eclipse.microprofile:build_time,io.opentracing:build_time,javax.json:build_time \
[INFO] -H:ReflectionConfigurationResources=META-INF/native-image/helidon-example-reflection-config.json \
[INFO] -H:ClassInitialization=io.helidon.webserver.ByteBufRequestChunk$OneTimeLoggerHolder:run_time \
[INFO] -H:CLibraryPath=/home/opc/opt/graalvm-ee-19.1.0/jre/lib/svm/clibraries/linux-amd64 \
[INFO] -H:Name=helidon-cowweb
[INFO] ]
[INFO] [helidon-cowweb:28423]    classlist:   7,507.52 ms
[INFO] [helidon-cowweb:28423]        (cap):   1,857.92 ms
[INFO] [helidon-cowweb:28423]        setup:   4,871.12 ms
[INFO] [DEBUG] (ForkJoinPool-2-worker-0) Using Console logging
[INFO] [helidon-cowweb:28423]   (typeflow):  32,332.01 ms
[INFO] [helidon-cowweb:28423]    (objects):  24,306.74 ms
[INFO] [helidon-cowweb:28423]   (features):   1,262.35 ms
[INFO] [helidon-cowweb:28423]     analysis:  58,778.63 ms
[INFO] [helidon-cowweb:28423]     (clinit):   1,247.50 ms
[INFO] [helidon-cowweb:28423]     universe:   2,483.38 ms
[INFO] [helidon-cowweb:28423]      (parse):   5,388.47 ms
[INFO] [helidon-cowweb:28423]     (inline):  12,906.53 ms
[INFO] [helidon-cowweb:28423]    (compile):  78,006.64 ms
[INFO] [helidon-cowweb:28423]      compile:  98,630.21 ms
[INFO] [helidon-cowweb:28423]        image:   3,683.73 ms
[INFO] [helidon-cowweb:28423]        write:     587.83 ms
[INFO] [helidon-cowweb:28423]      [total]: 176,847.36 ms
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  03:03 min
[INFO] Finished at: 2019-07-11T02:55:40Z
[INFO] ------------------------------------------------------------------------
```
