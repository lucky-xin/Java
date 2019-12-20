# SpringCloud项目快速构建镜像并部署
## SpringCloud pom.xml配置如下，这个是作为其它微服务开发模块pom.xml的parent
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0"
		 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>biz.datainsights</groupId>
	<artifactId>datainsights-framework</artifactId>
	<version>3.5.0</version>
	<name>${project.artifactId}</name>
	<packaging>pom</packaging>

	<organization>
		<name>DataInsights</name>
		<url>http://www.datainsights.com.hk/</url>
	</organization>

	<parent>
		<artifactId>spring-cloud-dependencies-parent</artifactId>
		<groupId>org.springframework.cloud</groupId>
		<version>2.1.6.RELEASE</version>
	</parent>

	<developers>
		<developer>
			<name>Luchaoxin</name>
			<email>cxlu@datainsights.biz</email>
			<roles>
				<role>leader</role>
			</roles>
		</developer>
	</developers>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<maven.compiler.source>1.8</maven.compiler.source>
		<maven.compiler.target>1.8</maven.compiler.target>
		<skipTests>true</skipTests>

		<spring-boot.version>2.1.6.RELEASE</spring-boot.version>
		<spring-cloud.version>Greenwich.RELEASE</spring-cloud.version>
		<spring-platform.version>Cairo-SR8</spring-platform.version>

		<spring-boot-admin.version>2.1.6</spring-boot-admin.version>
		<security.oauth.version>2.3.8.RELEASE</security.oauth.version>
		<security.oauth.auto.version>2.1.6.RELEASE</security.oauth.auto.version>

		<hutool.version>5.0.7</hutool.version>
		<swagger.fox.version>2.9.2</swagger.fox.version>
		<jasypt.version>2.1.1</jasypt.version>
		<ttl.version>2.10.2</ttl.version>
		<minio.version>6.0.8</minio.version>
		<logstash.encoder.version>6.2</logstash.encoder.version>
		<json.schema.version>2.2.11</json.schema.version>
		<spring.kafka.version>2.2.7.RELEASE</spring.kafka.version>
		<snakeyaml.vsercion>1.23</snakeyaml.vsercion>

		<elastic-job-lite.version>2.1.5</elastic-job-lite.version>
		<xxl.job.version>2.1.0</xxl.job.version>
		<activiti.version>5.22.0</activiti.version>
		<lcn.version>4.1.0</lcn.version>
		<swagger.core.version>1.5.22</swagger.core.version>
		<curator.version>2.10.0</curator.version>
		<google.findbugs.version>3.0.1</google.findbugs.version>

		<mp.weixin.version>3.3.0</mp.weixin.version>
		<lombok.version>1.18.10</lombok.version>
		<fastjson.version>1.2.62</fastjson.version>
		<aliyun.java.sdk.version>4.1.0</aliyun.java.sdk.version>
		<kaptcha.version>0.0.9</kaptcha.version>
		<validation.api.version>2.0.1.Final</validation.api.version>
		<velocity.version>1.7</velocity.version>
		<freemarker.version>2.3.28</freemarker.version>
		<!--		数据库相关配置-->
		<postgresql.version>42.2.8</postgresql.version>
		<sharding.jdbc.version>4.0.0-RC2</sharding.jdbc.version>
		<druid.version>1.1.21</druid.version>
		<mybatis-plus.version>3.2.0</mybatis-plus.version>
		<mysql.connector.version>8.0.17</mysql.connector.version>
		<!--		datainsights相关配置-->
		<datainsights.version>3.5.0</datainsights.version>
		<datainsights.utils.version>1.3.5</datainsights.utils.version>
		<elasticsearch.version>5.2.4</elasticsearch.version>
		<datainsights.boot.starter.version>2.0.8</datainsights.boot.starter.version>
		<!--docker 镜像相关配置-->
		<docker.plugin.version>1.0.0</docker.plugin.version>
		<base.image>openjdk:8-jre-alpine</base.image>
		<image.tag>3.5.6</image.tag>
		<registry.url>192.168.10.82:5555</registry.url>
		<server.port>8080</server.port>

	</properties>

	<dependencies>
		<!--依赖-->
	
	</dependencies>

	<modules>
		
	</modules>

    <!-- 版本控制，统一管理所有公共的依赖版本-->
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>${spring-boot.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>io.spring.platform</groupId>
				<artifactId>platform-bom</artifactId>
				<version>${spring-platform.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<!--spring cloud alibaba-->
			<dependency>
				<groupId>com.alibaba.cloud</groupId>
				<artifactId>spring-cloud-alibaba-dependencies</artifactId>
				<version>2.1.0.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>

		</dependencies>
	</dependencyManagement>

	<build>
		<finalName>${project.name}</finalName>
		<resources>
			<resource>
				<directory>src/main/resources</directory>
				<filtering>true</filtering>
			</resource>
		</resources>
        <!--插件统一管理-->
		<pluginManagement>
			<plugins>
				<plugin>
					<groupId>org.springframework.boot</groupId>
					<artifactId>spring-boot-maven-plugin</artifactId>
					<version>${spring-boot.version}</version>
					<executions>
						<execution>
							<goals>
								<goal>repackage</goal>
							</goals>
						</execution>
					</executions>
				</plugin>
				<plugin>
                    <!-- 构建SpringCloud镜像插件-->
					<groupId>com.spotify</groupId>
					<artifactId>docker-maven-plugin</artifactId>
					<version>${docker.plugin.version}</version>
					<dependencies>
						<dependency>
							<groupId>javax.activation</groupId>
							<artifactId>javax.activation-api</artifactId>
							<version>1.2.0</version>
						</dependency>
					</dependencies>
					<!--将插件绑定在某个phase执行-->
					<executions>
						<execution>
							<id>build-image</id>
						<!--将插件绑定在build这个phase上。也就是说，用户只需执行mvn build ，就会自动执行mvn docker:build-->
							<phase>package</phase>
							<goals>
								<goal>build</goal>
							</goals>
						</execution>
        
	                    <execution>
                            <id>push-image</id>
                            <phase>deploy</phase>
                            <goals>
                            <!--将插件绑定在deploy这个phase上。也就是说，用户只需执行mvn deploy ，就会推送镜像到镜像注册中心-->
                                <goal>push</goal>
                            </goals>
                            <configuration>
                                <imageName>${registry.url}/${project.name}:${image.tag}</imageName>
                            </configuration>
                        </execution>
					</executions>
					<configuration>
                        <!--            镜像中心url以及项目名称-->
						<imageName>${registry.url}/${project.name}</imageName>
						<!--指定标签-->
						<imageTags>
							<imageTag>${image.tag}</imageTag>
						</imageTags>
                        <!--       构建镜像Dockerfile所在目录  -->
						<dockerDirectory>${project.basedir}</dockerDirectory>
                        <!--       构建镜像所有ARG变量  -->
						<buildArgs>
							<APPLICATION_NAME>${project.name}</APPLICATION_NAME>
							<SERVER_PORT>${server.port}</SERVER_PORT>
							<VERSION>${project.version}</VERSION>
							<BASE_IMAGE>${base.image}</BASE_IMAGE>
						</buildArgs>
						<resources>
							<resource>
                                <!--         jar包配置       -->
								<targetPath>/</targetPath>
								<directory>${project.build.directory}</directory>
								<include>${project.build.finalName}.jar</include>
							</resource>
						</resources>
						<serverId>docker-hub</serverId>
						<registryUrl>https://index.docker.io/v1/</registryUrl>
					</configuration>
				</plugin>
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-resources-plugin</artifactId>
					<version>3.1.0</version>
				</plugin>
			</plugins>
		</pluginManagement>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.8.0</version>
				<configuration>
					<target>${maven.compiler.target}</target>
					<source>${maven.compiler.source}</source>
					<encoding>UTF-8</encoding>
					<skip>true</skip>
				</configuration>
			</plugin>
			<plugin>
				<groupId>pl.project13.maven</groupId>
				<artifactId>git-commit-id-plugin</artifactId>
				<version>2.2.5</version>
			</plugin>
			<plugin>
				<artifactId>maven-source-plugin</artifactId>
				<version>2.4</version>
				<configuration>
					<attach>true</attach>
				</configuration>
				<executions>
					<execution>
						<phase>compile</phase>
						<goals>
							<goal>jar</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

	<repositories>
	
		<!--阿里云私服-->
		<repository>
			<id>aliyun</id>
			<name>aliyun</name>
			<url>http://maven.aliyun.com/nexus/content/groups/public</url>
		</repository>
		<!--gitee 私服-->
		<repository>
			<id>gitee.wang</id>
			<name>gitee.wang</name>
			<url>http://nexus.gitee.wang/repository/maven-releases/</url>
		</repository>

	</repositories>
</project>

```
```text
Dockerfile在SpringCloud项目根目录下内容如下,JVM使用Garbage First 垃圾收集器,SpringCloud日志目录可以指定默认为/var/datainsights-logs
JVM大小可以指定默认512M,也可以指定JAVA_OPTS替换默认jar启动命令，所有的参数都为变量SpringCloud的Dockerfile通用
```
```dockerfile
ARG BASE_IMAGE=openjdk:8-jre-alpine
FROM ${BASE_IMAGE}

ARG APPLICATION_NAME
ARG SERVER_PORT

ENV APPLICATION_NAME ${APPLICATION_NAME}
ENV SERVER_PORT ${SERVER_PORT}

ENV LOG_DIR ${LOG_DIR:-/var/datainsights-logs}

ENV HEAP_SIZE ${HEAP_SIZE:-512M}

MAINTAINER "Luchaoxin<cxlu@datainsights.biz>"

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo 'Asia/Shanghai' > /etc/timezone \
    && mkdir -p /${APPLICATION_NAME}    \
    && mkdir -p /${LOG_DIR}/${APPLICATION_NAME}

WORKDIR /${APPLICATION_NAME}

EXPOSE ${SERVER_PORT}

ADD target/${APPLICATION_NAME}.jar ./

ENTRYPOINT sleep 10;java -Djava.security.egd=file:/dev/./urandom ${JAVA_OPTS:--Xms${HEAP_SIZE} -Xmx${HEAP_SIZE} -Xss256k -XX:+UseG1GC -XX:MaxGCPauseMillis=200} -jar ${APPLICATION_NAME}.jar
```
```text
如果有自己的镜像注册中心，并且配置了外网访问，则docker-compose,k8s获取镜像就特别方便，以下是内网镜像注册中心，挂载了日志到本机使用ELK
统一收集日志
```
```yaml
version: '3.7'
services:

  datainsights-sns-endpoint:
    image: 192.168.10.82:5555/datainsights-sns-endpoint:latest
    restart: always
    container_name: datainsights-sns-endpoint
    volumes:
      - /var/datainsights-logs/datainsights-sns-endpoint:/var/datainsights-logs/datainsights-sns-endpoint
    env_file:
      - ./.env
    ports:
      - 3001:3001
    networks:
      - datainsights-framework-net
    depends_on:
      - datainsights-redis
```