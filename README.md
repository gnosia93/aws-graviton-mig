## [AWS Transform Customs](https://docs.aws.amazon.com/transform/latest/userguide/transform-aws-customs.html) ##

### AWS/early-access-java-x86-to-graviton 구체적 기능 ###
이 transform은 Java 애플리케이션이 ARM64(Graviton) 환경에서 잘 동작하는지 검증하고, 호환되지 않는 부분을 자동으로 수정해주는 도구이다.

#### 핵심 기능 4가지 ####
* ARM64 호환성 검증 
  * Java 애플리케이션이 ARM64 아키텍처에서 동작 가능한지 전체적으로 분석한다.
* 의존성 업데이트
  * ARM64와 호환되지 않는 라이브러리/의존성을 찾아내고, 호환 가능한 버전으로 업데이트한다.
* 아키텍처 특정 코드 패턴 탐지 및 수정
  * 아키텍처 감지 로직 (예: os.arch 기반 분기)
  * 네이티브 라이브러리 로딩 코드 (.so 파일 등)
  * x86에 종속된 코드 패턴을 ARM64에서도 동작하도록 수정
* 네이티브 라이브러리 재컴파일
  * 소스 코드가 있는 네이티브 라이브러리의 경우 ARM64용으로 재컴파일한다.

#### 하지 않는 것 ####
* 일반적인 코드 리팩토링은 수행하지 않는다. 오직 ARM64 호환에 필요한 변경만 한다.
* Java 버전이나 JDK 배포판을 변경하지 않는다. 현재 사용 중인 버전을 그대로 유지한다.

#### 검증 방식 ####
변환 후 빌드와 테스트를 실행해서 호환성을 검증한다.
유닛 테스트는 사용자가 직접 작성해야 하고, 테스트 케이스가 없는 경우 테스트 하지 않는다. 즉 transform 이 직접 테스트 코드를 만들어 주지는 않는다.

#### 권장 사항 ####
* 최적의 결과를 위해 ARM64 기반 환경에서 실행하는 것을 권장한다 (예: Graviton EC2 인스턴스).
* 공식 문서에서도 언급하듯이, 많은 최신 Java 애플리케이션은 이미 ARM64 호환이 되어 있어서 변경이 거의 없을 수도 있다.

#### 사용 예시 ####
```
codeRepositoryPath: ./my-java-app
transformationName: AWS/early-access-java-x86-to-graviton
buildCommand: mvn clean install
additionalPlanContext: |
  Ensure all JNI native libraries are compatible with ARM64.
  Validate against our integration test suite.
```

### 총평 ###
순수 Java 코드는 JVM 위에서 돌아가기 때문에 x86이든 ARM64든 바이트코드 레벨에서 이미 플랫폼 독립적이다. 
이 transform이 실질적으로 의미 있는 경우는 굉장히 제한적이다.
```
JNI를 통해 네이티브 라이브러리(.so)를 직접 로딩하는 경우
네이티브 바인딩이 포함된 의존성을 쓰는 경우 (Netty epoll, RocksDB, LevelDB, SQLite JDBC 등)
Dockerfile이나 빌드 스크립트에 x86 아키텍처가 하드코딩된 경우
```

결국 이 도구의 본질은 "코드 변환"이라기보다는:

```
프로젝트 전체를 스캔해서 ARM64에서 문제될 수 있는 의존성을 찾아내고  --> JNI
해당 의존성의 ARM64 호환 버전으로 올려주고  ---> JNI
빌드/테스트를 돌려서 검증해주는   ---> 기존 유닛 테스트 있는 경우 실행, 없으면 패스
일종의 호환성 검증 + 의존성 업그레이드 자동화 도구에 가깝습니다.
```

공식 문서에서도 "Many modern Java applications are already Arm64-compatible"이라고 직접 언급하고 있다. 즉, 대부분의 순수 Java 앱은 이 transform을 돌려도 변경 사항이 거의 없을 수 있다는 걸 AWS도 인정하는 셈이다.

```
실제로 가치가 있는 시나리오는 레거시 Java 프로젝트에서 오래된 네이티브 의존성을 많이 쓰고 있는데, 어떤 게 ARM64에서 문제가 되는지 일일이 파악하기 어려울 때 정도이다. 대규모 마이그레이션에서 수십~수백 개 서비스를 한꺼번에 Graviton으로 옮길 때 자동 스캔 용도로 쓰는 거라고 보면 적절하다.
```

"변환"이라는 이름이 좀 과한 것 같고, "검증 도구"가 더 정확한 표현일 듯 하다.

### 사용 방법 ###

#### 1. 설치하기 ####
```
# 1. CLI 설치
curl -fsSL https://transform-cli.awsstatic.com/install.sh | bash

# 2. 설치 확인
atx --version

# 3. AWS 인증 설정
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_REGION=us-east-1

# 4. 프로젝트가 Git repo여야 함
cd my-java-project
git init
git add .
git commit -m "Initial commit"
```

#### 2. 실행하기 ####
* non-intractive mode 
```
atx custom def exec \
  -n AWS/early-access-java-x86-to-graviton \
  -p ./my-java-project \
  -c "mvn clean install" \
  -x \
  -t
```
* -n : transformation 이름
* -p : 프로젝트 경로
* -c : 빌드 명령어
* -x : 비대화형 모드 (사람 개입 없이 자동 실행)
* -t : 모든 도구 자동 승인

#### 설정 파일 만들어서 실행 ####
[config.yaml]
```
codeRepositoryPath: ./my-java-project
transformationName: AWS/early-access-java-x86-to-graviton
buildCommand: mvn clean install
additionalPlanContext: |
  이 프로젝트는 Netty와 RocksDB를 사용합니다.
  모든 기존 테스트가 통과해야 합니다.
```
아래 명령어로 실행한다.
```
atx custom def exec --configuration file://config.yaml -x -t
```

[출력 샘플]
```
🔍 Analyzing codebase at ./my-java-project...
📋 Scanning dependencies for Arm64 compatibility...

Found 3 potential Arm64 incompatibilities:

  1. io.netty:netty-transport-native-epoll:4.1.68.Final
     - classifier: linux-x86_64 (x86 only)
     → Updating to 4.1.100.Final with os-detected classifier

  2. org.rocksdb:rocksdbjni:6.22.1
     - Known Arm64 incompatibility in this version
     → Updating to 8.5.3 (Arm64 supported)

  3. src/main/java/com/example/NativeLoader.java:15
     - Hardcoded x86_64 library path detected
     → Adding aarch64 architecture detection

🔧 Applying changes...
  ✅ Updated pom.xml (2 dependency changes)
  ✅ Modified NativeLoader.java (architecture detection added)

🏗️ Running build command: mvn clean install
  ✅ BUILD SUCCESS

📊 Transformation Summary:
  - Files modified: 2
  - Dependencies updated: 2
  - Code patterns fixed: 1
  - Build: PASSED
  - Tests: 14/14 PASSED

Agent minutes used: 8.25
```

#### 3. 결과확인 ####
변환 완료 후 Git으로 변경 사항을 확인할 수 있다.
```
# 변경된 파일 확인
git status

# 변경 내역 상세 확인
git diff HEAD~1

# 커밋 로그 확인
git log --oneline
```

AWS Transform custom은 변환 작업을 수행할 때 중간 단계마다 Git 커밋을 자동으로 만든다.
그래서 transform 실행 전에 프로젝트가 반드시 Git repo여야 한다. (git init 필수).

동작 방식을 정리하면:
```
transform 실행 전 현재 상태가 Git에 커밋되어 있어야 함
transform이 변경을 가할 때마다 중간 커밋을 자동 생성
완료 후 git diff <원래 커밋 ID>로 전체 변경 사항을 한눈에 볼 수 있음
```
즉, transform이 뭘 바꿨는지 추적하고, 마음에 안 들면 git revert로 되돌릴 수 있게 하려는 목적이다. 변환 도구가 코드를 직접 건드리니까, Git을 일종의 안전장치로 쓰는 것이다.
