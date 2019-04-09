# Django-Lambda

zappa + django를 빠르게 배포할 수 있는 기본 세팅



## 포함된 기능



### 1. 공통 (base.py)

- [12-factor][https://12factor.net/ko/]를 사용하여 Django앱의 환경변수를 구성하는 **django-environ**
- 이외 모든 환경에서 필요로하는 최적화된 기본 설정들



### 2. 로컬 (local.py)

- 배포를 위한 **zappa**설정
- Debug를 위한 툴 (debug-toolbar, extensions)



### 3. 배포 (production.py)

- CAHCE 설정 ( Redis 사용하기, 주석처리 되어있음 )
- SECURITY 설정 ( 배포 후 https 설정하고 필요한 설정 구성, 주석처리 되어있음 )
- MEDIA, STATIC 파일 S3
- 메일 전송을 위한 **Anymail(Sengrid)** 설정
- 에러 트래킹을 위한 **Sentry**



### 4. 테스트 (test.py)

- 통합테스트를 위한 **tox**
- 코드 스타일 통합을 위한 **flake8, pylint, black**
- 장고 내부 테스트를 위한 **pytest** 및 **coverage**



## 프로젝트 구성

```
~/<project_name>/

<project_name>/
	- .requirements/
		- requirements_base.txt
		- requirements_local.txt
		- requirements_production.txt
		- requirements_test.txt
	- app
		+ .env_settings/
			+ .env_local
			+ .env_production
		- config/
		- <other_apps>/
		- manage.py
		+ zappa_settings.json
		...
		
	- .coveragerc
	- .flake8
	- .gitignore
	- .pylintrc
	- conftest.py
	- pytest.ini
	- tox.ini
```

배포환경에서 실행한 필요한 것은 모두 `app` 폴더 안에 넣는다. 또한 배포시에도 app안에서 app 안의 내용만 배포한다. 그 바깥에는 프로젝트 배포와는 관련없는 파일을 둔다.

`+`로 표시된 폴더 혹은 파일은 직접 생성해야한다. 공개되어서는 안되는 정보를 포함한다.



`env_local`

```text
## GENERAL ##
DJANGO_DEBUG=<True or False>
DJANGO_SECRET_KEY=<SECRET_KEY>


## ADMIN ##
DJANGO_ADMINS=<NAME:EMAIL>,<NAME:EMAIL> ...


## DATABASE ##

## CACHES ##

## SECURITY ##

## STORAGES ##

## EMAIL ##

## ANYMAIL ##

## LOGGING ##

```



`env_production`

```text
## GENERAL ##
DJANGO_DEBUG=<True or False>
DJANGO_SECRET_KEY=<SECRET_KEY>
DJANGO_ALLOWED_HOSTS=<HOST>,<HOST> ...

## ADMIN ##
DJANGO_ADMIN_URL=<CUSTOM_ADMIN_URL>
DJANGO_ADMINS=<NAME:EMAIL>,<NAME:EMAIL> ...


## DATABASE ##
DATABASE_URL=psql://<DATABASE_URL>
DATABASE_NAME=<DATABASE_NAME>
DATABASE_USER=<DATABASE_USER>
DATABASE_PASSWORD=<DATABASE_PASSWORD>

## CACHES ##

## SECURITY ##

## STORAGES ##
DJANGO_AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID>
DJANGO_AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY>
DJANGO_AWS_STORAGE_BUCKET_NAME=<AWS_BUCKET_NAME>
DJANGO_AWS_S3_REGION_NAME=ap-northeast-2


## EMAIL ##
DJANGO_DEFAULT_FROM_EMAIL=
DJANGO_SERVER_EMAIL=
DJANGO_EMAIL_SUBJECT_PREFIX=


## ANYMAIL ##
SENDGRID_API_KEY=<SENDGRID_API_KEY>


## LOGGING ##
SENTRY_DSN=<SENTRY_DSN>
DJANGO_SENTRY_LOG_LEVEL=<LOG_LEVEL>
```



`zappa_settings.json`

```text
{
    "dev": {
	"aws_region": "ap-northeast-2",
        "django_settings": "config.settings.production",
        "profile_name": "<AWS_CREDENTIA_PROFILE_NAMEL>",
        "project_name": "<PROJECT_NAME>",
        "runtime": "python3.6",
        "s3_bucket": "<S3_BUCKET_NAME>",
        "vpc_config": {
          "SubnetIds": [
            "<SUBNET1>",
            "<SUBNET2>"
          ],
          "SecurityGroupIds": [
            "<LAMBDA_SECURITY_GROUPID>"
          ]
        }
	}
}
```



## zappa deploy

Docker가 사용환경을 동일하게 만들어주므로 도커환경을 사용한다.



### 1. lambda용 Docker 설치 및 Dockerfile 설정

- 로컬에서 docker image를 받음

```shell
# For Python 3.6 projects
docker pull lambci/lambda:build-python3.6
```

- Dockerfile 생성

```dockerfile
FROM lambci/lambda:build-python3.6

LABEL maintainer="<your@email.com>"

WORKDIR /var/task

# Fancy prompt to remind you are in zappashell
RUN echo 'export PS1="\[\e[36m\]zappashell>\[\e[m\] "' >> /root/.bashrc

# Additional RUN commands here
# RUN yum clean all && \
#    yum -y install <stuff>

CMD ["bash"]
```

- Dockerfile Build

  **docker_image_name은 마음대로 설정**

```shell
$ cd /<project_name>
$ docker build -t <docker_image_name> .
```

- Docker를 빠르게 실행하기위해 **alias** 설정

  **docker_image_name은 마음대로 설정, aws_profile은 credential에 설정한 레이벨 이름을 사용**

  zsh을 사용하기때문에 **.zshrc**파일에 넣었고 zsh을 사용하지 않는경우 아래 파일중 하나에 넣는다.

  - `.profile` - 로그인할 때 로드된다. PATH처럼 로그인할 때 로드해야 하는데 bash와 상관없는 것들을 여기에 넣는다.
  - `.bash_profile` - 로그인할 때 로드된다. 'bash completion'이나 'nvm'같이 로그인할 때 로드해야 하는데 Bash와 관련된 것이면 여기에 넣는다.
  - `.bashrc` - 로그인하지 않고 Bash가 실행될 때마다 로드된다.

```shell
$ alias zappashell='sudo docker run -ti -e AWS_PROFILE=<aws_profile> -v "$(pwd):/var/task" -v ~/.aws/:/root/.aws  --rm <docker_image_name>'
$ alias zappashell | { read s; echo "alias ${s}" } >> ~/.zshrc
```



### 2. Docker를 사용하여 Deploy

-  docker내부에서 배포에 필요한 환경을 설치한다.

```shell
$ zappashell
zappashell> python -m venv ve
zappashell> source ve/bin/activate 
(ve) zappa> pip install -r .requirements/requirements_base.txt
(ve) zappa> pip install -r .requirements/requirements_production.txt
```

가상환경의 현재 디렉터리는 로컬 시스템의 프로젝트에 매핑되어있다. `virtualenv`가 현재 폴더에 `ve` 폴더를 만들고 그 안에 Python 라이브러리들이 설치되어 컨테이너가 종료되어도 유지될 수 있지만 , 시스템라이브러리의 경우 컨테이너가 종료되면 손실된다. 해결책은 Dockerfile에 `RUN` 명령어를 사용하여 설치를 유지하는것이다.

- deploy

```shell
(ve) zappa> cd app
(ve) zappa> zappa deploy <production_environment>
```

**production_environment** 는 **zappa_settings.json** 에 있는 설정중 하나 사용 (위 예제에서는 **dev**)

