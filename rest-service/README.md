### 프로젝트
#### 코드 빌드
```sh
./mvnw clean package
```

#### Jar 실행
```sh
java -jar target/rest-service-0.0.1-SNAPSHOT.jar
```

#### 서비스 테스트
```sh
curl http://localhost:8080/greeting; echo  
```

### Container 이미지 만들기
#### Dockerfile
```sh
FROM openjdk:8-jdk-alpine
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","app.jar"]
```

#### 이미지 빌드
```sh
docker build -t spring-boot-greeting:1.0 .
```