# 1. 도커 컴포즈를 통한 컨테이너 라이프사이클 관리

- **스크립트 분리**: 배포 관련 스크립트는 별도의 코드베이스나 저장소에 모아두는 것이 좋다.
- **이름 규칙**: `service`, `image`, `container_name`을 동일한 이름(예: `catalog-service`)으로 설정한다.
- **스레드 수 설정**: 패키토 빌드팩 환경 변수 `BPL_JVM_THREAD_COUNT=50`을 사용하여 메모리 계산을 위한 스레드 수를 설정한다.
- **네트워킹**: 도커 컴포즈는 기본적으로 두 개의 컨테이너를 같은 네트워크에 연결하므로 명시적으로 네트워크를 지정할 필요가 없다.

```
version: "3.8"
services:

  # Applications

  catalog-service:
    depends_on:
      - polar-postgres
    image: "catalog-service"
    container_name: "catalog-service"
    ports:
      - 9001:9001
      - 8001:8001
    environment:
      # Buildpacks environment variable to configure the number of threads in memory calculation
      - BPL_JVM_THREAD_COUNT=50
      # Buildpacks environment variable to enable debug through a socket on port 8001
      - BPL_DEBUG_ENABLED=true
      - BPL_DEBUG_PORT=8001
      - SPRING_CLOUD_CONFIG_URI=http://config-service:8888
      - SPRING_DATASOURCE_URL=jdbc:postgresql://polar-postgres:5432/polardb_catalog
      - SPRING_PROFILES_ACTIVE=testdata
  
  config-service:
    image: "config-service"
    container_name: "config-service"
    ports:
      - 8888:8888
      - 9888:9888
    environment:
      # Buildpacks environment variable to configure the number of threads in memory calculation
      - BPL_JVM_THREAD_COUNT=50
      # Buildpacks environment variable to enable debug through a socket on port 9888
      - BPL_DEBUG_ENABLED=true
      - BPL_DEBUG_PORT=9888

  # Backing Services

  polar-postgres:
    image: "postgres:14.12"
    container_name: "polar-postgres"
    ports:
      - 5432:5432
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=polardb_catalog

```

# 2. 스프링 부트 컨테이너 디버깅

- **컨테이너에서의 디버깅**: 컨테이너 내에서 실행될 때 로컬 컴퓨터에서 프로세스가 실행되지 않으므로 IDE에서 직접 디버깅을 할 수 없다.
- **JVM 디버깅**: 컨테이너 내부의 JVM이 특정 포트에서 디버그 연결을 대기하도록 설정해야 한다.
- **패키토 빌드팩 디버깅**: 패키토 빌드팩이 생성한 이미지에서는 디버그 모드를 위한 환경 변수를 지원한다:
  - `BPL_DEBUG_ENABLED`
  - `BPL_DEBUG_PORT`
- **디버그 설정**: IDE에서 `RUN/Debug Configurations`를 통해 원격 JVM 디버깅을 설정할 수 있다.

# 3. 배포 파이프라인: 패키지 및 등록

- **아티팩트 생성**: 지속적 전달에서 중요한 원칙은 아티팩트를 한 번만 생성하는 것이다(릴리스 후보 빌드 시).
- **취약점 검사**: 코드베이스뿐만 아니라 최종 생성된 아티팩트(시스템 라이브러리 포함)도 취약점 검사를 해야 한다.
- **이미지 서명**: 이미지 서명을 통해 모든 이해관계자가 컨테이너 이미지가 손상되지 않고 유효하다는 것을 확인할 수 있다.

# 4. 깃허브 액션을 통한 컨테이너 이미지 등록

- **워크플로 정의**: 커밋 단계에서 컨테이너를 생성하기 위한 YAML 파일(`.github/workflows/commit-stage.yml`)에 필요한 환경 변수를 정의한다.
- **환경 변수**:
  - **조건부 실행**: `If: ${{ github.ref == 'refs/heads/main' }}`를 사용하여 잡이 `main` 브랜치에서만 실행되도록 한다.
  - **권한 설정**:
    - `Permissions.contents: read`는 저장소를 체크아웃하기 위한 권한을 부여한다.
    - `Permissions.packages: write`는 이미지를 깃허브 컨테이너 레지스트리에 업로드하기 위한 권한을 부여한다.
- **보안 및 스캔**:
  - **Anchore/scan-action@v3**: 취약점 스캔을 위한 액션.
  - **도커 로그인**: `docker/login-action@v2`를 사용하여 `REGISTRY`, `github.actor`, `secrets.GITHUB_TOKEN` 같은 환경 변수를 사용해 레지스트리에 인증.
  - **보고서**: 취약점 스캔 결과는 깃허브 저장소의 '보안' 섹션에서 확인할 수 있다.
  - **공개 저장소**: 취약점 보고서는 깃허브 저장소가 공개 상태여야만 업로드 가능하며, 엔터프라이즈 구독 시 비공개 저장소도 지원된다.
 
```
name: Commit Stage
on: push

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: lcs86-dev/catalog-service
  VERSION: latest

jobs:
  build:
    name: Build and Test
    runs-on: ubuntu-22.04
    permissions:
      contents: read
      security-events: write
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
          cache: gradle
      - name: Build, unit tests and integration tests
        run: |
          chmod +x gradlew
          ./gradlew build
      - name: Code vulnerability scanning
        uses: anchore/scan-action@v3
        id: scan
        with:
          path: "${{ github.workspace }}"
          fail-build: false
          severity-cutoff: high
      - name: Upload vulnerability report
        uses: github/codeql-action/upload-sarif@v3
        if: success() || failure()
        with:
          sarif_file: ${{ steps.scan.outputs.sarif }}
  package:
    name: Package and Publish
    if: ${{ github.ref == 'refs/heads/main' }}
    needs: [ build ]
    runs-on: ubuntu-22.04
    permissions:
      contents: read
      packages: write
      security-events: write
    steps:
      - name: Checkout source code
        uses: actions/checkout@v4
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17
          cache: gradle
      - name: Build container image
        run: |
          chmod +x gradlew
          ./gradlew bootBuildImage \
            --imageName ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.VERSION }}
      - name: OCI image vulnerability scanning
        uses: anchore/scan-action@v3
        id: scan
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.VERSION }}
          fail-build: false
          severity-cutoff: high
      - name: Upload vulnerability report
        uses: github/codeql-action/upload-sarif@v3
        if: success() || failure()
        with:
          sarif_file: ${{ steps.scan.outputs.sarif }}
      - name: Log into container registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GH_TOKEN }}
      - name: Publish container image
        run: docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ env.VERSION }}

```
