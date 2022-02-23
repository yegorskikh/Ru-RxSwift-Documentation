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
var b = 2       // это только присвоит значение «2» для `b` один раз

if a + b >= 0 {
    c = "\(a + b) is positive" // это присвоит значение `c` только один раз
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
b.accept(-8) // ничего не печатает
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

        // синхронная проверка, здесь ничего особенного
        guard let username = username, !username.isEmpty else {

	    // Удобство построения синхронного результата.
            // В случае, если внутри одного и того же кода смешанный синхронный и асинхронный
            // этот метод создаст асинхронный результат, который обрабатывается немедленно.
            return Observable.just(.invalid(message: "Username can't be empty."))
        }

        // ...

        // Пользовательские интерфейсы, вероятно, должны отображать некоторое состояние во время асинхронных операций
        // выполняются.
        let loadingValue = Availability.pending(message: "Checking availability ...")

	// Это вызовет вызов сервера, чтобы проверить, существует ли уже имя пользователя.
        // Его тип `Observable<Bool>`
        return API.usernameAvailable(username)
          .map { available in
              if available {
                  return .available(message: "Username available")
              }
              else {
                  return .taken(message: "Username already taken")
              }
          }
          // используйте `loadingValue`, пока сервер не ответит
          .startWith(loadingValue)
    }

// Так как теперь у нас есть `Observable<Observable<Availability>>`
// нам нужно как-то вернуться к простому `Observable<Availability>`.
// Мы могли бы использовать оператор `concat` из второго примера, но на самом деле мы
// хотим отменить ожидающие асинхронные операции, если указано новое имя пользователя.
// Вот что делает `switchLatest`.
    .switchLatest()
    
// Теперь нам нужно как-то связать это с пользовательским интерфейсом.
// Старый добрый `subscribe(onNext:)` может сделать это.
// Это конец цепочки `Observable`.
    .subscribe(onNext: { [weak self] validity in
        self?.errorLabel.textColor = validationColor(validity)
        self?.errorLabel.text = validity.message
    })
    
// Это создаст объект `Disposable`, который может отвязать все и отменить
// ожидающие асинхронные операции.
// Вместо того, чтобы делать это вручную, что утомительно,
// давайте удалим все автоматически при освобождении контроллера представления.
    .disposed(by: disposeBag)
```

Это не становится проще, чем это. В репозитории есть [больше примеров](../RxExample), так что не стесняйтесь проверить их.
Они включают примеры того, как использовать Rx в контексте шаблона MVVM или без него.
