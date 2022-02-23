Examples
========

1. [Reactive values](#reactive-values)
1. [Simple UI bindings](#simple-ui-bindings)
1. [Automatic input validation](#automatic-input-validation)
1. [more examples](../RxExample)
1. [Playgrounds](Playgrounds.md)

## Reactive values

Во-первых, давайте начнем с некоторого императивного кода.
Цель этого примера состоит в том, чтобы привязать идентификатор `c` к значению, вычисляемому из `a` и `b`, если выполняется какое-то условие.

Вот императивный код, который вычисляет значение `c`:

```swift
// это стандартный императивный код
var c: String
var a = 1       // это только присвоит значение «1» для `a` один раз
var b = 2       // это только присвоит значение «2» для `b` один раз this will only assign the value `2` to `b` once

if a + b >= 0 {
    c = "\(a + b) is positive" // this will only assign the value to `c` once
}
```

Значение `c` теперь равно `3 положительное`. Однако, если мы изменим значение `a` на `4`, `c` все равно будет содержать старое значение.


```swift
a = 4            // `c` все равно будет равно "3 положительно", что нехорошо
                 // мы хотим, чтобы `c` было равно "6 положительно", так как 4 + 2 = 6
```


Это нежелательное поведение.

Это улучшенная логика с использованием RxSwift:

```swift
let a /*: Observable<Int>*/ = BehaviorRelay(value: 1)   // a = 1
let b /*: Observable<Int>*/ = BehaviorRelay(value: 2)   // b = 2

// Объединяет последние значения реле `a` и `b`, используя `+`

let c = Observable.combineLatest(a, b) { $0 + $1 }
	.filter { $0 >= 0 }               // если `a b >= 0` истинно, `a + b` передается оператору карты                                  
	.map { "\($0) is positive" }      // сопоставляет `a + b` с "\(a + b) is positive" 

// Поскольку начальные значения равны a = 1 и b = 2
// 1 + 2 = 3, что >= 0, поэтому `c` изначально равно "3 is positive" 

// Чтобы получить значения из Rx `Observable` `c`, подпишитесь на значения из `c`.
// `subscribe(onNext:)` означает подписку на next (свежие) значения `c`.
// Это также включает в себя начальное значение "3 is positive".
c.subscribe(onNext: { print($0) }) // печатает: "3 is positive"

// Теперь давайте увеличим значение `a`
a.accept(4) // печатает: "6 is positive"
// Сумма последних значений `4` и `2` теперь равна `6`.
// Так как это `>= 0`, оператор `map` выдает "6 is positive"
// и этот результат "присваивается" `c`.
// Поскольку значение `c` изменилось, будет вызван `{ print($0) }`,
// и будет напечатано "6 is positive".

// Теперь давайте изменим значение `b`
b.accept(-8) // doesn't print anything
// Сумма последних значений `4 + (-8)` равна `-4`.
// Так как это не `>= 0`, `map` не выполняется.
// Это означает, что `c` все еще содержит "6 is positive"
// Так как `c` не был обновлен, новое "next" значение не было создано,
// и `{ print($0) }` не будут вызываться.
```

## Simple UI bindings

* Вместо привязки к Relay давайте привяжем значения `UITextField` с помощью свойства `rx.text`.
* Затем `преобразуйте` `String` в `Int` и определите, является ли число простым, используя асинхронный API.
* Если текст изменен до завершения асинхронного вызова, новый асинхронный вызов заменит его через `concat`.
* Привязать результаты к `UILabel`.

```swift
let subscription/*: Disposable */ = primeTextField.rx.text.orEmpty // тип Observable<String>
            .map { WolframAlphaIsPrime(Int($0) ?? 0) }             // тип Observable<Observable<Prime>>
            .concat()                                              // тип Observable<Prime>
            .map { "number \($0.n) is prime? \($0.isPrime)" }      // тип Observable<String>
            .bind(to: resultLabel.rx.text)                         // возвращаем Disposable, который можно использовать для отмены привязки всего

// Это установит для `resultLabel.text` значение "number 43 is prime? true" после
// вызов сервера завершен. Вы вручную инициируете управляющее событие, поскольку они
// события UIKit, которые RxCocoa наблюдает внутренне.
primeTextField.text = "43"
primeTextField.sendActions(for: .editingDidEnd)

// ...

// чтобы все развязать, просто вызываем 
subscription.dispose()
```

Все операторы, используемые в этом примере, являются теми же операторами, что и в первом примере с реле. В этом нет ничего особенного.

## Automatic input validation

Если вы новичок в Rx, следующий пример, вероятно, поначалу покажется вам немного ошеломляющим. Однако здесь мы демонстрируем, как код RxSwift выглядит в реальном мире.

Этот пример содержит сложную логику проверки асинхронного пользовательского интерфейса с уведомлениями о ходе выполнения.
Все операции отменяются в момент освобождения `disposeBag`.

Давайте попробуем.

```swift
enum Availability {
    case available(message: String)
    case taken(message: String)
    case invalid(message: String)
    case pending(message: String)

    var message: String {
        switch self {
        case .available(let message),
             .taken(let message),
             .invalid(let message),
             .pending(let message): 

             return message
        }
    }
}

// связываем значения элементов управления UI напрямую
// используем имя пользователя из `usernameOutlet` в качестве источника значений имени пользователя
self.usernameOutlet.rx.text
    .map { username -> Observable<Availability> in

        // synchronous validation, nothing special here
        guard let username = username, !username.isEmpty else {
            // Convenience for constructing synchronous result.
            // In case there is mixed synchronous and asynchronous code inside the same
            // method, this will construct an async result that is resolved immediately.
            return Observable.just(.invalid(message: "Username can't be empty."))
        }

        // ...

        // User interfaces should probably show some state while async operations
        // are executing.
        let loadingValue = Availability.pending(message: "Checking availability ...")

        // This will fire a server call to check if the username already exists.
        // Its type is `Observable<Bool>`
        return API.usernameAvailable(username)
          .map { available in
              if available {
                  return .available(message: "Username available")
              }
              else {
                  return .taken(message: "Username already taken")
              }
          }
          // use `loadingValue` until server responds
          .startWith(loadingValue)
    }
// Since we now have `Observable<Observable<Availability>>`
// we need to somehow return to a simple `Observable<Availability>`.
// We could use the `concat` operator from the second example, but we really
// want to cancel pending asynchronous operations if a new username is provided.
// That's what `switchLatest` does.
    .switchLatest()
// Now we need to bind that to the user interface somehow.
// Good old `subscribe(onNext:)` can do that.
// That's the end of `Observable` chain.
    .subscribe(onNext: { [weak self] validity in
        self?.errorLabel.textColor = validationColor(validity)
        self?.errorLabel.text = validity.message
    })
// This will produce a `Disposable` object that can unbind everything and cancel
// pending async operations.
// Instead of doing it manually, which is tedious,
// let's dispose everything automagically upon view controller dealloc.
    .disposed(by: disposeBag)
```

Это не становится проще, чем это. В репозитории есть [больше примеров](../RxExample), так что не стесняйтесь проверить их.
Они включают примеры того, как использовать Rx в контексте шаблона MVVM или без него.
