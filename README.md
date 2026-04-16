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

## 총평 ##
순수 Java 코드는 JVM 위에서 돌아가기 때문에 x86이든 ARM64든 바이트코드 레벨에서 이미 플랫폼 독립적이다. 
이 transform이 실질적으로 의미 있는 경우는 굉장히 제한적이다.
```
JNI를 통해 네이티브 라이브러리(.so)를 직접 로딩하는 경우
네이티브 바인딩이 포함된 의존성을 쓰는 경우 (Netty epoll, RocksDB, LevelDB, SQLite JDBC 등)
Dockerfile이나 빌드 스크립트에 x86 아키텍처가 하드코딩된 경우
```

결국 이 도구의 본질은 "코드 변환"이라기보다는:

```
프로젝트 전체를 스캔해서 ARM64에서 문제될 수 있는 의존성을 찾아내고
해당 의존성의 ARM64 호환 버전으로 올려주고
빌드/테스트를 돌려서 검증해주는
일종의 호환성 검증 + 의존성 업그레이드 자동화 도구에 가깝습니다.
```

공식 문서에서도 "Many modern Java applications are already Arm64-compatible"이라고 직접 언급하고 있다. 즉, 대부분의 순수 Java 앱은 이 transform을 돌려도 변경 사항이 거의 없을 수 있다는 걸 AWS도 인정하는 셈이다.

```
실제로 가치가 있는 시나리오는 레거시 Java 프로젝트에서 오래된 네이티브 의존성을 많이 쓰고 있는데, 어떤 게 ARM64에서 문제가 되는지 일일이 파악하기 어려울 때 정도이다. 대규모 마이그레이션에서 수십~수백 개 서비스를 한꺼번에 Graviton으로 옮길 때 자동 스캔 용도로 쓰는 거라고 보면 적절하다.
```

"변환"이라는 이름이 좀 과한 것 같고, "검증 도구"가 더 정확한 표현일 듯 하다.
