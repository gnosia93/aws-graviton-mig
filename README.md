## [AWS Transform Customs](https://docs.aws.amazon.com/transform/latest/userguide/transform-aws-customs.html) ##

### AWS/early-access-java-x86-to-graviton 구체적 기능 ###
이 transform은 Java 애플리케이션이 ARM64(Graviton) 환경에서 잘 동작하는지 검증하고, 호환되지 않는 부분을 자동으로 수정해주는 도구이다.

#### 핵심 기능 4가지 ####
* ARM64 호환성 검증 
  * Java 애플리케이션이 ARM64 아키텍처에서 동작 가능한지 전체적으로 분석합니다.
* 의존성 업데이트
  * ARM64와 호환되지 않는 라이브러리/의존성을 찾아내고, 호환 가능한 버전으로 업데이트합니다.
* 아키텍처 특정 코드 패턴 탐지 및 수정
  * 아키텍처 감지 로직 (예: os.arch 기반 분기)
  * 네이티브 라이브러리 로딩 코드 (.so 파일 등)
  * x86에 종속된 코드 패턴을 ARM64에서도 동작하도록 수정
* 네이티브 라이브러리 재컴파일
  * 소스 코드가 있는 네이티브 라이브러리의 경우 ARM64용으로 재컴파일합니다.

#### 하지 않는 것 ####
* 일반적인 코드 리팩토링은 수행하지 않습니다. 오직 ARM64 호환에 필요한 변경만 합니다.
* Java 버전이나 JDK 배포판을 변경하지 않습니다. 현재 사용 중인 버전을 그대로 유지합니다.

#### 검증 방식 ####
변환 후 빌드와 테스트를 실행해서 호환성을 검증합니다.

#### 권장 사항 ####
* 최적의 결과를 위해 ARM64 기반 환경에서 실행하는 것을 권장합니다 (예: Graviton EC2 인스턴스).
* 공식 문서에서도 언급하듯이, 많은 최신 Java 애플리케이션은 이미 ARM64 호환이 되어 있어서 변경이 거의 없을 수도 있습니다.

#### 사용 예시 ####
```
codeRepositoryPath: ./my-java-app
transformationName: AWS/early-access-java-x86-to-graviton
buildCommand: mvn clean install
additionalPlanContext: |
  Ensure all JNI native libraries are compatible with ARM64.
  Validate against our integration test suite.
```
