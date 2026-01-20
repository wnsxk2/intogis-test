# Docker Compose 환경 구축 가이드

## 목차

- [프로젝트 개요](#프로젝트-개요)
- [아키텍처](#아키텍처)
- [사전 요구사항](#사전-요구사항)
- [빠른 시작](#빠른-시작)
- [환경 변수 설정](#환경-변수-설정)
- [서비스 상세 설명](#서비스-상세-설명)
- [커스터마이징 가이드](#커스터마이징-가이드)
- [문제 해결](#문제-해결)
- [유용한 명령어](#유용한-명령어)

## 프로젝트 개요

Spring Boot 기반의 웹 애플리케이션으로, Docker Compose를 사용하여 다음 서비스들을 컨테이너로 구성합니다:

- **Nginx**: 리버스 프록시 및 웹 서버
- **WAS**: Spring Boot 애플리케이션 (Tomcat 10.1 + JDK 17)

## 아키텍처

```
[Client]
    ↓ (Port 7880)
[Nginx Container]
    ↓ (Internal Network)
[WAS Container (Tomcat + Spring Boot)]
    ↓
[External PostgreSQL Database]
```

### 네트워크 구성

- 모든 컨테이너는 `intogis` 브리지 네트워크를 통해 통신
- 외부 접속은 Nginx의 7880 포트로만 가능
- WAS는 외부에 직접 노출되지 않음 (보안 강화)

## 사전 요구사항

- Docker Desktop 또는 Docker Engine (20.10 이상)
- Docker Compose (v2.0 이상)
- 최소 2GB 이상의 여유 메모리
- PostgreSQL 데이터베이스 (별도 구성 필요)

### Docker 설치 확인

```bash
docker --version
docker compose version
```

## 빠른 시작

### 1. 환경 변수 파일 설정

`.env` 파일을 프로젝트 루트에 생성하고 데이터베이스 정보를 입력합니다:

```bash
# .env 파일 예시
DB_URL=jdbc:postgresql://192.168.0.100:5432/yourdb
DB_USER=admin
DB_PASSWORD=your_secure_password

NGINX_VERSION=stable-alpine
JDK_VERSION=3.8.5-openjdk-17
TOMCAT_VERSION=10.1-jdk17

TZ=Asia/Seoul
```

### 2. 서비스 시작

```bash
# 백그라운드에서 모든 서비스 시작
docker compose up -d

# 로그 확인
docker compose logs -f
```

### 3. 접속 확인

브라우저에서 `http://localhost:7880`으로 접속

### 4. 서비스 종료

```bash
# 컨테이너 중지 (볼륨 유지)
docker compose down

# 컨테이너 및 볼륨 완전 삭제
docker compose down -v
```

## 환경 변수 설정

`.env` 파일에서 관리하는 주요 환경 변수:

### 데이터베이스 설정

```bash
# PostgreSQL 연결 정보
DB_URL=jdbc:postgresql://[HOST]:[PORT]/[DATABASE_NAME]
DB_USER=your_username
DB_PASSWORD=your_password
```

**주의사항:**

- 호스트가 `localhost`인 경우, Docker 컨테이너 내부에서는 접근 불가
- Windows/Mac: `host.docker.internal` 사용
- Linux: 호스트의 실제 IP 주소 사용 (예: 192.168.0.100)

### 버전 설정

```bash
# Nginx 버전 (Docker Hub의 nginx 태그)
NGINX_VERSION=stable-alpine    # 권장: alpine 베이스 (경량)

# Maven 빌드용 JDK 버전
JDK_VERSION=3.8.5-openjdk-17   # Maven 3.8.5 + OpenJDK 17

# Tomcat 런타임 버전
TOMCAT_VERSION=10.1-jdk17      # Tomcat 10.1 + JDK 17
```

### 기타 설정

```bash
# 컨테이너 타임존
TZ=Asia/Seoul                  # 서울 표준시 (KST)
```

## 서비스 상세 설명

### 1. Nginx (리버스 프록시)

#### 역할

- 외부 요청을 받아 WAS로 프록시
- 정적 파일 서빙 (필요시)
- SSL/TLS 터미네이션 (HTTPS 설정 시)
- 로드 밸런싱 (다중 WAS 구성 시)

#### 주요 설정 파일

- `nginx/Dockerfile`: Nginx 이미지 빌드 설정
- `nginx/conf.d/default.conf`: Nginx 가상 호스트 설정

#### 포트 매핑

```yaml
ports:
  - '7880:80' # [호스트 포트]:[컨테이너 포트]
```

#### 볼륨 마운트

```yaml
volumes:
  - ./nginx/conf.d:/etc/nginx/conf.d:ro # 설정 파일 (읽기 전용)
```

### 2. WAS (Web Application Server)

#### 역할

- Spring Boot 애플리케이션 실행
- 비즈니스 로직 처리
- 데이터베이스 연결

#### 빌드 프로세스 (Multi-stage Build)

1. **Stage 1 (Maven Build)**
   - Maven으로 프로젝트 빌드
   - 의존성 다운로드 및 캐싱
   - WAR 파일 생성

2. **Stage 2 (Tomcat Runtime)**
   - 빌드된 WAR 파일만 복사
   - 최종 이미지 크기 최소화

#### 환경 변수

```yaml
environment:
  - TZ=${TZ} # 타임존
  - DB_URL=${DB_URL} # 데이터베이스 URL
  - DB_USER=${DB_USER} # DB 사용자명
  - DB_PASSWORD=${DB_PASSWORD} # DB 비밀번호
```

## 커스터마이징 가이드

### 1. 포트 변경

#### Nginx 외부 포트 변경

`docker-compose.yml` 파일 수정:

```yaml
nginx:
  ports:
    - '8080:80' # 7880 → 8080으로 변경
```

### 2. Nginx 설정 커스터마이징

#### 업스트림 서버 변경

`nginx/conf.d/default.conf` 파일에서:

```nginx
upstream service_backend {
    server was:8080;           # 컨테이너명:포트
    keepalive 32;
}
```

#### 다중 WAS 로드 밸런싱

```nginx
upstream service_backend {
    server was1:8080;
    server was2:8080;
    keepalive 32;
}
```

#### 타임아웃 조정

```nginx
server {
    client_max_body_size 100M;      # 최대 업로드 크기
    proxy_read_timeout 300s;        # 읽기 타임아웃
    proxy_connect_timeout 75s;      # 연결 타임아웃
}
```

#### HTTPS 설정 (SSL/TLS)

```nginx
server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # ... 나머지 설정
}
```

볼륨 추가:

```yaml
nginx:
  volumes:
    - ./nginx/conf.d:/etc/nginx/conf.d:ro
    - ./nginx/ssl:/etc/nginx/ssl:ro # SSL 인증서 추가
```

### 3. 데이터베이스 변경

#### MySQL 사용 시

`.env` 파일:

```bash
DB_URL=jdbc:mysql://host.docker.internal:3306/yourdb?useSSL=false&characterEncoding=UTF-8
DB_USER=root
DB_PASSWORD=your_password
```

`was/pom.xml`에 드라이버 추가:

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>
```

### 4. JVM 메모리 설정

`was/Dockerfile` 수정:

```dockerfile
ENV JAVA_OPTS="-Xms512m -Xmx2048m -Dfile.encoding=UTF-8"
```

### 5. 로그 파일 호스트에 저장

`docker-compose.yml`에 볼륨 추가:

```yaml
was:
  volumes:
    - ./logs:/usr/local/tomcat/logs # Tomcat 로그
```

### 6. 개발 환경 설정

소스 코드 실시간 반영 (볼륨 마운트):

```yaml
was:
  volumes:
    - ./was/src:/app/src # 소스 코드 마운트
  environment:
    - SPRING_DEVTOOLS_RESTART_ENABLED=true
```

### 7. Docker Compose 프로필 사용

`docker-compose.yml`:

```yaml
services:
  was:
    # ... 기본 설정
    profiles:
      - prod

  was-dev:
    extends: was
    profiles:
      - dev
    environment:
      - SPRING_PROFILES_ACTIVE=dev
    volumes:
      - ./was/src:/app/src
```

실행:

```bash
# 프로덕션 모드
docker compose --profile prod up -d

# 개발 모드
docker compose --profile dev up -d
```

## 문제 해결

### 1. 컨테이너가 시작되지 않을 때

```bash
# 컨테이너 상태 확인
docker compose ps

# 로그 확인
docker compose logs [service-name]

# 예: WAS 로그 확인
docker compose logs was
```

### 2. 데이터베이스 연결 실패

**증상:**

```
java.sql.SQLException: Connection refused
```

**해결 방법:**

1. DB 호스트 확인 (localhost → host.docker.internal 또는 실제 IP)
2. DB 서버가 실행 중인지 확인
3. 방화벽 설정 확인
4. DB 연결 정보 (포트, 사용자명, 비밀번호) 재확인

### 3. Nginx 502 Bad Gateway

**원인:**

- WAS 컨테이너가 시작되지 않음
- 네트워크 설정 오류

**해결 방법:**

```bash
# WAS 상태 확인
docker compose ps was

# WAS 로그 확인
docker compose logs was

# 네트워크 확인
docker network ls
docker network inspect intogis3_intogis
```

### 4. 빌드 실패

```bash
# 캐시 무시하고 재빌드
docker compose build --no-cache

# 특정 서비스만 재빌드
docker compose build --no-cache was
```

### 5. 포트 충돌

**증상:**

```
Error: bind: address already in use
```

**해결 방법:**

```bash
# Windows
netstat -ano | findstr :7880

# Linux/Mac
lsof -i :7880

# 포트 변경 또는 충돌 프로세스 종료
```

## 유용한 명령어

### 컨테이너 관리

```bash
# 서비스 시작 (빌드 + 실행)
docker compose up -d --build

# 특정 서비스만 시작
docker compose up -d nginx

# 서비스 재시작
docker compose restart

# 서비스 중지
docker compose stop

# 컨테이너 완전 삭제
docker compose down -v

# 실행 중인 컨테이너 확인
docker compose ps
```

### 로그 확인

```bash
# 전체 로그 실시간 확인
docker compose logs -f

# 특정 서비스 로그
docker compose logs -f was

# 최근 100줄만 확인
docker compose logs --tail=100 was
```

### 컨테이너 내부 접속

```bash
# WAS 컨테이너 쉘 접속
docker compose exec was bash

# Nginx 컨테이너 쉘 접속
docker compose exec nginx sh
```

### 리소스 사용량 확인

```bash
# 실시간 모니터링
docker stats

# 디스크 사용량
docker system df

# 이미지 목록
docker images
```

### 클린업

```bash
# 사용하지 않는 컨테이너/네트워크/이미지 삭제
docker system prune

# 모든 리소스 완전 삭제 (주의!)
docker system prune -a --volumes
```

## 추가 리소스

### Docker 공식 문서

- [Docker Compose Reference](https://docs.docker.com/compose/)
- [Dockerfile Best Practices](https://docs.docker.com/develop/dev-best-practices/)

### 이미지 저장소

- [Nginx Docker Hub](https://hub.docker.com/_/nginx)
- [Maven Docker Hub](https://hub.docker.com/_/maven)
- [Tomcat Docker Hub](https://hub.docker.com/_/tomcat)

### 프로젝트 구조

```
INToGIS3/
├── docker-compose.yml          # Docker Compose 메인 설정
├── .env                        # 환경 변수 파일
├── nginx/
│   ├── Dockerfile             # Nginx 이미지 빌드 파일
│   └── conf.d/
│       └── default.conf       # Nginx 가상 호스트 설정
└── was/
    ├── Dockerfile             # WAS 이미지 빌드 파일
    ├── pom.xml                # Maven 프로젝트 설정
    └── src/                   # Spring Boot 소스 코드
```
