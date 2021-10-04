# 44. 세터와 게터 메서드 대신 평범한 애트리뷰트를 사용하라

## 1. setter, getter

- 아래와 같은 코드는 파이선스럽지 않음
- 특히 필드 값 증가, 감소같은 연산의 경우 getter, setter 지정하여 사용하면 가독성 떨어짐
- 해당 유틸리티 자체는 도움이 됨
    - python 에서는 명시적인 setter, getter 필요없음
    - 기본적으로 공개 애트리뷰트로 구현 시작

```python
class OldResistor:
    def __init__(self, ohms):
        self._ohms = ohms
        
    def get_ohms(self):
        return self._ohms
        
    def set_ohms(self, ohms):
        self._ohms = ohms

r0 = OldResistor(50e3)
print('이전:', r0.get_ohms())
r0.set_ohms(10e3)
print('이후:', r0.get_ohms())

r0.set_ohms(r0.get_ohms() - 4e3)
assert r0.get_ohms() == 6e3

>>>
이전: 50000.0
이후: 10000.0

```

⇒

```python
class Resistor:
    def __init__(self, ohms):
        self.ohms = ohms
        self.voltage = 0
        self.current = 0
        
r1 = Resistor(50e3)
r1.ohms = 10e3
```

## 2. `property` 데코레이터

- 애트리뷰트가 설정될 때 특별한 기능을 수행해야 하는 경우 사용
    - getter 로써 `property` 데코레이터를 붙인 메소드 사용, 메소드 이름은 애트리뷰트명과 동일
    - setter 로써 `[애트리뷰트명].setter` 데코레이터를 붙인 메소드 사용, 메소드 이름은 애트리뷰트명과 동일
- `r2.voltage` 값이 변경될 때 setter 또한 동작함

```python
class VoltageResistance(Resistor):
    def __init__(self, ohms):
        super().__init__(ohms)
        self._voltage = 0
        
    @property
    def voltage(self):
        return self._voltage
        
    @voltage.setter
    def voltage(self, voltage):
        self._voltage = voltage 
        self.current = self._voltage / self.ohms

r2 = VoltageResistance(1e3)
print(f'이전: {r2.current:.2f} 암페어')
r2.voltage = 10
print(f'이후: {r2.current:.2f} 암페어')

>>>
이전: 0.00 암페어 
이후: 0.01 암페어
```

- 프로퍼티에 대해 setter 를 지정하면 타입 검사, 클래스 프로퍼티에 전달된 값 검증 가능

```python
class BoundedResistance(Resistor):
    def __init__(self, ohms):
        super().__init__(ohms)

    @property
    def ohms(self):
        return self._ohms

    @ohms.setter
    def ohms(self, ohms):
        if ohms <= 0:
            raise ValueError(f'저항 > 0이어야 합니다. 실제 값: {ohms}') 
        self._ohms = ohms

r3 = BoundedResistance(1e3)
r3.ohms = 0

>>>
Traceback ...
ValueError: 저항 > 0이어야 합니다. 실제 값: 0

===================================================================

BoundedResistance(-5)

>>>
Traceback ...
ValueError: 저항 > 0이어야 합니다. 실제 값: -5
```

```python
class FixedResistance(Resistor):
    def __init__(self, ohms):
        super().__init__(ohms)

    @property
    def ohms(self):
        return self._ohms

    @ohms.setter
    def ohms(self, ohms):
        if hasattr(self, '_ohms'):
            raise AttributeError("Ohms는 불변 객체입니다")
        self._ohms = ohms

r4 = FixedResistance(1e3)
r4.ohms = 2e3

>>>
Traceback ...
AttributeError: Ohms는 불변 객체입니다
```

## 3. `property` 데코레이터를 사용한 getter, setter 구현 시 주의사항

- 예기치 않은 동작을 수행하지 않도록 주의해야 함
    - case 1. getter `property` 메소드에서 다른 애트리뷰트 설정 금지
        - 객체 상태를 `@property.setter` 메소드 안에서만 변경 권장
        - 만일 더 복잡하거나 느린 연산을 수행해야 하는 경우 일반적인 메소드 사용할 것
    
    ```python
    class MysteriousResistor(Resistor):
        @property
        def ohms(self):
            self.voltage = self._ohms * self.current
            return self._ohms
            
        @ohms.setter
        def ohms(self, ohms):
            self._ohms = ohms
    
    r7 = MysteriousResistor(10)
    r7.current = 0.01
    print(f'이전: {r7.voltage:.2f}')
    r7.ohms
    print(f'이후: {r7.voltage:.2f}')
    
    >>>
    이전: 0.00
    이후: 0.10
    ```
    
- `@property` 는 하위 클래스 사이에서만 공유됨
    - 서로 관련 없는 클래스 사이에서는 공유 불가
    - 재사용 가능한 프로퍼티 로직을 구현할 때는 디스크립터를 제공함
    - 디스크립터는 다음 장에서 설명