## Effectful Handlers

이 튜토리얼은 부수효과가 있는 순수한 이벤트 핸들러를 어떻게 구현하는지 보여준다. 맞다! 이건 놀라운 주장이다.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
<!-- ## Table Of Contents -->

<!-- - [Events Happen](#events-happen) -->
<!-- - [Handling The Happening](#handling-the-happening) -->
<!-- - [Your Handling](#your-handling) -->
<!-- - [90% Solution](#90%25-solution) -->
<!-- - [Bad, Why?](#bad-why) -->
<!-- - [The 2nd Kind Of Problem](#the-2nd-kind-of-problem) -->
<!-- - [Effects And Coeffects](#effects-and-coeffects) -->
<!-- - [Why Does This Happen?](#why-does-this-happen) -->
<!-- - [Doing vs Causing](#doing-vs-causing) -->
<!-- - [Et tu, React?](#et-tu-react) -->
<!-- - [Pattern Structure](#pattern-structure) -->
<!-- - [Effects: The Two Step Plan](#effects-the-two-step-plan) -->
<!-- - [Step 1 Of Plan](#step-1-of-plan) -->
<!-- - [Another Example](#another-example) -->
<!-- - [The Coeffects](#the-coeffects) -->
<!-- - [Variations On A Theme](#variations-on-a-theme) -->
<!-- - [Summary](#summary) -->

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

### 이벤트 발생

디스패치를 할 때 이벤트가 "발생"한다.

그래서 아래 코드는 이벤트를 만든다:
```clj
(dispatch [:repair-ming-vase true])
```

이벤트는 일반적으로 외부 에이전트를 통해 생긴다: 사용자가 버튼을 클릭하거나 웹소켓으로 서버에서 메시지가
내려오는 경우

### 이벤트 사건을 다루기

디스패치가 되면 이벤트는 반드시 다뤄져야 한다 - 반드시 진행하거나 행동해야한다는 뜻이다.

이벤트는 원래 무언가를 바꾼다. 만약 애플리케이션이 이벤트가 진행되기 전에 어떤 상태에 있었다면 이벤트가
진행되고 나서 다른 상태로 바뀔 것이다.

상태 변경은 매우 바람직하다. 애플리케이션 상태 변경이 없다면 버튼 클릭이나 새로운 웹소켓 메시지 도착에 아무
반응을 할 수 없다. 변경이 없다면 애플리케이션은 꼼짝하지 않고 그냥 가만이 있는 것이다.

상태 변경은 애플리케이션이 어떻게 진행 할 것인지 - 어떻게 동작할 것인지에 대한 것이다. 유용하게!

반면 컨트롤 로직과 상태 변경은 애플리케이션에서 가장 복잡하고 에러에 취약한 부분이다.

### Your Handling

이 잠재적인 복잡성에 대한 논쟁을 돕기 위해 re-frame은 단순한 프로그래밍 모델을 제공한다.

이벤트를 다룰 함수와 이벤트 아이디를 연결해 `reg-event-db`를 부를 수 있다:
```clj
(re-frame.core/reg-event-db        ;; <-- 핸들러를 등록하라고 호출
    :set-flag                      ;; 이것이 event id
   (fn [db [_ new-value]]          ;; 이것은 핸들러 함수
      (assoc db :flag new-value)))
```

주어진 `id`의 이벤트를 처리할 수 있는 함수를 등록했다.

그리고 `fn`은 순수함수 일것이라고 기대한다. 첫번째 인자로 `app-db`와 두번째 인자로 이벤트(벡터)가
주어진다. 그리고 새로운 `app-db`를 제공하기를 기대한다. (리턴 값)

데이터가 들어오고 연산하고 데이터를 내보낸다. 순수하다.

### 90% Solution

이 패러다임은 90% 정도 해결책을 제공한다. 하지만 충분하지 않은 경우가 있다.

아래는 지저분한 10% 예제다. 작업이 끝난 것을 알기 위해 이 핸들러는 부수효과를 가진다:
```clj
(reg-event-db
   :my-event
   (fn [db [_ bool]]
       (dispatch [:do-something-else 3])    ;; 윽! 부수효과!
       (assoc db :send-spam new-val)))
```

핸들러 안에 있는 `dispatch`는 다른 이벤트를 진행시키기 위해 큐에 들어간다. 이것은 세상을 바꾼다.

확실하게 하기 위해 이코드는 동작하나. 이 핸들러는 새로운 버전의 `db`를 리턴하고 다음에 `dispatch`는
이 핸들러가 끝나고 나서 매우 짧은 시간에 스스로 비동기적으로 실행 될 것이다. 더블 틱이다.

그래서 결과가 잘 나온다. 하지만 순수하지 않다.

이것은 더 지저분한 예제다:
```clj
(reg-event-db
   :my-event
   (fn [db [_ a]]
       (GET "http://json.my-endpoint.com/blah"   ;; 더럽게 큰 부수효과
            {:handler       #(dispatch [:process-response %1])
             :error-handler #(dispatch [:bad-response %1])})
       (assoc db :flag true)))
```

다시 말하지만 이건 잘 동작한다. 하지만 더럽게 큰 부수효과가 자유롭게 하지 못한다. 이것은 마치 하얀 튤립 정원
앞에 거대한 몬스터 트럭이 있는 것과 같다.

### Bad, Why?

The moment we stop writing pure functions there are well documented
consequences:

  1. Cognitive load for the function's later readers goes up because they can no longer reason locally.
  2. Testing becomes more difficult and involves "mocking".  How do we test that the http GET above is
     using the right URL?  "mocking" should be mocked. It is a bad omen.
  3. And event replay-ability is lost.

Regarding the 3rd point above, a re-frame application proceeds step by step, like a reduce. From the README:

> at any one time, the value in app-db is the result of performing a reduce over the entire
> collection of events dispatched in the app up until that time. The combining
> function for this reduce is the set of registered event handlers.

Such a collection of events is replay-able which is a dream for debugging and testing. But only
when all the handlers are pure. Handlers with side-effects (like that HTTP GET, or the `dispatch`) pollute the
replay, inserting extra events into it, etc., which ruins the process.


### The 2nd Kind Of Problem

다음은 순수성의 또 다른 문제다:
```clj
(reg-event-db
   :load-localstore
   (fn [db _]
     (let [val (js->clj (.getItem js/localStorage "defaults-key"))]  ;; <-- 문제
       (assoc db :defaults val))))
```

이벤트 핸들러가 LocalStore로 부터 데이터를 얻어 오는 것을 볼 수 있다.
You'll notice the event handler obtains data from LocalStore.

비록 부수효과는 없지만 - 세상을 바꾸지 않는다 - 인자가 아닌 다른 곳으로 부터 데이터를 가져온다는 것은 순수하지
않다는 뜻이다.

### Effects And Coeffects

When striving for pure event handlers [there are two considerations](http://tomasp.net/blog/2014/why-coeffects-matter/):
  - **Effects** - what your event handler does to the world  (aka side-effects)
  - **Coeffects** - the data your event handler requires from the world in order
    to do its computation (aka [side-causes](http://blog.jenkster.com/2015/12/what-is-functional-programming.html))

우리는 두 문제 모두 해결책이 필요하다

### Why Does This Happen?

It is inevitable that, say, 10% of your event handlers have effects and coeffects.

They have to implement the control logic of your re-frame app, which
means dealing with the outside, mutative world of servers, databases,
window.location, LocalStore, cookies, etc.

There's just no getting away from living in a mutative world, which sounds pretty ominous. Is that it? Are we doomed to impurity?

Well, luckily a small twist in the tale makes a profound difference. We
will look at side-effects first. Instead of creating event handlers
which *do side-effects*, we'll instead get them to *cause side-effects*.

### Doing vs Causing

나는 이 이벤트 핸들러가 순수하다고 자랑스럽게 주장한다:
```clj
(reg-event-db
   :my-event
   (fn [db _]
      (assoc db :flag true)))
```

`db`값을 받고 계산 한 후에 `db`값을 리턴한다. coeffect나 effect가 없다. 이것은 순수하다!

Yes, all true, but ... this purity is only possible because re-frame is doing
the necessary side-effecting.

Wait on.  What "necessary side-effecting"?

Well, application state is stored in `app-db`, right?  And it is a ratom. And after
each event handler runs, it must be `reset!` to the newly returned
value. Notice `reset!`.   That, right there, is the "necessary side effecting".

We get to live in our ascetic functional world because re-frame is
looking after the "necessary side-effects" on `app-db`.

### Et tu, React?

Turns out it's the same pattern with Reagent/React.

We get to write a nice pure component, like:
```clj
(defn say-hi
  [name]
  [:div "Hello " name])
```
and Reagent/React mutates the DOM for us. The framework is looking
after the "necessary side-effects".

### Pattern Structure

Pause and look back at `say-hi`. I'd like you to view it through the
following lens:  it is a pure function which **returns a description
of the side-effects required**. It says: add a div element to the DOM.

Notice that the description is declarative. We don't tell React how to do it.

Notice also that it is data. Hiccup is just vectors and maps.

This is a big, important concept.  While we can't get away from certain side-effects, we can
program using pure functions which **describe side-effects, declaratively, in data** and
let the backing framework look after the "doing" of them. Efficiently. Discreetly.

Let's use this pattern to solve the side-effecting event-handler problem.

### Effects: The Two Step Plan

From here, two steps:
  1. Work out how event handlers can declaratively describe side-effects, in data.
  2. Work out how re-frame can do the "necessary side-effecting". Efficiently and discreetly.


### Step 1 Of Plan

So, how would it look if event handlers returned side-effects, declaratively, in data?

Here is an impure, side effecting handler:
```clj
(reg-event-db
   :my-event
   (fn [db [_ a]]
       (dispatch [:do-something-else 3])    ;; <-- Eeek, side-effect
       (assoc db :flag true)))
```

Here it is re-written so as to be pure:
```clj
(reg-event-fx                              ;; <1>
   :my-event
   (fn [{:keys [db]} [_ a]]                ;; <2>
      {:db  (assoc db :flag true)          ;; <3>
       :dispatch [:do-something-else 3]}))
```

Notes: <br>
*<1>* we're using `reg-event-fx` instead of `reg-event-db` to register  (that's `-db` vs `-fx`) <br>
*<2>* the first parameter is no longer just `db`.  It is a map from which
[we are destructuring db](http://clojure.org/guides/destructuring), i.e.
it is a map which contains a `:db` key. <br>
*<3>* The handler is returning a data structure (map) which describes two side-effects:
  - a change to application state, via the `:db` key
  - a further event, via the `:dispatch` key

Above, the impure handler **did** a `dispatch` side-effect, while the pure handler **described**
a `dispatch` side-effect.


### Another Example

The impure way:
```clj
(reg-event-db
   :my-event
   (fn [db [_ a]]
       (GET "http://json.my-endpoint.com/blah"   ;; dirty great big side-effect
            {:handler       #(dispatch [:process-response %1])
             :error-handler #(dispatch [:bad-response %1])})
       (assoc db :flag true)))
```

the pure, descriptive alternative:
```clj
(reg-event-fx
   :my-event
   (fn [{:keys [db]} [_ a]]
       {:http {:method :get
               :url    "http://json.my-endpoint.com/blah"
               :on-success  [:process-blah-response]
               :on-fail     [:failed-blah]}
        :db   (assoc db :flag true)}))
```

Again, the old way **did** a side-effect (Booo!) and the new way **describes**, declaratively,
in data, the side-effects required (Yaaa!).

More on side effects in a minute, but let's double back to coeffects.

### The Coeffects

So far we've written our new style `-fx` handlers like this:
```clj
(reg-event-fx
   :my-event
   (fn [{:keys [db]} event]   ;; <--  destructuring to get db
       { ... }))
```

It is now time to name that first argument:
```clj
(reg-event-fx
   :my-event
   (fn [cofx event]       ;;  <--- thy name be cofx
       { ... }))
```

When you use the `-fx` form of registration, the first argument
of your handler will be a map of coeffects which we name `cofx`.

In that map will be the complete set of "inputs" required by your function.  The complete
set of computational resources (data) needed to perform its computation. But how?
This will be explained in an upcoming tutorial, I promise, but for the moment,
take it as a magical given.

One of the keys in `cofx` will likely be `:db` and that will be the value of `app-db`.

Remember this impure handler from before:
```clj
(reg-event-db            ;;  a -db registration
 :load-localstore
 (fn [db _]              ;; db first argument
  (let [defaults (js->clj (.getItem js/localStorage "defaults-key"))]  ;; <--  Eeek!!
    (assoc db :defaults defaults))))
```

It was impure because it obtained an input from other than its arguments.
We'd now rewrite it as a pure handler, like this:
```clj
(reg-event-fx             ;; notice the -fx
   :load-localstore
   (fn [cofx  _]          ;; cofx is a map containing inputs
     (let [defaults (:local-store cofx)]  ;; <--  use it here
       {:db (assoc (:db cofx) :defaults defaults)})))  ;; returns effects map
```

So, by some magic, not yet revealed, LocalStore will be queried before
this handler runs and the required value from it will be placed into
`cofx` under the key `:local-store` for the handler to use.

That process leaves the handler itself pure because it only sources
data from arguments.


### Variations On A Theme

`-db` handlers and `-fx` handlers are conceptually the same. They only differ numerically.

`-db` handlers take __one__ coeffect called `db`, and they return only __one__  effect (db again).

Whereas `-fx` handlers take potentially __many__ coeffects (a map of them) and they return
potentially __many__ effects (a map of them). So, One vs Many.

Just to be clear, the following two handlers achieve the same thing:
```clj
(reg-event-db
   :set-flag
   (fn [db [_ new-value]]
      (assoc db :flag new-value)))
```
vs
```clj
(reg-event-fx
   :set-flag
   (fn [cofx [_ new-value]]
      {:db (assoc (:db cofx) :flag new-value)}))
```

Obviously the `-db` variation is simpler and you'd use it whenever you
can. The `-fx` version is more flexible, so it will sometimes have its place.


### Summary

90% of the time, simple `-db` handlers are the right tool to use.

But about 10% of the time, our handlers need additional inputs (coeffects) or they need to
cause additional side-effects (effects).  That's when you reach for `-fx` handlers.

`-fx` handlers allow us to return effects, declaratively in data.

In the next tutorial, we'll shine a light on `interceptors` which are
the mechanism by which  event handlers are executed. That knowledge will give us a springboard
to then, as a next step, better understand coeffects and effects. We'll soon be writing our own.

***

Previous:  [Infographic Overview](EventHandlingInfographic.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Up:  [Index](README.md)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
Next:  [Interceptors](Interceptors.md)
