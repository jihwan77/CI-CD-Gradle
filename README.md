# 🚀 Jenkins CI/CD

## 🎯 프로젝트 목적

- **GitHub에 코드를 push**하면 **Jenkins가 자동으로 빌드**하고 **JAR 파일**을 생성  
- JAR 파일은 **Ubuntu 호스트와 Docker 컨테이너 간 Bind Mount**로 공유  
- **파일 변경을 감지하는 쉘 스크립트**를 사용하여 JAR 파일이 갱신되면 즉시 실행
- 빌드부터 서비스 실행까지 **자동화**되어, **최신 애플리케이션을 즉시 배포**할 수 있는 환경 구축

---

## 🌠 진행 과정

1. **Spring Application** 코드 작성  
2. **GitHub에 push**  
3. **GitHub Webhook**이 **Jenkins를 자동으로 트리거**  
4. Jenkins는 저장소에서 코드를 가져와 빌드하여 **JAR 파일**을 생성  
5. 빌드된 JAR 파일은 **Bind Mount**를 통해 **호스트 디렉토리**에 저장   
6. **파일 변경을 감지하는 쉘 스크립트**가 실행되며, JAR 파일이 변경되면 **자동으로 실행**  

---

## 👥 팀 구성원
<table>
  <tr>
    <td align="center">
      <a href="https://github.com/Minkyoungg0">
        <img src="https://github.com/Minkyoungg0.png" width="100px;" alt="Minkyoungg0"/><br />
        <sub><b>문민경</b></sub>
      </a>
    </td>
    <td align="center">
       <a href="https://github.com/hyunn522">
        <img src="https://github.com/hyunn522.png" width="100px;" alt="hyunn522"/><br />
        <sub><b>정서현</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/moonstone0514">
        <img src="https://github.com/moonstone0514.png" width="100px;" alt="moonstone0514"/><br />
        <sub><b>김문석</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/jihwan77">
        <img src="https://github.com/jihwan77.png" width="100px;" alt="jihwan77"/><br />
        <sub><b>황지환</b></sub>
      </a>
    </td>
    <td align="center">
      <a href="https://github.com/dlacowns21">
        <img src="https://github.com/dlacowns21.png" width="100px;" alt="dlacowns21"/><br />
        <sub><b>임채준</b></sub>
      </a>
    </td>
  </tr>
</table>

---

## 1️⃣ Spring Application 생성 (Gradle)

### a. 예시 코드
```java
@RestController
@RequestMapping("step04")
public class Controller {

    @GetMapping("/get")
    public String getReqRes() {
        return "get 방식 요청의 응답 데이터";
    }

    @PostMapping("/post")
    public String getReqRes2() {
        return "post 방식 요청의 응답 데이터";
    }
}
```

### b. 빌드 & 로컬 실행
```bash
./gradlew clean build

# 실행 테스트
java -jar build/libs/*-SNAPSHOT.jar
curl "http://localhost:8080/app/get"
```

---

## 2️⃣ Jenkins 컨테이너 생성

### a. 호스트 폴더 생성
```bash
# mount할 호스트 디렉토리 생성
mkdir /home/vboxuser/hostvol

# 빌드된 jar 복사
cp build/libs/*.jar ~/hostvol/app.jar
```

### b. Docker로 Jenkins 실행

#### 📌 Bind Mount 개념
> 호스트 디렉토리를 컨테이너 내부 경로에 직접 연결하는 방식  
> 호스트와 컨테이너가 **양방향으로 동일한 디렉토리를 공유**

- Jenkins 컨테이너를 Ubuntu와 **Bind Mount**하려면, 컨테이너 실행 시 `-v` 옵션을 반드시 지정해야 한다.

```bash
docker run --name myjenkins   -v /home/vboxuser/hostvol:/var/jenkins_home/workspace/   -d -p 8080:8080   jenkins/jenkins:lts-jdk17
```

⚠️ 이미 실행 중인 Jenkins 컨테이너에는 새로운 마운트를 추가할 수 없음.  
→ `docker commit`으로 새 이미지를 만든 뒤 실행해야 함.

```bash
# 실행 중인 컨테이너로부터 이미지 생성
docker commit myjenkins myjenkinsimg

# 이미지 확인
docker images

# 새로운 컨테이너 실행
docker run --name myjenkins   -v /home/vboxuser/hostvol:/var/jenkins_home/workspace/   -d -p 8080:8080   myjenkinsimg:latest
```

---

## 3️⃣ Jenkins Pipeline 생성

```groovy
pipeline {
    agent any

    environment {
        GITHUB_REPO = 'https://github.com/hyunn522/Jenkins_Build.git'
        BRANCH_NAME = 'main'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${BRANCH_NAME}", url: "${GITHUB_REPO}"
                sh 'ls -al'
            }
        }

        stage('Build') {
            steps {
                script {
                    if (fileExists('gradlew')) {
                        sh 'chmod +x gradlew'
                        sh './gradlew build'
                    } else if (fileExists('pom.xml')) {
                        sh 'mvn clean package'
                    } else {
                        error 'Gradle 또는 Maven 프로젝트가 아님'
                    }
                }
            }
        }
    }

    post {
        success { echo '✅ 빌드 성공!' }
        failure { echo '❌ 빌드 실패! 오류 확인 필요!' }
    }
}
```
> 빌드된 JAR : `build/libs/*.jar`  

---

## 4️⃣ GitHub Webhook 설정

### 🔔 Webhook이란?
> GitHub push 등 특정 이벤트 발생 시, 지정된 URL로 **자동 HTTP 요청을 보내는 방식**  
> 주기적 확인 없이 실시간 자동화가 가능하다.


### a. GitHub 저장소 Webhook 등록
<img width="1853" alt="webhook" src="https://github.com/user-attachments/assets/8e9d48b9-d5d2-4fcd-9349-8f9773b4ceeb" />

### b. ngrok을 통한 외부 노출

#### 📌 ngrok 개념
> - **ngrok**은 로컬 서버(예: `http://localhost:8080`)를  
>   외부에서 접근 가능한 **공개 URL**로 매핑해주는 터널링 도구   
> - 방화벽/공유기 설정 없이도 로컬 서버를 인터넷에 노출 가능  

---

### 1. ngrok 실행
```bash
ngrok http 8080

# ngrok http <특정 IP>:<특정 포트>
# 로컬호스트(127.0.0.1) IP를 지정해 외부 공개 가능
```

- Jenkins의 8080 포트를 외부로 노출  
- 실행 후 터미널에 발급된 URL 확인 가능  

<img width="500" alt="ngrok1" src="https://github.com/user-attachments/assets/b40fcbc0-e000-4de5-b208-0da3aa5e7b68" />  <br>
➡ ngrok 실행 후, Jenkins 8080 포트가 외부 주소로 매핑된 화면  

---

### 2. ngrok Public URL 확인
- `https://<랜덤ID>.ngrok.io` 형태의 주소가 발급됨  
- 이 URL을 **GitHub Webhook Payload URL**에 등록  

<img width="500" alt="ngrok2" src="https://github.com/user-attachments/assets/9d805dd0-f40d-4ae8-84bf-af0cd5a304a7" />  <br>
➡ ngrok에서 발급한 외부 접근용 URL 확인  

---

### 3. 브라우저 접속 확인
- 발급받은 ngrok 주소로 접속하면 Jenkins 로그인 화면이 나옴  
- 외부에서도 정상 접근 가능한지 확인 후 Webhook에 등록  

<img width="500" alt="ngrok3" src="https://github.com/user-attachments/assets/28473f4a-205e-458e-83e3-1899a493a3ad" />  <br>
➡ ngrok URL을 통해 Jenkins 로그인 페이지에 접속된 모습  

---

## 5️⃣ 자동화 환경 구성


### inotifywait 스크립트
```bash
#!/bin/bash

item_name=$1
if [ -z "$item_name" ]; then
    echo "❌ 사용법: $0 <Jenkins 아이템 이름>"
    exit 1
fi

dir_path="/home/vboxuser/hostvol/${item_name}"
if [ ! -d "$dir_path" ]; then
    echo "❌ 디렉토리가 존재하지 않습니다: $dir_path"
    exit 1
fi

inotifywait -m -r -e create,close_write --format '%e %w%f' "$dir_path" |
while read -r events file; do
    if [[ "$file" == *.jar && "$file" != *plain*.jar ]]; then
        echo "🔔 JAR 감지됨: $file"

        pkill -f "java -jar" 2>/dev/null
        sleep 2

        latest_jar=$(ls -t "$dir_path"/build/libs/*.jar | grep -v plain | head -n 1)
        echo "🚀 실행 시작: $latest_jar"
        nohup java -jar "$latest_jar" > "${latest_jar%.jar}.log" 2>&1 &
    fi
done
```

이 스크립트를 통해 **Jenkins 빌드 → JAR 파일 갱신 → 자동 실행**이 완성된다.  
즉, 코드를 **GitHub에 push**하는 순간부터 애플리케이션이 자동으로 **빌드·실행**된다.

---

## 6️⃣ 네트워크 & 서버 통신

단일 서버에서 동작하던 환경을 확장하여,  
**빌드 서버(Jenkins)와 실행 서버(WAS)를 분리한 멀티 서버 구조**로 발전시켰다.  

<img width="6448" height="3080" alt="image (5)" src="https://github.com/user-attachments/assets/67fddedd-5684-4491-a6f4-1aae4e178396" />


구축 과정은 **서버 분리 → 네트워크 및 IP 설정 → SSH key 생성 및 추가 → 자동화 스크립트 적용** 단계를 통해 진행되었다.

---

### 🔹 1단계: 네트워크 전환 (NAT → 브리지)

- VirtualBox 기본 네트워크 모드는 **NAT**라서, 외부(다른 PC)에서 VM에 직접 접근하기 어렵다.

> 💡 NAT 모드에서는 VM이 호스트를 통해서만 외부로 나갈 수 있고,  
> 외부(다른 PC)에서 VM으로 직접 들어오는 연결은 차단된다.

- 따라서 **브리지 네트워크**로 전환하여, VM이 로컬 네트워크의 독립 서버처럼 동작하도록 구성.  
- 이를 통해 **다른 노트북의 Jenkins 서버 ↔ WAS 서버 간 직접 통신**이 가능해졌다.

<img width="600" height="300" alt="image (6)" src="https://github.com/user-attachments/assets/8d9d7bbc-a6a8-47a5-b587-a8ff0d34a5ec" />

---

### 🔹 2단계: IP 설정 (netplan)

- Ubuntu에서 네트워크 설정은 `netplan`을 통해 관리됨.  
- 설정 파일을 수정하여 **DHCP 모드**로 IP를 자동 할당받도록 구성.  
- 고정된 사설 IP를 확보해 서버 간 안정적인 통신 기반을 마련.

```bash
sudo vi /etc/netplan/01-netcfg.yaml
```

```yaml
# jenkins server
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:  # 실제 인터페이스 이름으로 변경
      dhcp4: yes  # DHCP 활성화
      #addresses:
      # - 192.168.0.100/24  # 원하는 고정 IP
      # routes:
      # - to: default
      #   via: 192.168.0.1  # 게이트웨이
      # nameservers:
      #   addresses:
      #     - 8.8.8.8
      #     - 1.1.1.1
```

```bash
# ip 적용
sudo netplan apply

# 적용된 ip 확인
ip a
```

→ 위 내용을 jenkins server와 WAS server에 적용 (WAS server IP: 192.168.0.8)

---

### 🔹 3단계: SSH 키 기반 인증

- 서버 간 파일 전송이나 **원격 접속 시 매번 비밀번호 입력은 비효율적**
- **SSH Key Pair를 생성·교환**해 **비밀번호 없이 접속할 수 있도록 설정**
- 이를 통해 Jenkins에서 WAS로 **자동 파일 배포**가 가능해짐.

```bash
# SSH 키 쌍 생성 (Jenkins 서버에서 실행)
ssh-keygen -t rsa -b 4096

# 생성된 공개키를 WAS 서버에 등록
ssh-copy-id ubuntu@myserver02
```

---

### 🔹 4단계: 자동화 스크립트 적용

#### a) Jenkins → WAS 전송을 위한 스크립트 추가

> **check_modify_scp.sh** (jenkins server용)
```
#!/bin/bash

target_user="ubuntu"               # 파일 받을 VM의 사용자
target_host="192.168.0.8"             # 파일 받을 VM의 호스트/IP
target_dir="/home/ubuntu/hostvol"     # 받을 VM에 저장할 경로

if [ -z "$item_name" ]; then
    echo "❌ 사용법: $0 <Jenkins 아이템 이름>"
    exit 1
fi

dir_path="/home/vboxuser/hostvol/${item_name}"
if [ ! -d "$dir_path" ]; then
    echo "❌ 디렉토리가 존재하지 않습니다: $dir_path"
    exit 1
fi

(
  echo "📂 감시 시작: item_name=$item_name, dir_path=$dir_path"

  inotifywait -m -r -e create,close_write --format '%e %w%f' "$dir_path" |
  while read -r events file; do
      if [[ "$file" == *.jar && "$file" != *plain*.jar ]]; then
          echo "🔔 JAR 감지됨: $file"

          # 최신 JAR 찾기
          latest_jar=$(ls -t "$dir_path"/build/libs/*.jar | grep -v plain | head -n 1)
          echo "📤 파일 전송: $latest_jar → ${target_user}@${target_host}:${target_dir}/${item_name}"

          # scp 실행 (SSH 키 기반 무비번 전송 권장)
          scp "$latest_jar" "${target_user}@${target_host}:${target_dir}/${item_name}"
          if [ $? -eq 0 ]; then
              echo "✅ 전송 완료: $latest_jar"
          else
              echo "❌ 전송 실패: $latest_jar"
          fi
      fi
  done
)
disown
```

#### b) WAS 측 실행 워처 스크립트
- WAS로 받은 빌드 산출물(JAR)을 자동 실행시키기 위한 **감시/실행 스크립트**

> **check_modify.sh** (WAS 서버용)

```bash
#!/bin/bash

item_name=$1
if [ -z "$item_name" ]; then
    echo "❌ 사용법: $0 <Jenkins 아이템 이름>"
    exit 1
fi

dir_path="/home/ubuntu/hostvol/${item_name}"
if [ ! -d "$dir_path" ]; then
    echo "❌ 디렉토리가 존재하지 않습니다: $dir_path"
    mkdir -p "$dir_path"
fi

(
  echo "📂 감시 시작: item_name=$item_name, dir_path=$dir_path"

  inotifywait -m -r -e create,close_write --format '%e %w%f' "$dir_path" |
  while read -r events file; do
      if [[ "$file" == *.jar && "$file" != *plain*.jar ]]; then
          echo "🔔 JAR 감지됨: $file"

          pkill -f "java -jar" 2>/dev/null
          sleep 2

          latest_jar=$(ls -t "$dir_path"/*.jar | grep -v plain | head -n 1)
          echo "🚀 실행 시작: $latest_jar"
          nohup java -jar "$latest_jar" > "${latest_jar%.jar}.log" 2>&1 &
      fi
  done
) > watch-jar.log 2>&1 &
disown
```

```bash
# 실행 권한 부여
chmod 770 check_modify.sh
chmod 770 check_modify_scp.sh

# 감시 시작 (예: Jenkins 아이템 이름이 step03_teamArt 인 경우)
./check_modify.sh step03_teamArt
```

Jenkins에서 **빌드 → scp 전송**이 이뤄지면,  
WAS 서버에서 동작하는 **감시 스크립트**가 이를 감지하여 **기존 애플리케이션을 종료하고 최신 JAR로 재실행**한다.

---

## 🛠️ 트러블 슈팅

### 1. 현상
- `docker commit`으로 새 이미지를 만들었음  
- `docker run -v ...` 실행 시, 기존 `/var/jenkins_home/workspace/` 데이터가 보이지 않음  

### 2. 원인
- **바인드 마운트는 호스트 디렉토리를 최우선**으로 사용  
- 컨테이너 경로는 호스트 디렉토리로 완전히 덮어씌워짐  
- 결과적으로 **기존 데이터가 보이지 않는 것처럼 보이는 현상** 발생  

👉 실제로는 지워진 것이 아니라, **호스트 마운트에 가려져 접근할 수 없게 된 것**  

---

### 1. 현상 

* `.sh` 스크립트 실행 시 매개변수(`$1`, `$2` 등)를 입력했음  
* 하지만 실행 결과는 **항상 동일한 디렉토리 값**으로 처리됨  
* 매개변수가 반영되지 않고, 고정된 값으로 동작  


### 2. 원인

* 스크립트 내부에서 변수를 선언할 때, **이미 특정 디렉토리 값으로 초기화**해둠  
* 이후 매개변수를 읽더라도, **초기값에 의해 덮어씌워져 무시되는 현상** 발생  
* 즉, **매개변수를 사용하지 못하고 하드코딩된 값만 사용되는 문제**  

👉 해결 방법은 변수 선언 시 매개변수를 우선 적용하고, 기본값은 `${}` 구문으로 조건부 처리해야 함.  
