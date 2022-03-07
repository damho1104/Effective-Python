# Effective-Python 스터디 프로젝트


* [『파이썬 코딩의 기술』](http://book.naver.com/bookdb/book_detail.nhn?bid=10382589) 의 Better Way 를 정리 예정
* 자기가 부족하다고 생각되는 내용 위주로 정리 예정, 아는 내용의 경우 간단하게 요약 혹은 코드로 표현 예정  
<br>

## Sample Code Reference  
* https://github.com/bslatkin/effectivepython

<br>

## 목차


### 1장 파이썬답게 생각하기  
  
[1. 사용중인 파이썬 버전을 알아두라](./summary/BetterWay01.md)  
[2. PEP 8 Style 가이드를 따르라](./summary/BetterWay02.md)  
[3. bytes 와 str의 차이를 알아두라](./summary/BetterWay03.md)  
[4. C 스타일 형식 문자열을 str.format과 쓰기보다는 f-문자열을 통한 인터폴레이션을 사용하라](./summary/BetterWay04.md)  
[5. 복잡한 식을 쓰는 대신 도우미 함수를 작성하라](./summary/BetterWay05.md)  
[6. 인덱스를 사용하는 대신 대입을 사용해 데이터를 언패킹하라](./summary/BetterWay06.md)  
[7. range 보다는 enumerate를 사용하라](./summary/BetterWay07.md)  
[8. 여러 이터레이터에 대해 나란히 루프를 수행하려면 zip을 사용해라](./summary/BetterWay08.md)  
[9. for 나 while 루프 뒤에 else 블록을 사용하지 말라](./summary/BetterWay09.md)  
[10. 대입식을 사용해 반복을 피하라](./summary/BetterWay10.md)  

### 2장 리스트와 딕셔너리
  
[11. 시퀀스를 슬라이싱하는 방법을 익혀라](./summary/BetterWay11.md)  
[12. 스트라이드와 슬라이스를 한 식에 함께 사용하지 말라](./summary/BetterWay12.md)  
[13. 슬라이싱보다는 나머지를 모두 잡아내는 언패킹을 사용하라](./summary/BetterWay13.md)  
[14. 복잡한 기준을 사용해 정렬할 때는 key 파라미터를 사용하라](./summary/BetterWay14.md)  
[15. 딕셔너리 삽입 순서에 의존할 때는 조심하라](./summary/BetterWay15.md)  
[16. in 을 사용하고 딕셔너리 키가 없을 때 KeyError를 처리하기 보다는 get 을 사용하라](./summary/BetterWay16.md)  
[17. 내부 상태에서 원소가 없는 경우를 처리할 때는 setdefault보다 defaultdict를 사용하라](./summary/BetterWay17.md)  
[18. \_\_missing\_\_ 을 사용해 키에 따라 다른 디폴트 값을 생성하는 방법을 알아두라](./summary/BetterWay18.md)  

### 3장 함수
  
[19. 함수가 여러 값을 반환하는 경우 절대로 4 값 이상을 언패킹하지 말라](./summary/BetterWay19.md)  
[20. None 을 반환하기보다는 예외를 발생시켜라](./summary/BetterWay20.md)  
[21. 변수 영역과 클로저의 상호작용 방식을 이해하라](./summary/BetterWay21.md)  
[22. 변수 위치 인자를 사용해 시각적인 잡음을 줄여라](./summary/BetterWay22.md)  
[23. 키워드 인자로 선택적인 기능을 제공하라](./summary/BetterWay23.md)  
[24. None 과 독스트링을 사용해 동적인 디폴트 인자를 지정하라](./summary/BetterWay24.md)  
[25. 위치로만 인자를 지정하게 하거나 키워드로만 인자를 지정하게 해서 함수 호출을 명확하게 만들라](./summary/BetterWay25.md)  
[26. functools.wrap 을 사용해 함수 데코레이터를 정의하라](./summary/BetterWay26.md)  

### 4장 컴프리헨션과 제너레이터
  
[27. map과 filter 대신 컴프리헨션을 사용하라](./summary/BetterWay27.md)  
[28. 컴프리헨션 내부에 제어 하위 식을 세 개 이상 사용하지 말라](./summary/BetterWay28.md)  
[29. 대입식을 사용해 컴프리헨션 안에서 반복 작업을 피해라](./summary/BetterWay29.md)    
[30. 리스트를 반환하기보다는 제너레이터를 사용하라](./summary/BetterWay30.md)    
[31. 인자에 대해 이터레이션할 때는 방어적이 돼라](./summary/BetterWay31.md)    
[32. 긴 리스트 컴프리헨션보다는 제너레이터 식을 사용하라](./summary/BetterWay32.md)    
[33. yield from 을 사용해 여러 제너레이터를 합성하라](./summary/BetterWay33.md)    
[34. send 로 제너레이터에 데이터를 주입하지 말라](./summary/BetterWay34.md)    
[35. 제너레이터 안에서 throw 로 상태를 변화시키지 말라](./summary/BetterWay35.md)    
[36. 이터레이터나 제너레이터를 다룰 때는 itertools 를 사용하라](./summary/BetterWay36.md)    

### 5장 클래스와 인터페이스
  
[37. 내장 타입을 여러 단계로 내포시키기보다는 클래스를 합성하라](./summary/BetterWay37.md)  
[38. 간단한 인터페이스의 경우 클래스 대신 함수를 받아라](./summary/BetterWay38.md)  
[39. 객체를 제너릭하게 구성하려면 @classmethod를 통한 다형성을 활용하라](./summary/BetterWay39.md)  
[40. super 로 부모 클래스를 초기화하라](./summary/BetterWay40.md)  
[41. 기능을 합성할 때는 믹스인 클래스를 사용하라](./summary/BetterWay41.md)  
[42. 비공개 애트리뷰트보다는 공개 애트리뷰트를 사용하라](./summary/BetterWay42.md)  
[43. 커스텀 컨테이너 타입은 collections.abc 를 상속하라](./summary/BetterWay43.md)  

### 6장 메타클래스와 애트리뷰트

[44. 세터와 게터 메서드 대신 평범한 애트리뷰트를 사용하라](./summary/BetterWay44.md)  
[45. 애트리뷰트를 리팩터링하는 대신 @property 를 사용하라](./summary/BetterWay45.md)  
[46. 재사용 가능한 @property 메소드를 만들려면 디스크립터를 사용하라](./summary/BetterWay46.md)  
[47. 지연 계산 애트리뷰트가 필요하면 \_\_getattr\_\_, \_\_getattribute\_\_, \_\_setattr\_\_ 을 사용하라](./summary/BetterWay47.md)  
[48. \_\_init_subclass\_\_ 를 사용해 하위 클래스를 검증하라](./summary/BetterWay48.md)  
[49. \_\_init_subclass\_\_ 를 사용해 클래스 확장을 등록하라](./summary/BetterWay49.md)  
[50. \_\_set_name\_\_ 으로 클래스 애트리뷰트를 표시하라](./summary/BetterWay50.md)  
[51. 합성 가능한 클래스 확장이 필요하면 메타클래스보다는 클래스 데코레이터를 사용하라](./summary/BetterWay51.md)  

### 7장 동시성과 병렬성

[52. 자식 프로세스를 관리하기 위해 subprocess 를 사용하라](./summary/BetterWay52.md)  
[53. 블로킹 I/O의 경우 스레드를 사용하고 병렬성을 피하라](./summary/BetterWay53.md)  
[54. 스레드에서 데이터 경합을 피하기 위해 Lock 을 사용하라](./summary/BetterWay54.md)  
[55. Queue 를 사용해 스레드 사이의 작업을 조율하라](./summary/BetterWay55.md)  
[56. 언제 동시성이 필요할지 인식하는 방법을 알아두라](./summary/BetterWay56.md)  
[57. 요구에 따라 팬아웃을 진행하려면 새로운 스레드를 생성하지 말라](./summary/BetterWay57.md)  
[58. 동시성과 Queue 를 사용하기 위해 코드를 어떻게 리펙터링해야 하는지 이해하라](./summary/BetterWay58.md)  
[59. 동시성을 위해 스레드가 필요한 경우에는 ThreadpoolExecutor 를 사용하라](./summary/BetterWay59.md)  
[60. I/O 를 할 때는 코루틴을 사용해 동시성을 높여라](./summary/BetterWay60.md)  
[61. 스레드에 사용한 I/O 를 어떻게 asyncio 로 포팅할 수 있는지 알아두라](./summary/BetterWay61.md)  
[62. asyncio 로 쉽게 옮겨갈 수 있도록 스레드와 코루틴을 함께 사용하라](./summary/BetterWay62.md)  
[63. 응답성을 최대로 높이려면 asyncio 이벤트 루프를 블록하지 말라](./summary/BetterWay63.md)  
[64. 진정한 병렬성을 살리려면 concurrent.futures 를 사용하라](./summary/BetterWay64.md)  

### 8장 강건성과 성능

[65. try/except/else/finally 의 각 블록을 잘 활용하라](./summary/BetterWay65.md)  
[66. 재사용 가능한 try/finally 동작을 원한다면 contextlib 과 with 문을 사용하라](./summary/BetterWay66.md)  
[67. 지역 시간에는 time 보다는 datetime 을 사용하라](./summary/BetterWay67.md)  
[68. copyreg 를 사용해 pickle을 더 신뢰성 있게 만들라](./summary/BetterWay68.md)  
[69. 정확도가 매우 중요한 경우에는 decimal 을 사용하라](./summary/BetterWay69.md)  
[70. 최적화하기 전에 프로파일링을 하라](./summary/BetterWay70.md)  
[71. 생산자-소비자 큐로 deque 를 사용하라](./summary/BetterWay71.md)  
[72. 정렬된 시퀀스를 검색할 때는 bisect 를 사용하라](./summary/BetterWay72.md)  
[73. 우선순위 큐로 heapq 를 사용하는 방법을 알아두라](./summary/BetterWay73.md)  
[74. bytes 를 복사하지 않고 다루려면 memoryview와 bytearray를 사용하라](./summary/BetterWay74.md)  

### 9장 테스트와 디버깅
  
[75. 디버깅 출력에는 repr 문자열을 사용하라](./summary/BetterWay75.md)  
[76. TestCase 하위 클래스를 사용해 프로그램에서 연관된 행동 방식을 검증하라](./summary/BetterWay76.md)  
[77. setUp, tearDown, setUpModule, tearDownModule 을 사용해 각각의 테스트를 격리하라](./summary/BetterWay77.md)  
[78. 목을 사용해 의존 관계가 복잡한 코드를 테스트하라](./summary/BetterWay78.md)  
[79. 의존 관계를 캡슐화해 모킹과 테스트를 쉽게 만들라](./summary/BetterWay79.md)  
[80. pdb 를 사용해 대화형으로 디버깅하라](./summary/BetterWay80.md)  
[81. 프로그램이 메모리를 사용하는 방식과 메모리 누수를 이해하기 위해 tracemalloc을 사용하라](./summary/BetterWay81.md)  

### 10장 협업
  
[82. 커뮤니티에서 만든 모듈을 어디서 찾을 수 있는지 알아두라](./summary/BetterWay82.md)  
[83. 가상 환경을 사용해 의존 관계를 격리하고 반복 생성할 수 있게 하라](./summary/BetterWay83.md)  
[84. 모든 함수, 클래스, 모듈에 독스트링을 작성하라](./summary/BetterWay84.md)  
[85. 패키지를 사용해 모듈을 체계화하고 안정적인 API를 제공하라](./summary/BetterWay85.md)  
[86. 배포 환경을 설정하기 위해 모듈 영역의 코드를 사용하라](./summary/BetterWay86.md)  
[87. 호출자를 API로부터 보호하기 위해 최상위 Exception 을 정의하라](./summary/BetterWay87.md)  
[88. 순환 의존성을 깨는 방법을 알아두라](./summary/BetterWay88.md)  
[89. 리팩터링과 마이그레이션 방법을 알려주기 위해 warning 을 사용하라](./summary/BetterWay89.md)  
[90. typing 과 정적 분석을 통해 버그를 없애라](./summary/BetterWay90.md)  

