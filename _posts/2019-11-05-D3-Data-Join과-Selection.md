---
layout: post
title: D3 Data join과 Selection
excerpt: "D3의 핵심 개념인 Data Join과 Selection에 대해 정리한 것."
categories: [TIL]
comments: true
---

D3에서 가장 중요한 개념인 Data Join과 Selection에 대하여 다시 한 번 정리하면서 짚어본 것.

> 오명운님이 작성하신 셀렉션 작동원리(http://hanmomhanda.github.io/Docs/d3/How-Selections-Work.html)를 정리한 것이다.

# Data join과 Selection 이해하기
# 배열의 서브클래스
D3에서 *selection*은 배열의 서브클래스이다. `array.forEach`, `array.map`과 같은 네이티브 메서드도 상속받아 지원하지만, D3가 `selection.each`와 같은 편한 대체 메서드를 제공한다.

## 문서요소 그룹핑
selection은 group의 배열인데 각 group은 문서요소의 배열이다.

ex)
1. 단일 요소 선택 시(`d3.select`)
```javascript
var selection = d3.select("body");
```
![](/img/d3_selection.png)

2. 다중 요소 선택 시(`d3.selectAll`)
```javascript
var selection = d3.selectAll("div");
```
![](/img/d3_selection2.png)
1. 여러 개의 그룹 선택 시
```javascript
var selection = d3.selectAll("div").selectAll("p");
```
![](/img/d3_selection3.png)
```javascript
var selection = d3.selectAll("div").selectAll("p").selectAll("span");
```
![](/img/d3_selection4.png)

각 group 배열은 group 배열 내 모든 원소의 공통적인 부모 노드를 저장하는 parentNode 속성을 가지고 있다. 
부모 노드는 group 배열이 생성될 때 설정된다. 그래서, `d3.selectAll("div").selectAll("p")`를 호출하면, 부모가 div인 p 문서요소들을 원소로 하는 group 배열의 배열이 셀렉션으로 반환된다.

![](/img/d3_selection5.png)

## 그룹핑을 변경하지 않는 메서드들
`select`는 원래의 selection에 있는 그룹핑을 변경하지 않고 그대로 보존한다. `select` 메서드는 원래 selection의 데이터를 새로운 selection에 물려주지만, `selectAll`은 물려주지 않는다. 그렇기때문에 **data-join**이 필요하다.

`append` 메서드와 `insert` 메서드는 `select` 메서드를 래핑한 메서드로, `select`와 마찬가지로 그룹핑을 그대로 보존하고 원래의 selection에 있던 데이터를 새로운 selection에 물려준다. 

## Null 문서요소
group 배열은 문서요소가 없는 상태를 나타내기 위해 null을 포함할 수 있다.
null은 대부분의 처리에서 무시된다.

null이 발생되는 경우는 select 메서드가 해당 selector에 매칭되는 문서요소를 찾을 수 없을 때 발생할 수 있다. select 메서드는 그룹핑 구조를 그대로 보존해야하기 때문에 매칭되는 짝이 없는 곳은 null로 채운다.

ex) 4개의 section 노드 중 두 개만 aside를 가지고 있다면
```javascript
d3.selectAll("section").select("aside");
```
위 코드의 실행 결과는..
![](/img/d3_null.png)

## 데이터 바인딩
데이터는 셀렉션의 속성이 아닌, 셀렉션의 최말단 문서요소들의 속성이다.
각 문서요소의 `__data__`라는 속성에 할당된다. DOM에서 문서요소를 다시 select할 수 있으며, 선택된 해당 문서요소는 이전에 바인딩 된 데이터를 유지한다. 
=> 따라서, 데이터는 지속성이 있지만(persistent) 셀렉션은 지속성이 없다(transient).

<br>

**데이터가 바인딩될 수 있는 방법**
- `selection.data`를 통해 최말단 문서요소를 원소로 하는 group 배열에 조인
- `selection.datum`를 통해 개별 최말단 문서요소에 직접 할당
- `append`, `insert`, `select` 메서드를 통해 원래의 셀렉션으로부터 상속
