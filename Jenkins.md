### 0. 사전 정의된 변수
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