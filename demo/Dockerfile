# Stage 1: Build stage
FROM maven:3.8.5-openjdk-17 AS build
WORKDIR /app

# 1. 의존성 파일들을 먼저 복사하여 캐시 활용
COPY pom.xml .
RUN mvn dependency:go-offline

# 2. 소스 코드 복사 및 WAR 파일 빌드
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run stage
FROM tomcat:10.1-jdk17
WORKDIR /usr/local/tomcat/webapps/

# 3. 빌드 스테이지에서 생성된 WAR 파일을 Tomcat의 webapps 디렉토리로 복사
# 기본적으로 Spring Boot WAR는 ROOT.war로 이름을 바꾸면 컨텍스트 경로가 '/'가 됩니다.
COPY --from=build /app/target/*.war ./ROOT.war

# Tomcat 기본 포트 노출
EXPOSE 8080

# Tomcat 시작
CMD ["catalina.sh", "run"]
