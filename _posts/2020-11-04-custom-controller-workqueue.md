---
title: Kubernetes Custom Controller Workqueue
date: 2020-11-04 19:00:00 +0900
categories: [Kubernetes, Go]
tags: [Go, Kubernetes, client-go, controller]
---

## Workqueue란 
* Custom Controller에서 Object의 전달을 분리하기 위해서 만들어 놓은 큐
* Workqueue에서 Key를 통해 Object를 가져와 순차적으로 Handle하게 된다. 
* Workqueue는 
EventHandler로 등록(AddFunc, UpdateFunc, DeleteFunc)한 객체에 대해서 동작한다.

### Workqueue의 Interface
1. Add(item interface{})
2. Len() int
3. Get() (item interface{}, shutdown bool)
4. Done(item interface{})
5. ShutDown()
6. ShuttingDown() bool

### 중요한 Interface 구현체들
* 구현체를 보기 전에 미리 알아야될 2개의 map struct가 있다. 
    * processing (map[interface{}]interface{})
        * processing은 실제로 handle함수가 처리되고 있는지를 확인하는 Map이다 
        * **processing Map을 통해서 동일한 Object가 동시에 Handle되는 것을 막아준다.**
    * dirty (map[interface{}]interface{})
        * dirty는 queue에 들어온 모든 Object들을 저장하는 Map이다. 
        * **dirty를 통해서 Queue에 동일한 Object가 동시에 들어오는 것을 방지한다.**
* Add(item interface{})
    ```go
    func (q *Type) Add(item interface{}) {
	    q.cond.L.Lock()
	    defer q.cond.L.Unlock()
	    if q.shuttingDown {
		    return
	    }
	    if q.dirty.has(item) {
		    return
	    }

	    q.metrics.add(item)

	    q.dirty.insert(item)
	    if q.processing.has(item) {
		    return
	    }

	    q.queue = append(q.queue, item)
	    q.cond.Signal()
    }
    ```
    * Add함수에서는 item이 dirty에도 없고, processing 되고 있지도 않다면, queue에 item을 push하고 dirty Map에 item을 넣어주는 역할을 한다. 
* Get() (item interface{}, shutdown bool)
    ```go
    func (q *Type) Get() (item interface{}, shutdown bool) {
    
	q.cond.L.Lock()
    defer q.cond.L.Unlock()
    
	for len(q.queue) == 0 && !q.shuttingDown {
		q.cond.Wait()
	}
    
    if len(q.queue) == 0 {
		// We must be shutting down.
		return nil, true
	}

	item, q.queue = q.queue[0], q.queue[1:]

	q.metrics.get(item)

	q.processing.insert(item)
	q.dirty.delete(item)

	return item, false
    }
    ```
    * Get함수에서는 queue에서 item을 Pop하여, processing map에 넣어주고 dirty map을 지워주는 역할을 한다.
* Done(item interface{})
    ```go
    func (q *Type) Done(item interface{}) {
        q.cond.L.Lock()
        defer q.cond.L.Unlock()

        q.metrics.done(item)

        q.processing.delete(item)
        if q.dirty.has(item) {
            q.queue = append(q.queue, item)
            q.cond.Signal()
        }
    }
    ```
    * Done함수는 Handle이 처리되었을 때 필수적으로 호출되어야하는 함수이다. 
    * processing Map에서 Item을 지워주고, dirty가 남아있다면Handle 과정중에 같은 Object에 대해 Event가 발생하여 dirty Map에는 존재하지만, Processing이 들어온 순간에 되고 있어서 무시되었다는 뜻)다시 Queue에 넣어주고 다시 처리해주게 된다. 
