# 4. C 스타일 형식 문자열을 str.format과 쓰기보다는 f-문자열을 통한 인터폴레이션을 사용하라

## 1. 형식화

- 형식화: 미리 정의된 문자열에 데이터를 끼워 넣어 사람이 보기 좋은 문자열로 저장하는 과정

## 2. 4가지 방식 형식화

### 2.1. **% 형식화**

```python
format_string = '%d, %s' % (1, 'str')

>>>
1, str
```

- 형식 지정자 사용
- C 의 printf 에서 동일하게 사용, C 의 형식지정자와 동일
- 단점
    - tuple 내 데이터 순서를 바꾸거나 값의 타입을 변경하였을 때 타입 변환 불가
    - 형식화하기 전 값을 변경해야 한다면 식을 읽기가 어려워짐
    - 형식화 문자열에서 같은 값을 여러 번 사용하고 싶다면 튜플에서 같은 값 여러번 반복해야 함
        - 딕셔너리 형식 문자열을 사용하여 해결 가능

            ```python
            key = 'my_var'
            value = 1.234

            old_way = '%-10s = %.2f' % (key, value)’

            new_way = '%(key)-10s = %(value).2f' % {
                'key': key, 'value': value} # 원래 방식

            reordered = '%(key)-10s = %(value).2f' % {
                'value': value, 'key': key} # 바꾼 방식
            ```

    - 딕셔너리 형식 문자열 방법은 형식화 식이 길어져 시각적 단점(가독성 저하)이 생김

### 2.2. 내장 함수 format, str.format

- 반복 시 중복 존재
- 가독성은 떨어짐
    - 천 단위 구분자 표시용(,) 및 중앙에 값 표시용(^) 사용한 format 내장함수 예제

        ```python
        a = 1234.5678
        formatted = format(a, ',.2f')
        print(formatted)

        b = 'my 문자열'
        formatted = format(b, '^20s')
        print('*', formatted, '*')

        >>>
        1,234.57
        *        my 문자열        *
        ```

- 위치 지정자 {} 사용 예제

    ```python
    # 각 위치 지정자 형식 지정 방법(콜론 사용)
    formatted = '{:<10} = {:.2f}'.format(key, value)
    print(formatted)

    >>>
    my_var      = 1.23
    ```

    ```python
    # 딕셔너리 키, 리스트 인덱스 조합 사용
    formatted = '첫 번째 글자는 {menu[oyster][0]!r}'.format(menu=menu)
    print(formatted)

    >>>
    첫 번째 글자는 't’
    ```

## 2.3. 인터폴레이션을 통한 형식 문자열(사용 권장)

- f-문자열
    - 형식 문자열 앞에 f 문자 붙임
- 참고
    - 바이트 문자열 앞에 b 문자 붙임
    - 로우 문자열 앞에 r 문자 붙임(이스케이프하지 않아도 되는 문자열)

- 형식 문자열 표현력 극대화
- 형식화 단점을 피할 수 있음

```python
key = 'my_var'
value = 1.234

formatted = f'{key} = {value}'
print(formatted)

>>>
my_var = 1.234
```

```python
# 유니코드, repr 문자열로 변환 가능
formatted = f'{key!r:<10} = {value:.2f}'
print(formatted)

>>>
'my_var' = 1.23
```

```python
# 연속된 문자열을 서로 연결해주는 기능을 사용하여 f-문자열을 여러 줄로 나눈 예제
for i, (item, count) in enumerate(pantry):
    print(f'#{i+1}: '
    f'{item.title():<10s} = '
    f'{round(count)}')

>>>
#1: 아보카도     = 1
#2: 바나나     = 2
#3: 체리     = 15
```

```python
# 형식 지정자 옵션 사용
places = 3
number = 1.23456
print(f'내가 고른 숫자는 {number:.{places}f}')

>>>
내가 고른 숫자는 1.235
```