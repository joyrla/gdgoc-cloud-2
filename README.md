# gdgoc-cloud-2

GDGoC Yonsei Cloud 2회차 실습. 자기 소개 카드를 Docker 이미지로 만들어 Docker Hub에 올리고, 옆 사람 카드를 받아 실행합니다.

준비물: Docker Desktop, Docker Hub 계정, Git.

- `<아이디>`는 자기 Docker Hub 아이디로 바꿔서 입력합니다.
- 2단계의 clone 이후 모든 명령은 `mycard` 폴더 안에서 실행합니다.
- Windows는 PowerShell을 씁니다. cmd와 Git Bash는 경로 표기가 달라 일부 명령이 안 됩니다.

명령은 공식 [CLI cheat sheet](https://docs.docker.com/get-started/docker_cheatsheet.pdf)를 옆에 두고 진행합니다.

## 1. 수업 이미지 실행

```sh
docker run -d -p 8080:4321 --name card yonseijk/gdgoc_cloud_2:1.0
```

http://localhost:8080 에 빈 카드가 뜹니다. Node.js를 설치한 적이 없는데 뜹니다.

```sh
docker ps
docker logs card
docker exec -it card sh     # 안으로. ls, exit
docker stop card
docker start card
docker rm -f card           # 삭제. 이미지는 남음
docker images
```

이미지는 보관, 컨테이너는 실행.

## 2. clone, 수정

```sh
git clone https://github.com/joyrla/gdgoc-cloud-2.git mycard
cd mycard
```

`src/profile.json`에 이름, 역할, 목표를 적고 `public/photo.jpg`를 넣습니다.

```
FROM node:22-alpine                        # Node.js가 설치된 리눅스
WORKDIR /app
COPY package.json package-lock.json ./     # 의존성 목록 → /app
RUN npm install                            # 느림, 자주 안 바뀜
COPY . .                                   # 코드 전체 → /app. 빠름, 자주 바뀜
EXPOSE 4321
CMD ["npm", "run", "dev", "--", "--host"]
```

`Dockerfile`은 새 서버에서 손으로 할 일을 적은 것. 느린 단계를 앞에, 자주 바뀌는 코드를 뒤에.

## 3. build, run

```sh
docker build --platform linux/amd64 -t <아이디>/mycard:1.0 .     # 끝의 . 까지
docker run -d -p 8080:4321 --name mycard <아이디>/mycard:1.0
```

http://localhost:8080 에 내 카드. `--platform linux/amd64`는 Windows와 서버의 CPU 기준으로 만드는 옵션이고 Mac에서도 실행됩니다.

```sh
docker exec -it mycard sh   # node -v, ls, exit
docker history <아이디>/mycard:1.0
```

## 4. push, pull

```sh
docker login                # 브라우저가 열리면 거기서 로그인
docker push <아이디>/mycard:1.0
```

`Mounted from library/node`는 이미 Docker Hub에 있는 레이어. 내가 만든 레이어만 올라갑니다.

옆 사람 카드:

```sh
docker run --platform linux/amd64 -d -p 8090:4321 --name friend <옆사람아이디>/mycard:1.0
```

http://localhost:8090. 코드도 설치도 없이 실행. `--platform`은 amd64로 만든 이미지를 Mac에서 받을 때 필요합니다.

## 5. 수정, 다시 build (선택)

`src/profile.json`에서 한 줄을 고치고,

```sh
docker build --platform linux/amd64 -t <아이디>/mycard:1.1 .     # npm install 은 CACHED
docker push <아이디>/mycard:1.1                                   # 바뀐 레이어만
docker run -d -p 8080:4321 --name mycard <아이디>/mycard:1.1
```

`port is already allocated`: 1.0이 아직 8080을 쓰고 있습니다.

```sh
docker rm -f mycard
docker run -d -p 8080:4321 --name mycard <아이디>/mycard:1.1
```

## 정리

```sh
docker rm -f card mycard friend
```

이미지와 Docker Hub 저장소는 다음 시간에 씁니다.

## 문제

- `Cannot connect to the Docker daemon`: Docker Desktop 실행
- `denied: requested access to the resource is denied`: 이미지 이름이 `<아이디>/`로 시작하는지, `docker login` 했는지
- `port is already allocated`: `docker ps`, `docker rm -f <이름>`
- `no matching manifest for linux/arm64`: Mac에서 amd64 이미지를 받을 때. `docker run --platform linux/amd64 ...`
- 사진이 안 바뀜: `public/photo.jpg` 이름과 위치

## 처음부터 다시

꼬였을 때, 가벼운 것부터.

```sh
docker rm -f $(docker ps -aq)      # 컨테이너 전부 삭제. 이미지는 남음
```

```sh
cd .. && rm -rf mycard && git clone https://github.com/joyrla/gdgoc-cloud-2.git mycard && cd mycard
```

```sh
docker system prune -af            # 이미지까지 전부 삭제. 1단계부터 다시
```

명령이 아무 반응이 없으면 Docker Desktop을 종료하고 다시 엽니다. Docker Hub는 되돌릴 것이 없습니다. 같은 태그로 다시 push하면 덮어씁니다.
