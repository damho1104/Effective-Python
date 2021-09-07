# Effective-Python 스터디 프로젝트


* [『파이썬 코딩의 기술』](http://book.naver.com/bookdb/book_detail.nhn?bid=10382589) 의 Better Way 를 정리 예정
* 자기가 부족하다고 생각되는 내용 위주로 정리 예정, 아는 내용의 경우 간단하게 요약 혹은 코드로 표현 예정

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
