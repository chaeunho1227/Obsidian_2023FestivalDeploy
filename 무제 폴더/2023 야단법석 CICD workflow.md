### 코드 전문

```Bash
name: Web Application Server - CI/CD

on:
    push:
        branches: ["main"]
    pull_request_target:
        types: [labeled, closed]

jobs:
# safe tag에 대한 gradlew test && merged에 대한 docker image build and push
  CI:
    if: contains(github.event.pull_request.labels.*.name, 'safe')
    runs-on: ubuntu-20.04

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Create .env
      shell: bash
      run:
        touch .env
        echo SECRET_KEY=${{ secrets.SECRET_KEY }} >> .env
        echo ADMIN_PATH=${{ secrets.ADMIN_PATH }} >> .env
        echo DEBUG_VALUE=${{ secrets.DEBUG_VALUE }} >> .env
        echo DJANGO_DEPLOY=${{ secrets.DJANGO_DEPLOY }} >> .env
        echo DATABASE_ENGINE=${{ secrets.DATABASE_ENGINE }} >> .env
        echo DATABASE_NAME=${{ secrets.DATABASE_NAME }} >> .env
        echo DATABASE_USER=${{ secrets.DATABASE_USER }} >> .env
        echo DATABASE_USER_PASSWORD=${{ secrets.DATABASE_USER_PASSWORD }} >> .env
        echo DATABASE_HOST=${{ secrets.DATABASE_HOST }} >> .env
        echo DATABASE_PORT=${{ secrets.DATABASE_PORT }} >> .env
        cat .env

    - name: Set up Python 3.8
      uses: actions/setup-python@v2
      with:
        python-version: 3.8

    - name: Install dependencies
      run:
        python -m pip install --upgrade pip
        pip3 install -r requirements.txt

    - name: Run tests
      run:
	    python3 manage.py test

    ### Docker Image Build and Push ###
    - name: Login to Docker Hub
      if: github.event.pull_request.merged == true
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
            
    - name: Set up Docker Buildx
      if: github.event.pull_request.merged == true
      uses: docker/setup-buildx-action@v2
                
    - name: Build and push
      if: github.event.pull_request.merged == true
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKERHUB_REPONAME }}
  
  # closed에 대한 server deploy
  CD:
    if: github.event.pull_request.merged == true
    needs: [CI]
    
    runs-on: ubuntu-20.04

    steps:
    ### SSH Connect and Docker Image Pull and Container Run
    - name: Docker Image Pull and Container Run
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.SSH_HOST }}
        username: ${{ secrets.SSH_USERNAME }}
        password: ${{ secrets.SSH_PASSWORD }}
        port: ${{ secrets.SSH_PORT }}
        script: |
          docker stop festival2023-was
          docker rm festival2023-was
          docker image rm ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKERHUB_REPONAME }}
          docker run -d --net festival -v festival-media:/app/media -v festival-static:/app/static -e TZ=Asia/Seoul -p 8000:8000 --name festival2023-was ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKERHUB_REPONAME }}
```


*** 
## 코드 정리
#### workflow 정의
- workflow 이름 정의
```Bash
name: Web Application Server - CI/CD
```
github의 workflow의 이름을 정하는 내용

- workflow의 실행조건 정의
```bash
on:
    push:
        branches: ["main"]
    pull_request_target:
        types: [labeled, closed]
```
이 workflow는 `main` 브랜치로 push 되고 label이 달려있거나 closed 된 PR에 대해서만 실행된다.
#### Job : CI
- CI 실행조건 추가
```Bash
CI:
    if: contains(github.event.pull_request.labels.*.name, 'safe')
    runs-on: ubuntu-20.04
```
`safe` 라는 이름의 label이 달린 PR에 대해서 작동한다.

- 기본 세팅
```Bash
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Create .env
      shell: bash
      run:
        touch .env
        echo SECRET_KEY=${{ secrets.SECRET_KEY }} >> .env
        echo ADMIN_PATH=${{ secrets.ADMIN_PATH }} >> .env
        echo DEBUG_VALUE=${{ secrets.DEBUG_VALUE }} >> .env
        echo DJANGO_DEPLOY=${{ secrets.DJANGO_DEPLOY }} >> .env
        echo DATABASE_ENGINE=${{ secrets.DATABASE_ENGINE }} >> .env
        echo DATABASE_NAME=${{ secrets.DATABASE_NAME }} >> .env
        echo DATABASE_USER=${{ secrets.DATABASE_USER }} >> .env
        echo DATABASE_USER_PASSWORD=${{ secrets.DATABASE_USER_PASSWORD }} >> .env
        echo DATABASE_HOST=${{ secrets.DATABASE_HOST }} >> .env
        echo DATABASE_PORT=${{ secrets.DATABASE_PORT }} >> .env
        cat .env
```
여기서 checkout은 "현재 github의 내용을 action에서 사용하겠다." 라는 내용으로 이해 가능 
이때 checkout을 통해 받아온 내용에는 `.env` 파일이 없기에 github secrets의 값을 `.env`를 만들어 저장한다.

- 실행 환경 세팅
```Bash
    - name: Set up Python 3.8
      uses: actions/setup-python@v2
      with:
        python-version: 3.8

    - name: Install dependencies
      run:
        python -m pip install --upgrade pip
        pip3 install -r requirements.txt

    - name: Run tests
      run:
	    python3 manage.py test
```
프로젝트의 python 버전에 맞춰 3.8버전으로 실행환경의 python 버전을 설정한다.
이후 `pip install -r requirements.txt`로실행에 필요한 패키지를 설치 후 실행이 잘 되는지 테스트한다..

- Docker image build & Push
```Bash
    - name: Login to Docker Hub
      if: github.event.pull_request.merged == true
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
            
    - name: Set up Docker Buildx
      if: github.event.pull_request.merged == true
      uses: docker/setup-buildx-action@v2
                
    - name: Build and push
      if: github.event.pull_request.merged == true
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKERHUB_REPONAME }}
```
각각의 step에 if문을 이용해 merge가 완료된 내용만 실행되도록 설정한다.
docker image를 docker hub에 push하기 위해 로그인을 하도록 한다.
현재 git action이 실행되고 있는 환경에서 docker image를 만들기 위해 필요한 `docker buildx` 플러그인을 설치합니다.
실행폴더의 dockerfile으로 image를 만든 후 dockerhub에 push합니다.

#### Job : CD
- 실행환경과 실행조건
```bash
  CD:
    if: github.event.pull_request.merged == true
    needs: [CI]
    
    runs-on: ubuntu-20.04
```
CI와 마찬가지로 merge된 내용에 대해서만 실행하고 CI가 실행되어야지만 CD가 실행되도록 한다.
백엔드 서버에서 CD 과정을 실행하므로 백엔드 서버의 운영체제인 ubuntu-20.04로 설정합니다.

```
    steps:
    ### SSH Connect and Docker Image Pull and Container Run
    - name: Docker Image Pull and Container Run
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.SSH_HOST }}
        username: ${{ secrets.SSH_USERNAME }}
        password: ${{ secrets.SSH_PASSWORD }}
        port: ${{ secrets.SSH_PORT }}
        script: |
          docker stop festival2023-was
          docker rm festival2023-was
          docker image rm ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKERHUB_REPONAME }}
          docker run -d --net festival -v festival-media:/app/media -v festival-static:/app/static -e TZ=Asia/Seoul -p 8000:8000 --name festival2023-was ${{ secrets.DOCKERHUB_USERNAME }}/${{ secrets.DOCKERHUB_REPONAME }}
```
백엔드 서버 SSH로 접속한다. 기존의 django project가 열려있는 festival2023-was container와 image를 삭제 후 docker hub의 내용을 pull 받아 container를 실행합니다.
