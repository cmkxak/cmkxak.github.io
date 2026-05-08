---
title: "[DevOps] 젠킨스로 CI/CD 구축하기"
excerpt: "GitHub에 push 한 번이면 빌드, 테스트, Docker 이미지 생성, 원격 배포, Slack 알림까지. 제가 실제로 구축한 Jenkins CI/CD 흐름을 처음부터 차근히 풀어볼게요."

toc: true
toc_sticky: true

date: 2026-05-08
last_modified_at: 2026-05-08

categories:
  - DevOps
tags:
  - Jenkins
  - CI/CD
  - Pipeline
  - Docker
  - GitHub
  - Spring Boot
---

안녕하세요. 결제 시스템을 만들고 운영하는 백엔드 엔지니어 곽철민이에요.

새 프로젝트에 합류하면 코드 작성보다 먼저 부딪히는 일이 있어요. 바로 *“이 프로젝트의 CI/CD 환경을 어떻게 가져갈까”* 라는 질문이죠. 매일 손으로 빌드해서 jar 올리고, 서버 들어가서 재시작하던 시절이 있었는데요. 그 시절을 지나오면서 자연스럽게 든 생각이 있어요. 자동화는 결국 *사람의 실수가 일어나는 시간*을 줄이는 일이라는 거였어요.

이 글에서는 제가 직접 구축하고 운영하면서 정리한, Jenkins 기반 CI/CD 파이프라인을 차근히 풀어볼게요. Spring Boot 프로젝트를 예시로, GitHub에 push가 들어오면 → 빌드와 테스트가 돌고 → Docker 이미지가 만들어져 → 원격 운영 서버에 배포되고 → Slack 알림이 가는 흐름까지요.

## 그 전에, CI/CD가 뭔가요

저도 처음엔 약자만 보고 한참 헤맸어요. 풀어보면 이런 뜻이에요.

- **CI (Continuous Integration, 지속적 통합)** — 코드를 push하면 *자동으로* 빌드하고 테스트해서, 합쳐지는 코드의 품질을 지켜주는 일이에요.
- **CD (Continuous Delivery / Deployment, 지속적 배포)** — 검증된 결과물을 *자동으로* 운영 직전이나 운영 환경까지 올려주는 일이고요.

수동 배포를 오래 해보면 알게 돼요. 새벽 3시에 jar 잘못 올리는 순간, 그날 아침의 출근길이 어떤지요. 결국 *자동화 = 안정성* 이라는 말을, 한 번쯤 호되게 겪고 나서야 진심으로 받아들이게 되는 것 같아요.

> Jenkins는 이 자동화를 도와주는 **오픈소스 CI/CD 서버**예요. 1,800개가 넘는 플러그인 덕에 GitHub, GitLab, Bitbucket 같은 어지간한 도구는 모두 연결돼요.

---

## 1. Jenkins 설치 — Docker로 시작하는 게 가장 깔끔해요

설치 방법은 여러 가지인데요. war 파일을 직접 띄우거나, brew·apt로 설치할 수도 있지만, 저는 로컬이든 운영이든 **Docker로 띄우는 방식**을 가장 자주 써요. 환경이 격리되고, 옮길 때도 부담이 없거든요.

### 데이터 보관용 디렉터리부터 준비할게요

```bash
mkdir -p ~/jenkins-data && chmod 777 ~/jenkins-data
```

### 컨테이너 띄우기

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v ~/jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --restart unless-stopped \
  jenkins/jenkins:lts-jdk17
```

옵션이 좀 있는데, 각각이 어떤 의미인지 정리해볼게요.

| 옵션 | 의미 |
|------|------|
| `-p 8080:8080` | Jenkins 웹 UI 접속용 포트예요. |
| `-p 50000:50000` | 분산 빌드(Agent) 통신 포트고요. |
| `-v ~/jenkins-data:/var/jenkins_home` | 컨테이너를 지워도 잡과 설정이 그대로 살아남게 해줘요. 처음 세팅할 때 빼먹기 쉬운데, 이게 빠지면 나중에 진짜 곤란해져요. |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Jenkins 안에서 호스트의 Docker를 쓰려고 마운트하는 거예요. 뒤에서 Docker 빌드할 때 필요해요. |
| `lts-jdk17` | LTS 버전에 JDK 17이 들어 있어, Spring Boot 3.x와 잘 맞아요. |

### 초기 비밀번호를 한 번 꺼내야 해요

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

이 비밀번호로 `http://localhost:8080`에 들어가서, **Install suggested plugins**를 누르고 관리자 계정을 만들면 첫 단계는 끝이에요.

> 운영 환경이라면 nginx 리버스 프록시로 HTTPS를 꼭 씌워주세요. 평문 통신은 한 번이라도 외부에 노출되면 위험할 수 있어요.

---

## 2. 핵심 개념을 짧게 짚고 갈게요

처음 Jenkins를 만지면 용어가 살짝 많게 느껴져요. 그래도 다섯 개 정도만 잡고 들어가면 충분해요.

| 용어 | 한 줄 설명 |
|------|-----------|
| **Job (= Project)** | 자동화하고 싶은 작업의 단위예요. 빌드, 테스트, 배포 모두 Job이에요. |
| **Build** | Job이 한 번 실행된 결과예요. #1, #2처럼 번호가 붙어요. |
| **Workspace** | Job이 실제로 작업하는 디렉터리예요. |
| **Pipeline** | Job을 *코드*로 정의한 거예요. `Jenkinsfile`을 써요. |
| **Stage / Step** | Pipeline 안의 큰 단계(Stage)와, 그 안의 개별 명령(Step)이에요. |
| **Agent** | Job이 실제로 돌아가는 머신이에요. master 외에 별도 노드를 두기도 해요. |
| **Credentials** | SSH 키, 토큰, 비밀번호처럼 민감한 정보를 담아두는 보관함이에요. |

처음 시작은 GUI에서 클릭으로 만드는 Freestyle Job이 쉽긴 한데요. **운영 환경에서는 Pipeline을 코드로 정의하는 방식이 표준**이에요. 코드로 버전 관리되고, 코드 리뷰가 가능하고, 무엇보다 *무엇이 어떻게 돌아가고 있는지가 한 화면에 잡혀요.* 처음엔 살짝 진입 장벽이 있지만, 한 번 익숙해지면 다시는 GUI로 돌아가고 싶지 않더라고요.

---

## 3. Freestyle Job — 일단 한 번 손에 익혀볼까요

가볍게 *“Jenkins가 내 명령을 실행해주는 느낌”* 만 빠르게 보고 갈게요.

> 새 Item → Freestyle project → 이름은 `hello-jenkins`로 할게요.

### Build steps → Execute shell

```bash
echo "Hello, Jenkins!"
echo "Build #${BUILD_NUMBER} on $(date)"
```

저장하고 **Build Now**를 누르면 좌하단에 #1 빌드가 떠요. 이 한 줄이 화면에 찍히는 순간이 사실 꽤 보람 있어요.

이 정도는 5분이면 끝나는데요. 실무에서는 빌드 → 테스트 → 도커 이미지 → 원격 배포 → 알림이 한 번에 이어져야 해서, 자연스럽게 다음 단계인 Pipeline으로 넘어가게 돼요.

---

## 4. Declarative Pipeline의 기본 구조

`Jenkinsfile` 한 장이면 빌드부터 배포까지 한 흐름으로 정의할 수 있어요. 가장 작은 형태부터 같이 보고 갈게요.

```groovy
pipeline {
    agent any                    // 어디서 돌릴지 (any = 아무 노드)

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()                                   // 콘솔에 시간 표시
        buildDiscarder(logRotator(numToKeepStr: '20'))  // 빌드 20개만 보관
    }

    environment {
        APP_NAME = 'mytongjang-pay'
        REGISTRY = 'registry.example.com'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps { sh './gradlew clean build -x test' }
        }

        stage('Test') {
            steps { sh './gradlew test' }
            post {
                always { junit 'build/test-results/test/*.xml' }
            }
        }
    }

    post {
        success { echo '✅ build succeeded' }
        failure { echo '❌ build failed' }
        always  { cleanWs() }   // 워크스페이스 정리
    }
}
```

블록을 하나씩 짚어보면 이해가 쉬워요.

- `agent` — 빌드를 어디서 돌릴지 정해요. `any`, `none`, `label 'docker'`, `docker { image '...' }`처럼 다양하게 쓸 수 있어요.
- `environment` — 모든 stage가 공유하는 환경 변수예요.
- `stages > stage > steps` — 실제 실행 단위이고요.
- `post` — 빌드가 끝난 뒤의 후처리예요. `always`, `success`, `failure` 같은 조건으로 분기할 수 있어요.

여기까지가 Pipeline의 뼈대예요. 이제 실제 프로젝트에 붙여볼게요.

---

## 5. GitHub 연동 — push 한 번에 빌드가 도는 환경 만들기

저는 이 단계가 늘 가장 신나는 순간이에요. push 했더니 노트북에서 알림이 뜨고, 빌드 화면에 새 #이 찍히는 그 흐름이요.

### 5-1. GitHub 자격 증명 등록

**Manage Jenkins → Credentials → System → Global → Add Credentials**

- Kind: **Username with password** (또는 SSH Username with private key)
- Username: GitHub 사용자명
- Password: **Personal Access Token (PAT)** — repo 권한 있는 토큰으로 만들어주세요.
- ID: `github-cred`

### 5-2. Multibranch Pipeline 만들기

브랜치마다 자동으로 Job이 생기는 형태예요. 실무에서 가장 자주 쓰는 패턴이고요.

> New Item → **Multibranch Pipeline** → 이름은 `mytongjang-pay`로 할게요.

- Branch Sources → Add source → **GitHub**
  - Credentials: `github-cred`
  - Repository HTTPS URL: `https://github.com/your-org/mytongjang-pay`
- Build Configuration: by Jenkinsfile (기본값 그대로 두면 돼요)
- Scan Multibranch Pipeline Triggers: 1분 주기로 두거나, 뒤에 만들 Webhook을 사용해요.

### 5-3. GitHub Webhook 등록

GitHub Repo → Settings → Webhooks → Add webhook으로 들어가주세요.

- Payload URL: `https://jenkins.example.com/github-webhook/`
- Content type: `application/json`
- Events: **Just the push event** — PR도 자동 빌드하고 싶으면 PR event를 추가하면 돼요.

이러면 push 한 번에 Jenkins가 알아채고, 해당 브랜치의 Job을 알아서 돌려줘요. 처음 이 흐름이 동작하는 걸 보는 순간이 사실 가장 보람 있어요.

---

## 6. 실전 예제 — Spring Boot + Docker + 원격 서버 배포

여기서부터가 진짜 본론이에요. 시나리오부터 그려볼게요.

1. `main` 브랜치에 push가 들어오면
2. Jenkins가 Gradle 빌드와 테스트를 돌리고
3. 통과하면 Docker 이미지를 만들어 사내 레지스트리로 push해요.
4. 운영 서버에 SSH로 접속해서, 새 이미지로 컨테이너를 다시 띄우고요.
5. 마지막으로 결과를 Slack 채널에 알려요.

### 6-1. Jenkinsfile 전체

```groovy
pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    environment {
        APP_NAME    = 'mytongjang-pay'
        REGISTRY    = 'registry.example.com'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        IMAGE       = "${REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
        DEPLOY_HOST = 'deploy@10.0.0.21'
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Test') {
            steps { sh './gradlew clean build' }
            post {
                always { junit 'build/test-results/test/*.xml' }
            }
        }

        stage('Docker Build & Push') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry',
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sh '''
                        echo "$REG_PASS" | docker login $REGISTRY -u "$REG_USER" --password-stdin
                        docker build -t $IMAGE -t $REGISTRY/$APP_NAME:latest .
                        docker push $IMAGE
                        docker push $REGISTRY/$APP_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sshagent (credentials: ['deploy-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no $DEPLOY_HOST "
                            docker pull $IMAGE &&
                            docker stop $APP_NAME 2>/dev/null || true &&
                            docker rm   $APP_NAME 2>/dev/null || true &&
                            docker run -d \\
                                --name $APP_NAME \\
                                --restart unless-stopped \\
                                -p 8080:8080 \\
                                -e SPRING_PROFILES_ACTIVE=prod \\
                                $IMAGE
                        "
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend(
                color: 'good',
                message: "✅ ${APP_NAME} #${BUILD_NUMBER} 배포 성공\n${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "❌ ${APP_NAME} #${BUILD_NUMBER} 빌드/배포 실패\n${env.BUILD_URL}"
            )
        }
        always { cleanWs() }
    }
}
```

### 6-2. 멀티스테이지 Dockerfile

저는 빌드는 빌드 스테이지에서, 실행 이미지는 가볍게 가져가는 멀티스테이지 방식을 선호해요. 운영 이미지에 빌드 도구가 같이 들어 있을 이유가 없거든요.

```dockerfile
# ── Build stage ──
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /workspace
COPY gradle ./gradle
COPY gradlew settings.gradle build.gradle ./
COPY src ./src
RUN chmod +x ./gradlew && ./gradlew clean bootJar -x test

# ── Run stage ──
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=builder /workspace/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-jar", "/app/app.jar"]
```

### 6-3. 필요한 Credentials를 정리하면 이래요

| ID | Kind | 어디에 쓰이나요 |
|----|------|------|
| `github-cred` | Username + PAT | GitHub repo 체크아웃 |
| `docker-registry` | Username/Password | 사내 레지스트리 로그인 |
| `deploy-ssh-key` | SSH Username + Private Key | 운영 서버 SSH 접속 |
| `slack-token` | Secret text | Slack Bot Token |

> ⚠️ Jenkinsfile에 비밀번호를 직접 적는 건 위험해요. 가능한 모든 비밀 정보는 `withCredentials`나 `sshagent` 블록을 통해 주입해서 쓰는 걸 권해요.

---

## 7. 자주 쓰는 패턴들

운영하다 보면 자연스럽게 손에 익는 패턴들이 있어요. 그중에서 진짜 자주 쓰는 것 몇 가지만 정리해볼게요.

### 7-1. 병렬 실행으로 시간 줄이기

테스트, 정적 분석, 의존성 검사처럼 서로 독립적인 일은 굳이 줄 세울 필요가 없어요.

```groovy
stage('Quality Gates') {
    parallel {
        stage('Unit Test')        { steps { sh './gradlew test' } }
        stage('Static Analysis')  { steps { sh './gradlew sonarqube' } }
        stage('Dependency Check') { steps { sh './gradlew dependencyCheckAnalyze' } }
    }
}
```

빌드 시간이 거의 절반 가까이 줄어드는 경우도 많아서, 한 번 도입해두면 두고두고 효과를 봐요.

### 7-2. 운영 배포 직전, 사람의 OK 받기

자동화가 좋다고 해도, 운영 배포는 *눈으로 한 번 더 확인하고 싶은* 순간이 있어요.

```groovy
stage('Approve to Production') {
    when { branch 'main' }
    steps {
        timeout(time: 30, unit: 'MINUTES') {
            input message: '운영 배포를 진행할까요?', ok: '배포'
        }
    }
}
```

### 7-3. Slack 알림

Manage Jenkins → System → **Slack** 섹션에 워크스페이스와 토큰을 등록하고 나면, 한 줄로 알림을 보낼 수 있어요.

```groovy
slackSend channel: '#deploy', color: 'good', message: '배포 시작할게요.'
```

### 7-4. 환경별 분기 (운영 / 스테이징)

브랜치에 따라 다른 서버로 배포되도록 만들어두면 편해요.

```groovy
stage('Deploy') {
    steps {
        script {
            def host = (env.BRANCH_NAME == 'main') ?
                'deploy@prod-01' : 'deploy@stg-01'
            sh "ssh ${host} 'systemctl restart mytongjang-pay'"
        }
    }
}
```

### 7-5. 빌드 결과물 보관

이건 평소엔 잊고 지내다가, 롤백이 필요한 어느 순간 진짜 고마워하게 되는 옵션이에요.

```groovy
post {
    success {
        archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
    }
}
```

---

## 8. 막혔을 때 보는 트러블슈팅 체크리스트

저도 여기 적힌 항목들을 한 번씩은 다 겪어봤어요. 처음 만나면 당황스럽지만, 한 번씩 만나두면 다음에는 훨씬 빠르게 대응할 수 있어요.

| 증상 | 확인할 곳 |
|------|----------|
| Webhook이 와도 Job이 안 도네요 | GitHub Webhook 페이지의 *Recent Deliveries* 응답 코드를 먼저 확인해보세요. |
| `docker: command not found` | Jenkins 컨테이너 띄울 때 `/var/run/docker.sock`을 마운트했는지 확인해주세요. |
| `permission denied (sock)` | Jenkins 사용자가 docker 그룹 권한을 가지고 있는지 봐주세요. |
| SSH 배포 도중 멈춰요 | `ssh -o StrictHostKeyChecking=no`를 쓰거나, `~/.ssh/known_hosts`에 호스트를 미리 등록해주세요. |
| Gradle 캐시를 매번 새로 받아요 | `~/.gradle` 디렉터리를 볼륨으로 마운트하거나 Gradle 캐시 플러그인을 도입해보세요. |
| 빌드 로그가 너무 쌓여요 | `buildDiscarder(logRotator(numToKeepStr: '20'))`로 보관 갯수를 제한해주세요. |
| jenkins-data 권한 오류 | 호스트 디렉터리의 소유자/권한이 jenkins UID(1000)과 맞는지 확인해주세요. |

---

## 9. 운영하면서 알게 된 것들

마지막으로, 직접 한 시스템을 일정 기간 운영해보고 나서야 비로소 와닿게 된 몇 가지를 적어둘게요.

- **Jenkinsfile은 모듈화하는 편이 좋아요.** 공통 로직을 `vars/`에 Shared Library로 빼서 여러 프로젝트에서 재사용하면, 작은 회사에서도 효과가 꽤 커요.
- **Master(Controller)는 직접 빌드를 돌리지 않는 게 안전해요.** Agent를 따로 두고, 빌드 부하와 보안 영역을 분리하는 걸 권해요.
- **이미지 태그는 절대 `latest` 하나로 운영하지 마세요.** 빌드 번호나 커밋 SHA를 박아두면, 롤백할 때 그 한 줄이 정말 든든해져요.
- **백업은 평소에 챙겨두세요.** `~/jenkins-data` 디렉터리만 통째로 백업하면 Jenkins 전체가 복구돼요. 정기 스냅샷은 *언젠가의 자기 자신을 구해주는 일* 이라고 생각해요.
- **플러그인 업데이트는 신중히 하세요.** 호환성이 깨지는 경우가 의외로 자주 있어서, 테스트 환경에서 한 번 돌려보고 적용하는 걸 추천드려요.

---

## 마무리하며

CI/CD를 처음 구축할 땐 *이렇게까지 자동화가 필요한가* 싶을 때가 있어요. 그런데 한번 흐름이 자리 잡으면, 그 다음부터는 push 한 번에 모든 게 알아서 돌아가는 풍경이 너무 자연스럽게 느껴져요. 사람이 챙겨야 하는 일이 줄어드는 만큼, 진짜 중요한 일에 시간을 더 쓸 수 있게 되더라고요.

이 글이 첫 Jenkins 환경을 만드는 누군가의 시작점이 된다면 좋겠어요. 다음 글에서는 같은 흐름을 **GitHub Actions로 만들 때의 차이**, 그리고 **Kubernetes 환경으로 확장하는 방법**을 이어서 정리해볼게요.
