# gdgoc-cloud-2

GDGoC Yonsei Cloud 2회차 실습. 자기 소개 카드를 Docker 이미지로 만들어 실행하고 Docker Hub에 올립니다.

준비물: Docker Desktop, Docker Hub 계정, Git. Node.js는 없어도 됩니다.

- `<아이디>`는 자기 Docker Hub 아이디로 바꿔서 입력합니다.
- 2단계부터는 모든 명령을 `mycard` 폴더 안에서 실행합니다.
- Windows는 PowerShell을 씁니다. cmd와 Git Bash는 경로 표기가 달라 일부 명령이 안 됩니다.

명령은 공식 [CLI cheat sheet](https://docs.docker.com/get-started/docker_cheatsheet.pdf)를 옆에 두고 진행합니다.

## 0. 첫 컨테이너

```sh
docker run -d -p 8080:80 --name web nginx
```

http://localhost:8080

```sh
docker ps
docker logs web
docker exec -it web sh      # 안으로. exit 로 나옴
docker stop web
docker start web
docker rm -f web
docker images
docker rmi nginx
```

이미지는 보관, 컨테이너는 실행.

## 1. 노트북에서 실행

```sh
git clone https://github.com/joyrla/gdgoc-cloud-2.git mycard
cd mycard
npm install
npm run dev
```

http://localhost:4321 에 카드가 뜨거나, Node.js 버전 오류가 나거나, `npm: command not found`가 납니다.
같은 코드, 노트북마다 다른 결과. Ctrl+C로 멈춥니다.

## 2. Dockerfile

```
FROM node:22-alpine                        # Node.js가 설치된 리눅스
WORKDIR /app
COPY package.json package-lock.json ./     # 의존성 목록
RUN npm install                            # 느림, 자주 안 바뀜
COPY . .                                   # 코드. 빠름, 자주 바뀜
EXPOSE 4321
CMD ["npm", "run", "dev", "--", "--host"]
```

새 서버에서 손으로 할 일을 적은 것. 느린 단계를 앞에, 자주 바뀌는 코드를 뒤에.

## 3. build, run

`src/profile.json`에 이름, 역할, 목표를 적고 `public/photo.jpg`를 넣습니다.

```sh
docker build -t <아이디>/mycard:1.0 .     # 끝의 . 까지 입력
docker run -d -p 8080:4321 --name mycard <아이디>/mycard:1.0
```

http://localhost:8080. Node.js가 있든 없든 같은 결과.

```sh
docker exec -it mycard sh   # node -v, ls, exit
docker logs mycard
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
docker run -d -p 8090:4321 --name friend <옆사람아이디>/mycard:1.0
```

http://localhost:8090. 코드도 설치도 없이 실행.

## 5. 수정, 다시 build

`src/profile.json`에서 한 줄을 고치고,

```sh
docker build -t <아이디>/mycard:1.1 .     # npm install 은 CACHED
docker push <아이디>/mycard:1.1           # 바뀐 레이어만
docker run -d -p 8080:4321 --name mycard <아이디>/mycard:1.1
```

`port is already allocated`: 1.0이 아직 8080을 쓰고 있습니다.

```sh
docker rm -f mycard
docker run -d -p 8080:4321 --name mycard <아이디>/mycard:1.1
```

## 6. 마운트

5단계는 한 줄 고칠 때마다 build였습니다. 개발할 때는 폴더를 컨테이너에 연결합니다.

```sh
docker rm -f mycard
docker run -d -p 8080:4321 --name dev -v "$PWD/src":/app/src <아이디>/mycard:1.1
```

`src/profile.json`을 고치고 저장하면 새로고침만으로 반영됩니다. 노트북의 `src`와 컨테이너의 `/app/src`가 같은 폴더입니다.

```sh
docker exec dev cat /app/src/profile.json
docker rm -f dev
```

서버에는 내 폴더가 없으므로 서버에서는 쓰지 않습니다. 개발은 마운트, 배포는 이미지.
Windows cmd는 `"$PWD"` 대신 `%cd%`.

## 정리

```sh
docker rm -f mycard friend dev
```

이미지와 Docker Hub 저장소는 다음 시간에 씁니다.

## 문제

- `Cannot connect to the Docker daemon`: Docker Desktop 실행
- `denied: requested access to the resource is denied`: 이미지 이름이 `<아이디>/`로 시작하는지, `docker login` 했는지
- `port is already allocated`: `docker ps`, `docker rm -f <이름>`
- `exec format error`: Mac과 Windows의 CPU가 다름. `docker build --platform linux/amd64`로 다시 build, push
- 사진이 안 바뀜: `public/photo.jpg` 이름과 위치
