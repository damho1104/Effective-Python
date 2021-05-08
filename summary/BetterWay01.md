# 1. 사용중인 파이썬 버전을 알아두라

## 1. 사용하는 파이썬 버전을 알아둬야 하는 이유

- 협업하는 상황에서 각 개발자 PC 환경에 따라 파이썬 버전이나 패키지 버전이 다르게 되면 이로 인한 side-effect 발생 가능성이 높아짐
- 패키지의 경우 `requirements.txt` 에 패키지와 버전을 통일화하여 같은 버전의 패키지를 사용함
- 같은 프로젝트에서 협업하는 개발자들은 같은 버전의 같은 패키지를 사용이 필요함
- 파이썬 3 사용을 권장

## 2. 버전 확인법

- Command-Line Interface 에서 확인

    ```bash
    $ python --version

    >>> Python 3.8.1
    ```

- 내장 모듈 sys 값 사용

    ```python
    import sys

    print(sys.version)
    print(sys.version_info)

    >>> 
    3.8.9 (tags/v3.8.9:a743f81, Apr  6 2021, 14:02:34) [MSC v.1928 64 bit (AMD64)]
    sys.version_info(major=3, minor=8, micro=9, releaselevel='final', serial=0)
    ```

    - `sys.version`: 문자열
    - `sys.version_info`: 클래스, 각 어트리뷰트 접근 가능함