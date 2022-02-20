# 83. 가상 환경을 사용해 의존 관계를 격리하고 반복 생성할 수 있게 하라

## 1. 패키지 의존성

- 파이선으로 개발 시 다양한 패키지에 의존하게 됨
- 문제
    - pip 가 새로운 패키지를 기본적으로 모든 파이선 인터프리터가 볼 수 있는 전역 위치에 저장
    - 내 PC 에서 실행되는 모든 파이선 프로그램이 설치한 모듈에 영향을 받음
    - 패키지의 transitive 의존 관계에 의해 문제 발생 가능
        - 설치한 패키지가 다른 패키지에 의존하는 경우
    - sphinx 패키지 의존 관계
    
    ```bash
    $ python3 -m pip show Sphinx
    Name: Sphinx
    Version: 2.1.2
    Summary: Python documentation generator
    Location: /usr/local/lib/python3.8/site-packages
    Requires: alabaster, imagesize, requests, sphinxcontrib-applehelp, sphinxcontrib-qthelp, Jinja2, setuptools, sphinxcontrib-jsmath, sphinxcontrib-serializinghtml, Pygments, snowballstemmer, packaging, sphinxcontrib-devhelp, sphinxcontrib-htmlhelp, babel, docutils
    Required-by:
    ```
    
    - flask 패키지 의존 관계
    
    ```bash
    $ python3 -m pip show flask
    Name: Flask
    Version: 1.0.3
    Summary: A simple framework for building complex web applications.              
    Location: /usr/local/lib/python3.8/site-packages
    Requires: itsdangerous, click, Jinja2, Werkzeug
    Required-by:
    ```
    
    - 시간이 지남에 따라 Sphinx, flask 패키지가 서로 달라지므로 의존 관계 충돌 발생 가능
    - 의존 관계 지옥 발생
        - 설치한 패키지 일부는 새로운 버전을 사용해야 하고 다른 일부를 예전 버전을 사용해야 한다면 시스템이 제대로 작동하지 못하게 됨
    - 새로운 버전의 라이브러리가 해당 라이브러리의 API 에 의존하는 코드의 동작을 변경시킬 수 있음
    

## 2. `venv` 패키지

- 특정 버전의 파이선 환경을 독립적으로 구성 가능
- 한 시스템 안에서 같은 패키지의 다양한 버전을 서로 충돌 없이 사용 가능
- 여러 다른 프로젝트를 진행하면서 프로젝트마다 각각 다른 도구/패키지 활용 가능

- `venv` 를 사용해 myproject 새 가상 환경 생성
- 각 가상 환경은 서로 다른 유일한 디렉터리 안에 존재해야 함
- `venv` 를 통해 가상 환경을 생성하면 가상 환경을 관리하기 위해 필요한 파일과 디렉토리 트리 존재

```bash
$ which python3
/usr/local/bin/python3
$ python3 --version
Python 3.8.0

$ python3 -m venv myproject
$ cd myproject
$ ls

bin         include     lib         pyvenv.cfg
```

- 리눅스에서 가상 환경을 사용하려면 bin/activate 스크립트에 대해 source 명령 사용, 윈도우의 경우 배치파일 실행

```bash
# in Linux
$ source bin/activate
(myproject)$

# in Windows(for CMD)
C:\> myproject\Scripts\activate.bat
(myproject) C:>

#in Windows(for PowerShell)
PS C:\> myproject\Scripts\activate.ps1
(myproject) PS C:>
```

- 가상 환경을 활성화하면 python3, pip 경로가 가상 환경 디렉토리 경로로 변경됨

```bash
(myproject)$ which python3
/tmp/myproject/bin/python3
(myproject)$ ls -l /tmp/myproject/bin/python3
... -> /usr/local/bin/python3.8
```

- 가상 환경 밖의 시스템에 전역으로 설치된 pytz 패키지를 사용하려 시도하면 오류 발생
- 가상 환경 내 새로 설치

```bash
(myproject)$ python3 -c 'import pytz'
Traceback (most recent call last):
  File "<string>", line 1, in <module>
ModuleNotFoundError: No module named 'pytz'
```

```bash
(myproject)$ python3 -m pip install pytz
Collecting pytz
  Downloading ...               
Installing collected packages: pytz
Successfully installed pytz-2019.1
```

- 가상 환경에서 기본 시스템으로 돌아가고 싶은 경우 deactivate 명령 실행

```bash
(myproject)$ which python3
/tmp/myproject/bin/python3
(myproject)$ deactivate
$ which python3
/usr/local/bin/python3
```

## 3. 의존 관계 재생성하기

- `venv` 를 사용하면 명시적으로 의존하는 모든 의존 관계를 파일제 저장 가능

```bash
(myproject)$ python3 -m pip freeze > requirements.txt
(myproject)$ cat requirements.txt
certifi==2019.3.9
chardet==3.0.4
idna==2.8
numpy==1.16.2
pytz==2018.9
requests==2.21.0
urllib3==1.24.1
```

- `venv` 로 가상 환경 생성 , activate 로 가상 환경 활성화

```bash
$ python3 -m venv otherproject
$ cd otherproject
$ source bin/activate
(otherproject)$
```

- `python3 -m pip freeze` 명령으로 만든 `requirements.txt` 에 대해 가상 환경 내에서 `pip install` 을 실행하면 가상 환경에서의 패키지 설치 가능

```bash
(otherproject)$ python3 -m pip install -r /tmp/myproject/requirements.txt
...
(otherproject)$ python3 -m pip list
Package    Version
---------- --------
certifi    2019.3.9
chardet    3.0.4
idna       2.8
numpy      1.16.2
pip        10.0.1
pytz       2018.9
requests   2.21.0
setuptools 39.0.1
urllib3    1.24.1
```

- `requirements.txt` 는 버전 관리 시스템을 사용해 다른 사람과 협업할 때 이상적
- 가상 환경 디렉토리를 통째로 옮기는 경우 모든 요소가 깨져버림
    - python3 와 같은 명령줄 도구 경로가 하드코딩되어 있기 때문
    - 이 경우 새로운 디렉토리에 가상 환경 생성 후 `requirements.txt` 를 통해 패키지 다시 내려받으면 끝