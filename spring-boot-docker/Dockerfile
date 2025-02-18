# 基于Gradle和Java 21的构建镜像
FROM gradle:8.12-jdk21-alpine AS build

# 设置工作目录
WORKDIR /app

# 复制Gradle构建文件
COPY build.gradle .

# 下载项目依赖
RUN gradle --no-daemon dependencies

# 复制项目源代码
COPY src ./src

# 构建Spring Boot项目
RUN gradle --no-daemon build

# 使用Java 21运行时镜像
FROM eclipse-temurin:21-jre-alpine

# 设置工作目录
WORKDIR /app

# 从构建阶段复制生成的Spring Boot可执行JAR包
COPY --from=build /app/build/libs/*.jar app.jar

# 暴露Spring Boot应用的默认端口，默认为8080
EXPOSE 8080

# 定义容器启动时执行的命令
ENTRYPOINT ["java", "-jar", "app.jar"]