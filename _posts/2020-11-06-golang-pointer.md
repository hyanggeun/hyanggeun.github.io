---
title: Method Receiver에서 Pointer를 언제 붙여야 할까?
date: 2020-11-06 10:33:00 +0900
categories: [Programming, Go]
tags: [Go]
---

## 개요
go 언어로 개발을 하다 보면 Method Receiver에 언제 포인터를 붙이고, 붙이지 말아야할지 헷갈릴 때가 있다. 
한번 정리할 필요성을 느껴서 글로 정리해본다. 
먼저 문제를 풀어보며 설명을 이어나가도록 하겠다. 

### Method Receiver에 포인터가 꼭 필요한 곳은?
문제: 
age를 변수로 가지고 있는 Person이라는 구조체가 있다. 또한 이 구조체는 age의 setter, getter인 setAge, getAge Method Receiver를 가지고 있다. 이 때 setter, getter중 *꼭* 포인터를 붙여야하는 함수는 무엇인가?(단, 직접 구조체의 변수를 수정하는 것은 사용하지 않는다고 가정하자.)

정답: setter인 setAge

이유를 생각해보면 간단하다. 포인터는 **주소값**를 저장한다. 그렇기 때문에 Method Receiver에 포인터를 붙인다면 **call by reference**, 포인터를 붙이지 않는다면 **call by value**로 값을 받게 된다. 반면 getter는 단순히 struct의 주소의 값을 변경하지 않고, 그 값만 가져오기 때문에 call by value로 값을 가져와도 무방하다. 



Wrong Answer Code
```go
type Person struct {
	age  int
}

func (p Person) setAge(age int) {
	p.age = age
}

func (p Person) getAge() int {
	return p.age
}

func main(){
    p := Person{age: 3}
    p.setAge(10)
    fmt.Println(p.getAge)

    //OUTPUT
    //3
}
```
Answer Code (Setter에 포인터를 붙이는 경우)
```go
type Person struct {
	age  int
}

func (p *Person) setAge(age int) {
	p.age = age
}

func (p Person) getAge() int {
	return p.age
}

func main(){
    p := Person{age: 3}
    p.setAge(10)
    fmt.Println(p.getAge)

    //OUTPUT
    //10
}
```
어느 글에서는 Method Receiver에 포인터를 붙이는 이유에 대해 다음과 같이 정리해 놓았다. 
    
    * receiver의 값을 변경하고자 할 때(단순히 읽기가 아닌 쓰기 작업도 같이)
    * struct의 크기가 커서 deep copy 비용이 클 때 
    * 코드 일관성(option): 어떤 함수가 포인터 receiver를 사용하는 경우 일관성을 줄 때 
### Priority Queue 코드를 통해서 알아보는 Pointer Receiver
위에 문제를 풀어보며 대충 어떨 때 포인터를 붙여야 하는지는 이해가 되었을 것이다. 그렇다면 실전으로 넘어가보자.

go에서는 Priority 큐를 구현하기 위해서 ```heap.Interface```, ```sort.Interface``` 2개의 Interface를 구현하면 된다.(현재는 Priority Queue가 중요한 것은 아니니 자세한 내용은 넘어가도록 하자.)
```go
type Interface interface {
    sort.Interface
	Push(x interface{})
	Pop() interface{} 
}

//sort.Interface
type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}
```
Kubernetes client-go/cache 패키지의 workqueue에서도 Priority Queue를 구현하고 있는데, 여기서 포인터에 대한 이해도를 높일수 있었다. 같이 한번 살펴보도록 하자
```go

type waitFor struct {
	data    t
	readyAt time.Time
	// index in the priority queue (heap)
	index int
}

type waitForPriorityQueue []*waitFor

func (pq waitForPriorityQueue) Len() int {
	return len(pq)
}
func (pq waitForPriorityQueue) Less(i, j int) bool {
	return pq[i].readyAt.Before(pq[j].readyAt)
}

func (pq waitForPriorityQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}

func (pq *waitForPriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*waitFor)
	item.index = n
	*pq = append(*pq, item)
}

func (pq *waitForPriorityQueue) Pop() interface{} {
	n := len(*pq)
	item := (*pq)[n-1]
	item.index = -1
	*pq = (*pq)[0:(n - 1)]
	return item
}
``` 

위의 코드를 보고 궁금한점이 2가지정도가 생길 수 있다. 
1. 왜 ```waitForPriorityQueue``` Slice의 ```waitFor```구조체에는 Pointer Receiver를 사용했을까? 
2. Slice에 어떤 값을 append할 때는 무조건 pointer를 사용해야 하는가? 

1번 궁금증부터 해결해보곘다. 
이 궁금증에 대한 대답은 Push, Pop 함수에 있다. Pop 함수를 한번 살펴보자. 
```go
func (pq *waitForPriorityQueue) Pop() interface{} {
	n := len(*pq)
	item := (*pq)[n-1]
	item.index = -1
	*pq = (*pq)[0:(n - 1)]
	return item
}
```
여기서 ```item := (*pq)[n-1]``` 부분을 유심히 보면 포인터로 한 이유가 명확해진다. 만약 ```waitFor```를 포인터로 하지 않았다면 item은 새로운 메모리에 할당이 되고,  ```waitForPriorityQueue``` Slice안의 값은 변경되지 않고, 기존 값으로 남아 있을 것이다. 

2번 궁금증을 해결해보자. 

goland에서 Method Receiver안에서 slice에 append 함수나, slicing을 적용하려고 한다면 ```Assignment to method receiver doesn't propagate to other calls``` 이런 권고 사항이 뜬다. 즉 Call By Value로 받은 Method Receiver안에서 값을 변경한다면 그 값은 함수 외부로 전달되지 않는다. 그러므로 Call By Reference인 Pointer Receiver로 전달하여 Slice의 값을 변경해야 한다. 


## 마치며
Method Receiver를 어느 경우에 포인터로 넘기고, 넘기지 않아야 하는지 간단하게나마 이해하는 시간이였다. 