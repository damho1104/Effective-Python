# 3. bytes 와 str의 차이를 알아두라

## 1. 문자열을 표현하는 방법

- byte: **부호가 없는** 8바이트 데이터가 그대로 들어감

    ```python
    a = b'h\x65llo'
    print(list(a))
    print(a)

    >>>
    [104, 101, 108, 108, 111]
    b'hello'
    ```

- str: human-readable 한 문자인 유니코드 코드 포인트가 들어감

    ```python
    a = 'a\u0300 propos'
    print(list(a))
    print(a)

    >>>
    ['a'. '`', ' ', 'p', 'r', 'o', 'p', 'o', 's']
    a propos
    ```

## 2. 유니코드 데이터와 이진 데이터 변환 방법

- 유니코드 데이터 → 이진 데이터
    - encode 메소드
- 이진 데이터 → 유니코드 데이터
    - decode 메소드

```python
text_string = 'hello'

# 유니코드 데이터 -> 이진 데이터
text_byte = text_string.encode()
print(text_byte)

# 이진 데이터 -> 유니코드 데이터
text_string = text_byte.decode('utf-8')
print(text_string)

>>>
b'hello'
hello
```

## 3. 문자열 사용 시 발생하는 문제 해결 방안

- 유니코드 문자열을 다룰 때 한쪽 타입으로 통일시켜 줄 필요가 있음
    - 이유: 타입 호환이 되지 않기 때문
    - 유니코드 문자열로 변환(인코딩: utf-8)

        ```python
        def to_str(value):
        	if isinstance(value, bytes):
        		return value.decode('utf-8')
        	else:
        		return value
        ```

    - 이진 문자열로 변환(인코딩: 인자로 전달)

        ```python
        def to_byte(value, encoding):
        	if isinstance(value, bytes):
        		return value
        	else:
        		return value.encode(encoding)
        ```

- 파일 핸들과 관련된 operator 들이 유니코드 문자열을 요구함, 명확한 모드를 명시하여 사용해야 함
    - 'w', 'r' ⇒ 유니코드 문자열 요구
    - 'wb', 'rb' ⇒ 이진 문자열 요구

- 유니코드 문자열을 다룰 때, 시스템 기본 인코딩 확인 필요

    ```python
    import locale
    print(locale.getpreferredencoding())

    >>>
    UTF-8
    ```

- 기본 인코딩과 다른 인코딩을 필요로 하면 `encoding` 파라미터 명시적으로 사용

## 4. 해결하지 못한 문자열 관련 문제점

- OS 별 혹은 언어 별 사용하는 텍스트 인코딩이 달라 문제를 겪은 적이 굉장히 많았음
    - 윈도우 한글: `CP949`
    - 윈도우 중국: `CP936`
    - Linux: `UTF-8`
- 실제 현업에서는 텍스트 인코딩을 구분 않고 쓰는 경우가 많음
- 이 경우 추후 문제가 발생할 소지가 다분하며 한 인코딩으로 set 하는 것이 필수적이라 생각함
- python 에서 `byte` 타입으로 읽어 문자열 텍스트 인코딩을 추측하는 방법(using `chardet`)이 있으나 성능 저하 우려가 있음
- 아직 만능은 없는듯... 더 찾아보고 공부해야 할 것 같다.