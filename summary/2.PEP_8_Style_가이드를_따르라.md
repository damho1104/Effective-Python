# 2. PEP 8 Style 가이드를 따르라

- **PEP(Python Enhancement Proposal) #8** 은 python 코드에 대한 스타일 가이드임

## 1. **공백**

- 탭 대신 **스페이스**를 사용하여 들여쓰기
- 문법적으로 중요한 들여쓰기는 **4칸 스페이스** 사용
- 라인 길이는 **79 문자 이하**
- long expression을 다음 라인에 이어서 써여 할 경우 **4 스페이스를 더 들여쓰기**
- 함수와 클래스 사이 **newline 2줄**
- 클래스 내 메서드 사이 **newline 1줄**
- Dictionary 의 Key 와 콜론(:) 사이 공백 X, 한 줄로 Key, Value 모두 표현시 콜론과 Value 사이 공백
- 변수 대입 시 = 사이로 공백
    - 타입 표기를 하는 경우 변수 이름과 콜론 사이 공백 X, 콜론과 타입 정보 사이 공백 O

## 2. **명명 규약**

- 함수, 변수, attribute 는 **lowercase_underscore** 사용
- 보호돼야 하는 인스턴스 attribute 는 **밑줄**로 시작
- private 인스턴스 attribute 는 **밑줄 2개**로 시작
- 클래스 및 Exception 은 **Camel Case** 로 사용
- 모듈 수준 상수는 모든 글자 대문자, 단어 사이 **_**
- 인스턴스 메소드는 첫번째 인자 이름으로 반드시 **self**
- 클래스 메서드는 첫번째 인자 이름으로 반드시 **cls**

## 3. **Expression 과 Statement**

- if not a is b (X) ⇒ if a is not b (O)
- 빈 컨테이너([])나 시퀀스('') 검사 시 길이를 0로 비교하지 말라(암묵적으로 `False` 취급)

    ```python
    something = []
    if len(something) == 0:
    ⇒
    if not something:
    ```

- 비어 있지 않은 컨테이너([])나 시퀀스('') 검사 시 길이로 비교하지 말라(암묵적으로 `True` 취급)

    ```python
    something = [1, 2, 3]
    if len(something) != 0:
    ⇒
    if something:
    ```

- 한 줄짜리 if 문이나 한 줄짜리 `for`, `while`, 한 줄짜리 `except` 복합문 사용 금지
    - 명확성을 위해 여러 줄 처리

- expression 을 한 줄로 쓸 수 없는 경우 괄호로 둘러싸고 줄바꿈과 들여쓰기 사용
- 여러 줄에 걸쳐 expression 작성하는 경우 \ 대신 괄호 사용

## 4. **import**

- import, from 문은 항상 파일 맨 앞에 위치
- import 시, absolute name 사용, 현 모듈 경로의 상대 이름은 사용 금지
- 상대 경로로 import 하는 경우 `from . import foo` 처럼 명시적으로 사용
- import 순서: 1. 표준 라이브러리 모듈, 2. 서드 파티 모듈, 3. 사용자 모듈, 알파벳 순서