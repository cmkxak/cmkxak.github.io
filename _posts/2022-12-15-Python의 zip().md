## zip()

여러개의 순회 가능한(Iterable)한 객체를 인자로 받고, 각 객체가 담고 있는 원소를 Tuple 형태로 차례대로 접근할 수 있는 반복자(Iterable)을 반환합니다.
zip() 함수를 이용하면, 양 측에 있는 데이터를 하나씩 차례로 짝을 지어줍니다.

```python
numbers = [1,2,3]
letters = ['A', 'B', 'C']
for pair in zip(numbers, letters):
  print(pair)
# (1,'A')
# (2,'B')
# (3,'C')
```

## 병렬 처리
zip() 함수를 활용 시, 여러 그룹의 데이터를 루프를 도는 것과 같이 동작 시킬 수 있습니다. 가변 인자를 받기 때문에 2개 이상의 인자를 넘겨 병렬 처리 할 수 있습니다.
코드를 통해 이해해봅시다.
```python
cur = "hot"
words = ["dog"]
for i in range(len(words)):
    for a,b in zip(cur, words[i]):
        print(a,b)
# h d
# o o
# t g
```
위와 같이 cur 리스트와 words 리스트의 원소들이 a,b 변수에 순차적으로 담겨져 출력되는 것을 알 수 있습니다.

## 사전 변환
zip() 함수를 사용하면, 두 개의 리스트나 터플 부터 쉽게 사전(dictionary)를 만들 수 있습니다. 키를 담고 있는 리스트와, 값을 담고 있는 리스트를 zip()함수로 묶어준 뒤, 이 결과를 다시 dict() 함수에 넘기면 됩니다.

```python
numbers = [7,13,20]
players = ["손흥민", "박지성", "이동국"]
dict(zip(numbers, players))
#{7: '손흥민', 13:'박지성', 20,'이동국'}
```

## 주의 사항
zip() 함수로 넘기는 인자의 길이가 다를 때는 주의해야 합니다. 왜냐하면 가장 짧은 인자를 기준으로 데이터가 엮이고 나머지는 버려지기 때문입니다.

```python
numbers = ["1","2","3"]
letters = ["B"]
list(zip(numbers, letters))
[('1', "B")]
```

### 참조 링크 : [파이썬의 zip() 내장 함수로 데이터 엮기](https://www.daleseo.com/python-zip/?q=cache:h7Uq8l_NO8MJ:https://www.daleseo.com/python-zip/&cd=1&hl=ko&ct=clnk&gl=kr)