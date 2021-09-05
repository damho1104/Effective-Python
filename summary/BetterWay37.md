# 37. 내장 타입을 여러 단계로 내포시키기보다는 클래스를 합성하라

## 1. 내장 타입을 사용한 예제

```python
class SimpleGradebook:
    def __init__(self):
        self._grades = {}
        
    def add_student(self, name):
        self._grades[name] = []
        
    def report_grade(self, name, score):
        self._grades[name].append(score)
        
    def average_grade(self, name):
        grades = self._grades[name]
        return sum(grades) / len(grades)

book = SimpleGradebook()
book.add_student('아이작 뉴턴')
book.report_grade('아이작 뉴턴', 90)
book.report_grade('아이작 뉴턴', 95)
book.report_grade('아이작 뉴턴', 85)

print(book.average_grade('아이작 뉴턴'))

>>>
90.0
```

- 과하게 확장하면서 꺠지기 쉬운 코드 작성 위험성 존재
- 전체 성적이 아닌 과목별 성적을 리스트로 저장하고 싶은 예제
    - `_grades` 딕셔너리를 학생별 딕셔너리로 변경, 값 딕셔너리 밑에 성적 리스트 매핑

```python
from collections import defaultdict
class BySubjectGradebook:
    def __init__(self):
        self._grades = {}                       # 외부# dict
        
    def add_student(self, name):
        self._grades[name] = defaultdict(list)  # 내부# dict

    def report_grade(self, name, subject, grade):
        by_subject = self._grades[name]
        grade_list = by_subject[subject]
        grade_list.append(grade)
        
    def average_grade(self, name):
        by_subject = self._grades[name]
        total, count = 0, 0
        for grades in by_subject.values():
            total += sum(grades)
            count += len(grades)
        return total / count

book = BySubjectGradebook()
book.add_student('알버트 아인슈타인')
book.report_grade('알버트 아인슈타인', '수학', 75)
book.report_grade('알버트 아인슈타인', '수학', 65)
book.report_grade('알버트 아인슈타인', '체육', 90)
book.report_grade('알버트 아인슈타인', '체육', 95)
print(book.average_grade('알버트 아인슈타인'))

>>>
81.25
```

- 각 점수의 가중치를 함께 저장해서 중간고사, 기말고사가 다른 쪽지 시험보다 성적에 더 큰 영향을 끼치게 하고 싶은 경우
    - 과목, 성적 매핑했던 것을 성적, 가중치 튜플 리스트로 변경

```python
class WeightedGradebook:
    def __init__(self):
        self._grades = {}
        
    def add_student(self, name):
        self._grades[name] = defaultdict(list)
        
    def report_grade(self, name, subject, score, weight):
        by_subject = self._grades[name]
        grade_list = by_subject[subject]
        grade_list.append((score, weight))

    def average_grade(self, name):
        by_subject = self._grades[name]

        score_sum, score_count = 0, 0
        for subject, scores in by_subject.items():
            subject_avg, total_weight = 0, 0
            
            for score, weight in scores:
                subject_avg += score * weight
                total_weight += weight
                
            score_sum += subject_avg / total_weight
            score_count += 1

        return score_sum / score_count

book = WeightedGradebook()
book.add_student('알버트 아인슈타인')
book.report_grade('알버트 아인슈타인', '수학', 75, 0.05)
book.report_grade('알버트 아인슈타인', '수학', 65, 0.15)
book.report_grade('알버트 아인슈타인', '수학', 70, 0.80)
book.report_grade('알버트 아인슈타인', '체육', 100, 0.40)
book.report_grade('알버트 아인슈타인', '체육', 85, 0.60)
print(book.average_grade('알버트 아인슈타인'))

>>>
80.25
```

- `report_grade` 에서는 성적 리슽가 튜플 인스턴스를 저장하도록 변경
- **단점**
    - `average_grade` 에서는 루프가 추가적으로 더 필요하게 되어 가독성 떨어짐
    - 복잡도 증가

## 2. 클래스를 활용한 리팩터링

- 원소가 2개인 경우의 tuple

```python
grades = []
grades.append((95, 0.45))
grades.append((85, 0.55))
total = sum(score * weight for score, weight in grades)
total_weight = sum(weight for _, weight in grades)
average_grade = total / total_weight
```

- 원소가 3개 이상인 tuple 을 사용해야 할 경우
    - `namedtuple` 사용
        - **`namedtuple`의 한계**
            - 디폴트 인자 지정 불가
                - 프로퍼티가 4~5개 보다 많아지면 `dataclasses` 내장 모듈을 사용하는 것을 추천
            - 모든 부분을 제어할 수 있는 상황이 아닌 경우 명시적으로 새로운 클래스를 정의하는 것이 나음

```python
from collections import namedtuple

Grade = namedtuple('Grade', ('score', 'weight'))
```

- 점수를 포함한 단일 과목을 표현하는 클래스

```python
class Subject:
    def __init__(self):
        self._grades = []
        
    def report_grade(self, score, weight):
        self._grades.append(Grade(score, weight))
        
    def average_grade(self):
        total, total_weight = 0, 0
        for grade in self._grades:
            total += grade.score * grade.weight
            total_weight += grade.weight
        return total / total_weight
```

- 한 학생이 수강하는 과목들을 쵸현하는 클래스

```python
class Student:
    def __init__(self):
        self._subjects = defaultdict(Subject)
        
    def get_subject(self, name):
        return self._subjects[name]
        
    def average_grade(self):
        total, count = 0, 0
        for subject in self._subjects.values():
            total += subject.average_grade()
            count += 1
        return total / count
```

- 모든 학생을 저장하는 컨테이너

```python
class Gradebook:
    def __init__(self):
        self._students = defaultdict(Student)
        
    def get_student(self, name):
        return self._students[name]
```

```python
book = Gradebook()
albert = book.get_student('알버트 아인슈타인')
math = albert.get_subject('수학')
math.report_grade(75, 0.05)
math.report_grade(65, 0.15)
math.report_grade(70, 0.80)
gym = albert.get_subject('체육')
gym.report_grade(100, 0.40)
gym.report_grade(85, 0.60)
print(albert.average_grade())

>>>
80.25
```
