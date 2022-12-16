---
title:  "Python - Hash(해시)"
excerpt: "파이썬의 Hash 총정리."

toc: true
toc_sticky: true
 
date: 2022-12-16
last_modified_at: 2022-12-16
---

## 해시(Hash)
파이썬에서 해시는 Dictionary라는 자료구조를 통해 해시를 제공합니다. 그리고 Dictionary는 dict 클래스에 구현되어 있습니다.

## 해시(Hash) 언제 사용하면 좋을까?
우리는 보통 동일한 연속된 데이터를 저장할 때 배열을 이용하곤 합니다. 그러나, 배열은 숫자 인덱스를 활용한다는 제약이 있습니다.
### 1. 리스트를 쓸 수 없을 때 
위와 같은 제약을 해결하기 위해 인덱스 값을 숫자가 아닌 다른 모든 타입 (문자열, 튜플)으로 사용할 수 있는데, 이는 딕셔너리 즉, 해시가 해결해줄 수 있습니다.

```python
player={"손흥민":7, "박지성":13}
print(player["손흥민"])
# 7
```
#### 2. 빠른 접근/ 탐색이 필요할 때
딕셔너리 함수의 시간복잡도는 대부분 O(1)이므로 아주 빠른 자료구조 입니다.

### 3. 집계가 필요할 때
원소의 개수를 세는 문제는 코딩테스트에서 많이 출제되는 문제입니다. 이 떄, 해시와, Collections 모듈의 Counter 클래스를 사용하면 아주 빠르게 문제를 풀 수 있습니다. 

### Dictionary 사용 방법
초기화
아래와 같은 방식으로 빈 딕셔너리를 선언할 수 있습니다.
- {}
- dict()

```python
dict1 = {}
dict2 = dict()
```

```python
Player = {
  '손흥민': 30,
  'weight':78,
  'height':180
}
#위와 같이 key:value 쌍으로 정의할 수 있습니다.
```

👀 **Get**
Dictionary에서 원소를 가져오는 방법은 두가지 입니다.
1. [key] 사용
2. get()

- get(key, x): 딕셔너리에 key가 없는 경우, x를 리턴해줘라.
- 딕셔너리를 카운터하는 (집계) 경우에 get 함수가 유용하게 사용됩니다.

```python
# [] 기호로 원소 가져오기 
player = {
  '손흥민': 30,
  'weight':78,
  'height':180
}
player['손흥민'] # 30 
#위와 같이 key:value 쌍으로 정의할 수 있습니다.
```

```python
# get 메서드로 원소 가져오기
# 딕셔너리에 해당 key가 없을 떄, key 에러 대신, 선언한 값을 가져오도록 한다. 
player = {
  '손흥민': 30,
  'weight':78,
  'height':180
}
player.get('박지성', 40) # 40
```
👀 **Set**
딕셔너리에 값을 집어 넣거나, 값을 업데이트 할 때 []를 사용.

```python
# 값 삽입
player = {
  '손흥민': 30,
}
player['조규성'] = 25
player # {'손흥민':30, '조규성':25}
```

```python
# 값 수정 1
player = {
  '손흥민': 30,
  '조규성': 25
}
player['손흥민'] = 29
player # {'손흥민':29, '조규성':25}
```

👀 **Delete**
딕셔너리에 특정 key의 값을 지울 때 아래와 같은 방법을 사용한다
- del dict_obj[key]
  - del은 키워드로써, 만약 딕셔너리에 해당 key가 존재하지 않으면 key 에러가 발생한다.
- pop(key, [,default])
  - pop은 메서드로써, pop 메서드는 key에 해당하는 value를 리턴한다. key가 없다면 두번쨰 파라미터인 default 값을 리턴합니다.
  - default를 설정하지 않았을 경우엔 KeyError 발생


```python
# 값 삭제 - del
player = {
  '손흥민': 30,
  '조규성': 25
}
del player['손흥민']
player #{'조규성':25}
```

```python
# 값 삭제 - del (key가 없는 경우)
player = {
  '손흥민': 30,
  '조규성': 25
}
del player['황의조']
'''
keyError
'''
```

```python
# 값 삭제 - pop
player = {
  '손흥민': 30,
  '조규성': 25
}
player.pop['손흥민', 10] # 30
```

```python
# 값 삭제 - pop (key가 없는 경우)
player = {
  '손흥민': 30,
  '조규성': 25
}
player.pop['황의조', 30] # 30
```

👀 **Iterate**
이번엔 코딩테스트에서 많이 활용할 수 있는 순회 방식이다.
딕셔너리로 순회하는 방식은 두가지가 존재한다.
1. key로만 순회하기
2. key, value 둘다 동시에 순회하기 - items() 사용

```python
# key로만 순회
player = {
  '손흥민': 30,
  '조규성': 25
}
for p in player:
  print(p)
'''
손흥민
조규성
'''
```

```python
# key - value 동시 순회
player = {
  '손흥민': 30,
  '조규성': 25
}
for p,age in player.items():
  print(p, age)
'''
손흥민 30
조규성 25
'''
```

### 그 밖에 딕셔너리 유용한 팁
1. 특정 Key가 딕셔너리에 있는지 없는지 조회할 때 - in 사용

```python
player = {
  '손흥민': 30,
  '조규성': 25
}
print('손흥민' in player) #True
print('손흥민' not in player) #False
```
