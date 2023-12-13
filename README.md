Kubernetes 환경에서 새로운 Pod 기동 시간 최소화하기:Native Image로의 전환
=====================================

## 배경
### Spring Boot 2의 EOS와 3버전의 제약조건
 - 23년 11월 24일 Spring Boot 2.0 EOS
   - OSS(Open Source Software) 지원이 23년 11월 24일 종료
  <img width="716" alt="Screenshot 2023-12-11 at 9 52 14 AM" src="https://github.com/u4rang/NativeImageWithGraalVM/assets/26420767/7229d4ad-5962-4183-bfd9-c2226d56b25e">   
  출처 : [spring.io 사이트의 Spring Boot Support Plan](https://spring.io/projects/spring-boot#support)
   - LEGO Framework
     - 지원 버전 (2023.12.11 기준) 
       - Spring Boot : 2.7 
       - JDK : 1.8
     - 출처 : [LEGO 홈페이지](https://code.sdsdev.co.kr/LEGO/2.0)
     - 개인적인 의견
       - 24년은 JDK 버전 업그레이드와 Spring Boot 3 버전으로 변경하는 프로젝트 다수 예상
 - 22년 11월 24일 Spring Boot 3.0의 정식 릴리즈
   - JDK : 17 버전 이상
   - GraalVM을 사용한 Native Image 생성 기능 추가

### Kubernetes 환경에서 새로운 Pod 추가 시, 기동에 필요한 시간 줄이기
 - Kubernetes 환경(GKE)에서 1개에서 20개의 Pod로 Scale out 시, 약 2분 이상의 시간이 소요되었습니다.
<img width="571" alt="Screenshot 2023-12-13 at 11 14 47 AM" src="https://github.com/u4rang/NativeImageWithGraalVM/assets/26420767/c15c5ae7-eb2e-4222-8cf8-7786e7cc69fd">

## Native Image의 개요
### Native Image 란 ?
 - Java code를 미리 바이너리(Native executable)로 컴파일 하는 기술

   ```
   			Native Image Bulider
   Java code ---------------------------> Native executable
   			Native Image Generator
   ```

 - Native executable은 '필요한 리소스의 일부만' 을 사용해서 빠른 장점

### 기존 Image와의 차이점
| 특징                           | Native Image | 기존 Image    |
| ---------------------------- | ------------ | ----------- |
| 실행 방식                        | 네이티브 코드      | JVM         |
| 시작 시간                        | 매우 짧음        | 상대적으로 오래 걸림 |
| 실행 속도                        | 매우 빠름        | 상대적으로 느림    |
| 메모리 사용량                      | 최소화됨         | 상대적으로 많음    |
| JVM 설치 필요성                   | 불필요          | 필요          |
| 애플리케이션 크기                    | 비교적 큼        | 비교적 작음      |
| 종속성 변경 시 Native Image 재생성 여부 | 필요           | 필요 없음       |

## Native Image의 제역사항, 추천 환경 그리고 장단점
### Native Image의 고려사항
 - 네이티브 이미지는 이미지 빌드 시에 이미 알려진 폐쇄된 환경을 가정
 - 다시 말해, 이미지를 빌드할 때 어플리케이션의 모든 바이트 코드에 대한 정보가 제공되어야 하며, 런타임 중에는 새로운 코드가 로드되지 않음
 - 따라서 동적으로 생성되는 부분들은 네이티브 이미지를 빌드할 때 해당 정보가 전달 필요
   - Reflection
   - Resource
   - JNI
   - Dynamic Proxies


### Native Image 적용 시 장단점
#### 장점
 - 시작 시간과 실행 속도가 크게 향상
 - 메모리 사용량이 최소화
 - JVM을 설치할 수 없는 환경에서도 실행

#### 단점
 - 애플리케이션의 크기가 커짐
   - [upx 툴](https://upx.github.io/)를 사용하여 Native Image 크기 작게 생성 가능 
 - 애플리케이션의 종속성이 변경될 때마다 Native Image를 다시 생성 필요

### Native Image 추천 환경
 - 시작 시간과 실행 속도를 최우선으로 고려하는 시스템
 - 메모리 사용량을 최소화해야 하는 시스템
 - JVM을 설치할 수 없는 환경에서 실행해야 하는 애플리케이션
 - ex. 실시간 처리 시스템,

## Native Image 빌드 및 실행 도구 GraalVM

### GraalVM 이란
 - The General Recursive Applicative and Algorithmic Language Virtual Machine
 - 발음 : 그랄 VM
 - 고성능 JDK
 - 네이티브 이미지 빌더

### GraalVM 설치방법
 - [OS별 GraalVM 설치방법 in graalvm.org](https://www.graalvm.org/latest/reference-manual/native-image/#:~:text=Further%20Reading-,Prerequisites,-%23)

### Native Image를 가능하게 하는 'AOT 네이티브 이미지 컴파일러'
 - Code를 최적화하고 컴파일 후 Native Code로 제공
 - Native Image 컴파일이 가능하다는 것은 Java가 버이너리 파일을 만들수 있다는 의미

### 필요한 리소스의 일부만 사용하는 방법
 - 정적 분석 프로세스를 통해 필요한 리소스를 다음과 같이 스캔
   ```
   Java code ---> byte code(class file)을 스캔
   ```
 - 네이티브 이미지 힙이라는 메모리 영역에서 루트객체(main)부터 스캔 -> 연결할수 있는 클래스 확인
-> 새로 검색된 요소는 추가 스캔(즉, 요소의 도달 가능성 반복적으로 스캔)
 - 정적 분석(Static Analystic) : 애플리케이션에서 사용되는 프로그램 요소(class, method, field)을 결정하는 프로세스


## 기존 이미지와 Native Image 적용 시 성능 비교
### 'Hello World' 실행 시, 성능 비교
 - Source Code : Hello.java
   ```java
   public class Hello {
     public static void main(String[] args) {
       System.out.println("Hello World");
     }
   }
   ```

 - 명령어
   - OpenJDK JIT
     ```sh
     # Compile
     javac Hello.java
     
     # Run
     time java Hello
     ```
   - GraalVM Native
     ```sh
     # Compile
     native-image Hello.java
     
     # Run
     time ./Hello
     ```
- 수행결과
  ![image](https://github.com/u4rang/NativeImageWithGraalVM/assets/26420767/6bd5d236-8125-4660-acd4-4a7a805e78c0)

   | 구 분   | OpenJDK | Native Image |
   |--------|---------|--------------|
   | 소요시간 |   0.04 s |  0.27 s       |
   | 메모리사용량 | 39408 KB | 6384 KB    |

### Pod 기동에 소요되는 시간 성능 비교


## 결론
### Native Image의 한계점

### Native Image의 전망

## 참고자료
### GraalVM 이란
 - The General Recursive Applicative and Algorithmic Language Virtual Machine
 - 그랄VM
 - 고성능 JDK
 - 네이티브 이미지 빌더
 - [GraalVM in Youtube](https://www.youtube.com/@GraalVM)
 - [GraalVM 릴리즈](https://github.com/oracle/graal/releases)
 - [GraalVM란?](https://velog.io/@new_ego_doc/GraalVM%EB%9E%80)
 - [GraalVM 기반 컨테이너 이미지 만들기](https://thekoguryo.github.io/oracle-cloudnative/appdev/2.graalvm/1.use-graal-vm/)
 - [GraalVM in Oracle](https://docs.oracle.com/search/?q=GraalVM+Native+Image)
 - [Native Image in GraalVM](https://www.graalvm.org/latest/reference-manual/native-image/)

### zsh에서 time 명령어의 출력 형식 변경
```sh
TIMEFMT=$'\n# Elapsed Time: %E s Max RSS: %M KB'
```