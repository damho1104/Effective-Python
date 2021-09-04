# 36. 이터레이터나 제너레이터를 다룰 때는 itertools 를 사용하라

## 1. `itertools`

- **여러 이테레이터 연결하기**
    - `chain`
        - 여러 이터레이터를 하나의 순차적인 이터레이터로 합치고 싶을 때 사용

        ```python
        import itertools
        it = itertools.chain([1, 2, 3], [4, 5, 6])
        print(list(it))

        >>>
        [1, 2, 3, 4, 5, 6]
        ```

        - 각 리스트를 이터레이터로 변환, 이터레이터 끼리 연결

    - `repeat`
        - 특정 값을 반복하여 반복하여 전달하고 싶을 때 사용
        - 2번째 인자: 반복 최대 횟수

        ```python
        import itertools
        it = itertools.repeat('안녕', 3)
        print(list(it))

        >>>
        ['안녕', '안녕', '안녕']
        ```

    - `cycle`
        - 해당 리스트의 이터레이터를 반복하고 싶은 경우 사용

        ```python
        import itertools
        it = itertools.cycle([1, 2])
        result = [next(it) for _ in range (10)]
        print(result)

        >>>
        [1, 2, 1, 2, 1, 2, 1, 2, 1, 2]
        ```

    - `tee`
        - 하나의 리스트의 이터레이터를 여러개 쓰고 싶은 경우 사용
        - 2번째 인자: 이터레이터 개수
        - 여러 개 이터레이터의 싱크가 맞지 않은 경우, 각 이터레이터마다 다른 값을 가질 수 있으므로 메모리 낭비의 원인이 될 수 있음

        ```python
        import itertools
        it1, it2, it3 = itertools.tee(['하나', '둘'], 3)
        print(list(it1))
        print(list(it2))
        print(list(it3))

        >>>
        ['하나', '둘']
        ['하나', '둘']
        ['하나', '둘']
        ```

    - `zip_longest`
        - `zip` 내장 함수의 변종
        - 여러 이터레이터를 동시에 수행할 수 있는데 그 중 짧은 이터레이터의 원소를 다 사용한 경우 `fillvalue` 로 전달한 값으로 채움
            - `fillvalue` 로 전달하지 않은 경우 기본값: `None`

        ```python
        import itertools
        keys = ['하나', '둘', '셋']
        values = [1, 2]

        normal = list(zip(keys, values))
        print('zip:', normal)

        it = itertools.zip_longest(keys, values, fillvalue='없음')
        longest = list(it)
        print('zip_longest:', longest)

        >>>
        zip: [('하나', 1), ('둘', 2)]
        zip_longest: [('하나', 1), ('둘', 2), ('셋', '없음')]
        ```

- **이러테이터에서 원소 필터링**
    - `islice`
        - 이터레이터를 복사하지 않으면서 원소 인덱스를 이용하여 슬라이싱하고 싶은 경우 사용
        - 2번째 인자: stop or start 인덱스, 인자 개수가 2개면 stop, 3개 이상이면 start, `None` ⇒ 0
        - 3번째 인자: end 인덱스, `None` ⇒ -1
        - 4번째 인자: step, start 부터 end 까지중에서 건너띌 수, 기본값: 1

        ```python
        import itertools
        values = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

        first_five = itertools.islice(values, 5)
        print('앞에서 다섯 개:', list(first_five))

        middle_odds = itertools.islice(values, 2, 8, 2)
        print('중간의 홀수들:', list(middle_odds))

        >>>
        앞에서 다섯 개: [1, 2, 3, 4, 5]
        중간의 홀수들: [3, 5, 7]
        ```

    - `takewhile`
        - 주어진 `predicate` 가 `False` 를 반환하는 첫번째 원소가 나타날 때까지의 원소들을 반환, `False` 된 원소 전까지

        ```python
        import itertools
        values = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        less_than_seven = lambda x: x < 7
        it = itertools.takewhile(less_than_seven, values)
        print(list(it))

        >>>
        [1, 2, 3, 4, 5, 6]
        ```

    - `dropwhile`
        - 주어진 `predicate` 가 `False` 를 반환하는 첫번째 원소가 나타날 때까지의 원소들을 건너뜀, True 된 원소들 반환

        ```python
        import itertools
        values = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        less_than_seven = lambda x: x < 7
        it = itertools.dropwhile(less_than_seven, values)
        print(list(it))

        >>>
        [7, 8, 9, 10]
        ```

    - `filterfalse`
        - 이터레이터에서 `Predicate` 가 `False` 를 반환하는 모든 원소 반환
        - `filter` 내장 함수의 반대

        ```python
        import itertools
        values = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        evens = lambda x: x % 2 == 0

        filter_result = filter(evens, values)
        print('Filter:', list(filter_result))

        filter_false_result = itertools.filterfalse(evens, values)
        print('Filter false:', list(filter_false_result))

        >>>
        Filter: [2, 4, 6, 8, 10]
        Filter false: [1, 3, 5, 7, 9]
        ```

- **이터레이터에서 원소의 조합 만들어내기**
    - `accumulate`
        - 파라미터를 2개 받는 함수를 반복 적용하면서 이터레이터 원소를 값 하나로 줄여줌
        - fold 연산
        - 함수를 적용하지 않는다면 주어진 입력 이터레이터 원소의 합계 계산

        ```python
        import itertools
        values = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
        sum_reduce = itertools.accumulate(values)
        print('합계:', list(sum_reduce))

        def sum_modulo_20(first, second):
            output = first + second
            return output % 20
        modulo_reduce = itertools.accumulate(values, sum_modulo_20)
        print('20으로 나눈 나머지의 합계:', list(modulo_reduce))

        >>>
        합계: [1, 3, 6, 10, 15, 21, 28, 36, 45, 55]
        20으로 나눈 나머지의 합계: [1, 3, 6, 10, 15, 1, 8, 16, 5, 15]
        ```

    - `product`
        - 하나 이상의 이터레이터에 들어 있는 아이템들의 cartesian product 반환

        ```python
        import itertools
        single = itertools.product([1, 2], repeat=2)
        print('리스트 한 개:', list(single))

        multiple = itertools.product([1, 2], ['a', 'b'])
        print('리스트 두 개:', list(multiple))

        >>>
        리스트 한 개: [(1, 1), (1, 2), (2, 1), (2, 2)]
        리스트 두 개: [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]
        ```

    - `permutations`
        - 원소들로부터 길이 N 인 순열을 반환
        - 2번째 인자: 순열 길이 N

        ```python
        import itertools
        it = itertools.permutations([1, 2, 3, 4], 2)
        print(list(it))

        >>>
        [(1, 2), (1, 3), (1, 4), (2, 1), (2, 3), (2, 4), (3, 1), (3, 2), (3, 4), (4, 1), (4, 2), (4, 3)]
        ```

    - `combinations`
        - 원소들로부터 길이 N인 조합을 반환
        - 2번째 인자: 조합 길이 N

        ```python
        import itertools
        it = itertools.combinations([1, 2, 3, 4], 2)
        print(list(it))

        >>>
        [(1, 2), (1, 3), (1, 4), (2, 3), (2, 4), (3, 4)]
        ```

    - `combinations_with_replacement`
        - `combinations` 와 같으나 원소 반복 허용(중복 허용)

        ```python
        it = itertools.combinations_with_replacement([1, 2, 3, 4], 2)
        print(list(it))

        >>>
        [(1, 1), (1, 2), (1, 3), (1, 4), (2, 2), (2, 3), (2, 4), (3, 3), (3, 4), (4, 4)]
        ```