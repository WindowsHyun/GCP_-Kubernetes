### 0. Kubernetes Docker Build 사전 정의된 변수
```
DOCKER_NAME="my-app"
buildfileName="MyAPP"
GIT_REPOSITORIES="GCP_Kubernetes"
pwd_dir="$PWD"
RESET_GIT=false
```

### 1. Git Clone & Update
```
clone_or_update_repo() { 
    local repo_dir="$1" 
    local repo_url="$2" 
    cd "$pwd_dir" 
    if [ ! -d "$repo_dir" ]; then 
        git clone "$repo_url" 
    fi 
    cd "$repo_dir" 
    git reset --hard 
    git pull 
}

tidy_go_modules() { 
    go mod tidy 
    go mod vendor 
} 

build_dir="$pwd_dir/$GIT_REPOSITORIES" 
# RESET_GIT 을 선택시 해당 폴더 삭제
if [ "$RESET_GIT" == true ]; then
    echo "Folder Delete:$build_dir" 
    sudo rm -rf $build_dir 
fi 
clone_or_update_repo "$build_dir" "git@github.com:WindowsHyun/$GIT_REPOSITORIES.git"
```

### 2. Go Build
```
echo "|| Go Init ||"
tidy_go_modules

echo "|| Go Build ||"
CGO_ENABLED=0 go build -o "$build_dir/$DOCKER_NAME" -a -ldflags '-s' main.go

if [ $? -ne 0 ]; then
  echo "Go 빌드 실패"
  exit 1
fi
```

### 3-0. Dockerfile Example
- Dockerfile
```
FROM debian:buster-slim

ENV TZ=Asia/Seoul
RUN apt update && apt install -y --no-install-recommends tzdata ca-certificates
RUN update-ca-certificates
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN mkdir -p /workdir
WORKDIR /workdir
COPY MyAPP /workdir/MyAPP
COPY conf /tmp/conf
COPY /Docker/docker.sh /workdir/docker.sh

RUN chmod +x MyAPP
RUN mkdir -p logs
RUN chmod +x logs/
RUN chmod +x docker.sh

EXPOSE 9000

CMD [ "bash", "docker.sh"]
```

- docker.sh
```
#!/bin/sh
# 디렉토리와 임시 디렉토리를 배열로 정의
workdirs=("/workdir/conf")
tmpdirs=("/tmp/conf")
target_files=("conf.toml" "bytecode_conf.json")

# 반복해서 작업하는 함수 정의
process_directory() {
    local workdir="$1"
    local tmpdir="$2"

    # 폴더가 없으면 생성
    if [ ! -d "$workdir" ]; then
        mkdir -p "$workdir"
    fi

    # 폴더 내 파일 개수 확인
    file_count=$(find "$workdir" -maxdepth 1 -type f | wc -l)

    # 1. 폴더에 파일이 없는 경우
    if [ "$file_count" -eq 0 ]; then
        if [ -d "$tmpdir" ]; then
            cp -r "$tmpdir"/* "$workdir/"
            find "$workdir" -type f -name "*.go" -delete
            find "$workdir" -type f -name ".DS_Store" -delete
            echo "${workdir##*/} directory copied and cleaned up!"
        else
            echo "${workdir##*/} error: $tmpdir not found!"
        fi

    # 2. 폴더에 파일이 있는 경우
    else
        # 특정 파일 누락 확인 및 복사
        for filename in "${target_files[@]}"; do
            if [ ! -f "$workdir/$filename" ]; then
                if [ -f "$tmpdir/$filename" ]; then
                    cp "$tmpdir/$filename" "$workdir/"
                    echo "$filename copied to ${workdir##*/}"
                else
                    echo "Error: $filename not found in $tmpdir"
                fi
            fi
        done
        echo "${workdir##*/} directory exists. Checked for missing files."
    fi

     # 디렉토리 내용 출력 (쉼표 구분, 권한 표시)
    files=$(find "$workdir" -maxdepth 1 -type f -printf "%f (%M)\n")
    echo "Files in ${workdir##*/}: $(echo "$files" | tr '\n' ',' | sed 's/,$//')"
    echo "Total files: $(find "$workdir" -maxdepth 1 -type f | wc -l)"
}

# ConfigMap에 저장된 JSON 파일 복사
if [ -d "/configmap-volume" ]; then
  cp -r /configmap-volume/* /workdir/conf/
  echo "ConfigMap JSON files copied to /workdir/conf/"
fi

# 반복 작업 실행
for i in "${!workdirs[@]}"; do
    process_directory "${workdirs[i]}" "${tmpdirs[i]}"
done

if [ "$TZ" != "Asia/Seoul" ]; then
    ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
else
    TZ="Asia/Seoul"
    ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
fi

# MyApp를 직접 실행. 로그는 Docker 컨테이너의 로그로 직접 흐름.
./MyApp 2>&1 &
app_pid=$!

# MyApp 프로세스가 살아 있는지 체크
while kill -0 $app_pid 2> /dev/null; do
    sleep 1
done

# MyApp 프로세스가 종료되면 스크립트도 종료되어 컨테이너가 중지됨
echo "MyApp가 종료됨. 컨테이너를 종료합니다."
exit 1
```

### 3. Docker Build
```
cd Docker
docker build -t "us-west1-docker.pkg.dev/[자신이 설정한 리소스]/kubernetes/my-app:latest" -f Dockerfile ..
if [ $? -ne 0 ]; then
  echo "Docker 빌드 실패"
  exit 1
fi
```

### 4. Docker Push
```
echo "|| Docker Push ||" 
gcloud auth activate-service-account --key-file=/home/ubuntu/artifact_service.json
export GOOGLE_APPLICATION_CREDENTIALS=/home/ubuntu/artifact_service.json
docker push us-west1-docker.pkg.dev/[자신이 설정한 리소스]/kubernetes/my-app:latest
if [ $? -ne 0 ]; then 
    echo "Docker Push 실패" 
    exit 1 
fi 
```

### 5. Kubernetes 사전 정의된 파일 작업
* [Kubernetes Definition](KubernetesDefinition.md)

### 6. Kubernetes Docker Run 사전 정의 변수
```
PROJECT_ID="live"
CLUSTER_NAME="live-server"
REGION="asia-southeast1"
BUILDFILE_NAME="website"
DOCKER_ENV="TARGET=live;PRIVATEKEY=0x0000;TESTDAY=0"
BUILD_ARGS="--build-arg NEXT_GOOGLE_CLIENT_ID=apps.googleusercontent.com --build-arg NEXT_PUBLIC_API_SERVER=https://thisisserver.com"
HOST_URL="thisisserver.com"
```

### 7. Kubernetes Docker Run
```
# Google Cloud SDK 인증 정보 설정 (사전에 만들어진 ArtifactService Credential)
gcloud auth activate-service-account --key-file="/home/jenkins/jenkins_credential.json"

# kubectl 인증 정보 설정
gcloud container clusters get-credentials "${CLUSTER_NAME}" --region "${REGION}" --project "${PROJECT_ID}"

# 배포할 서버의 yaml 을 만들어야 한다.
# 1. 배포할 서버의 폴더를 만들어서 default.yaml을 복사한다.
BASE_FOLDER="/home/jenkins/default"
BUILD_FOLDER="/home/jenkins/kubernetes/$BUILDFILE_NAME"
mkdir -p "$BUILD_FOLDER"
if [ "$FRONT_BUILD" = true ]; then
	cp "$BASE_FOLDER/frontDefault.yaml" "$BUILD_FOLDER/default.yaml"
else
	cp "$BASE_FOLDER/default.yaml" "$BUILD_FOLDER/default.yaml"
    cp "$BASE_FOLDER/logstashDefault.yaml" "$BUILD_FOLDER/logstashDefault.yaml"
fi
cp "$BASE_FOLDER/configDefault.yaml" "$BUILD_FOLDER/configDefault.yaml"
cp "$BASE_FOLDER/hpaDefault.yaml" "$BUILD_FOLDER/hpaDefault.yaml"
cp "$BASE_FOLDER/ingressDefault.yaml" "$BUILD_FOLDER/ingressDefault.yaml"
cp "$BASE_FOLDER/lbDefault.yaml" "$BUILD_FOLDER/lbDefault.yaml"
```

### 7-1. Kubernetes Docker Run
```
# 2. DOCKER_ENV 데이터를 configDefault.yaml에 추가한다.
# 기존 DOCKER_ENV 변수 사용 또는 비어있을 경우 BUILD_ARGS에서 가져오기
if [[ -z "$DOCKER_ENV" ]]; then
	# --build-arg 를 제거하고 key=value 쌍만 남김
	pairs=$(echo "$BUILD_ARGS" | sed 's/--build-arg //g')
	# 공백을 ";"로 변경
	DOCKER_ENV=$(echo "$pairs" | tr ' ' ';')
fi

echo "-----------------------------------------"
echo "ENV:"$DOCKER_ENV
echo "-----------------------------------------"

# ENV를 configDefault에 넣어준다.
data=""
IFS=';' read -r -a pairs <<< "$DOCKER_ENV"
for pair in "${pairs[@]}"; do
  # key와 value를 '='로 분할
  IFS='=' read -r key value <<< "$pair"
  # key와 value를 따옴표로 감싸고 data 변수에 추가
  data="$data  $key: \"$value\"\n"
done
echo -e "$data" >> "$BUILD_FOLDER/configDefault.yaml"
sed -i "s/\[ContainerName\]/${BUILDFILE_NAME}/g" "$BUILD_FOLDER/configDefault.yaml"
```

### 7-2. Kubernetes Docker Run (ENV 설정)
```
# 3. default.yaml 을 수정한다.
vars=(${DOCKER_ENV//;/ })
# env 부분 생성
env_yaml=""
for var in "${vars[@]}"; do
  key=${var%%=*}
  value=${var#*=}
  env_yaml="${env_yaml}        - name: $key\n          valueFrom:\n            configMapKeyRef:\n              key: $key\n              name: [ContainerConfigName]\n"
done
sed -i "s/\[DefaultENV\]/$env_yaml/g" "$BUILD_FOLDER/default.yaml"
sed -i "s/\[ContainerName\]/${BUILDFILE_NAME}/g" "$BUILD_FOLDER/default.yaml"
sed -i "s/\[ContainerConfigName\]/${BUILDFILE_NAME}-config/g" "$BUILD_FOLDER/default.yaml"
DOCKER_IMAGE_CONTENT=$(cat "$BUILD_FOLDER/$BUILDFILE_NAME.docker")
DOCKER_IMAGE_CONTENT_ESCAPED=$(echo "$DOCKER_IMAGE_CONTENT" | sed 's/\//\\\//g')
sed -i "s/\[DockerImage\]/${DOCKER_IMAGE_CONTENT_ESCAPED}/g" "$BUILD_FOLDER/default.yaml"
```

### 7-3. Kubernetes Docker Run (수평, 수직 확장 설정)
```
# 4. hpaDefault.yaml 을 수정한다.
sed -i "s/\[ContainerName\]/${BUILDFILE_NAME}/g" "$BUILD_FOLDER/hpaDefault.yaml"
```

### 7-4. Kubernetes Docker Run (로드벨런스 설정)
```
# 5. lbDefault.yaml 을 수정한다.
if [ "$FRONT_BUILD" = true ]; then
	FRONT_PORT=80
	sed -i "s/\[ContainerPort\]/${FRONT_PORT}/g" "$BUILD_FOLDER/lbDefault.yaml"
else
	sed -i "s/\[ContainerPort\]/${SERVER_PORT}/g" "$BUILD_FOLDER/lbDefault.yaml"
fi
sed -i "s/\[ContainerTargetPort\]/${SERVER_PORT}/g" "$BUILD_FOLDER/lbDefault.yaml"
sed -i "s/\[ContainerName\]/${BUILDFILE_NAME}/g" "$BUILD_FOLDER/lbDefault.yaml"
```

### 7-5. Kubernetes Docker Run (ingress 설정 & logstash 설정)
```
# 6. ingressDefault.yaml 을 수정한다.
if [ "$FRONT_BUILD" = true ]; then
	FRONT_PORT=80
	sed -i "s/\[ContainerPort\]/${FRONT_PORT}/g" "$BUILD_FOLDER/ingressDefault.yaml"
else
	sed -i "s/\[ContainerPort\]/${SERVER_PORT}/g" "$BUILD_FOLDER/ingressDefault.yaml"
fi
sed -i "s/\[ContainerName\]/${BUILDFILE_NAME}/g" "$BUILD_FOLDER/ingressDefault.yaml"
sed -i "s/\[ContainerHost\]/${HOST_URL}/g" "$BUILD_FOLDER/ingressDefault.yaml"

# 7. logstashDefault.yaml 을 수정한다.
if [ "$FRONT_BUILD" = false ]; then
	sed -i "s/\[LogName\]/${LOG_FILE_NAME}/g" "$BUILD_FOLDER/logstashDefault.yaml"
	sed -i "s/\[ContainerName\]/${BUILDFILE_NAME}/g" "$BUILD_FOLDER/logstashDefault.yaml"
fi
```

### 7-6. Kubernetes Docker Run (배포)
```
# configDefault 부터 진행
kubectl apply -f "$BUILD_FOLDER/configDefault.yaml"
kubectl apply -f "$BUILD_FOLDER/default.yaml"
kubectl apply -f "$BUILD_FOLDER/hpaDefault.yaml"
kubectl apply -f "$BUILD_FOLDER/lbDefault.yaml"
kubectl apply -f "$BUILD_FOLDER/ingressDefault.yaml"
if [ "$FRONT_BUILD" = false ]; then
	kubectl delete configmap "$BUILDFILE_NAME-logstash" -n default
	kubectl create configmap "$BUILDFILE_NAME-logstash" \
		--from-file=logstash-config.yaml="$BUILD_FOLDER/logstashDefault.yaml" \
		-n default
fi
```