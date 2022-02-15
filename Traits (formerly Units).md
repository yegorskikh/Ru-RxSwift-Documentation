Traits (formerly Units)
=====

Этот документ попытается описать, что такое трейты, почему они являются полезной концепцией и как их использовать и создавать.

* [General](#general)
  * [Why](#why)
  * [How they work](#how-they-work)
* [RxSwift traits](#rxswift-traits)
  * [Single](#single)
    * [Creating a Single](#creating-a-single)
  * [Completable](#completable)
    * [Creating a Completable](#creating-a-completable)
  * [Maybe](#maybe)
    * [Creating a Maybe](#creating-a-maybe)
* [RxCocoa traits](#rxcocoa-traits)
  * [Driver](#driver)
      * [Why is it named Driver](#why-is-it-named-driver)
      * [Practical usage example](#practical-usage-example)
  * [Signal](#signal)
  * [ControlProperty / ControlEvent](#controlproperty--controlevent)


## General
### Why

Swift имеет мощную систему типов, которую можно использовать для улучшения правильности и стабильности приложений и сделать использование Rx более интуитивно понятным и простым.

Черты помогают сообщать и обеспечивать наблюдаемые свойства последовательности через границы интерфейса, а также предоставляют контекстуальное значение, синтаксический сахар и нацелены на более конкретные варианты использования по сравнению с необработанным наблюдаемым, который можно использовать в любом контексте. **По этой причине Черты совершенно необязательны. Вы можете свободно использовать необработанные последовательности Observable везде в своей программе, так как все основные API-интерфейсы RxSwift/RxCocoa поддерживают их.**

_**Примечание.** Некоторые трейты, описанные в этом документе (например, «Драйвер»), относятся только к проекту [RxCocoa](https://github.com/ReactiveX/RxSwift/tree/main/RxCocoa). , а некоторые являются частью общего проекта [RxSwift](https://github.com/ReactiveX/RxSwift). Однако те же самые принципы могут быть легко реализованы в других реализациях Rx, если это необходимо. Никакой частной магии API не требуется._

### How they work

Черты — это просто структура-оболочка с одним доступным только для чтения свойством последовательности Observable.

```swift
struct Single<Element> {
    let source: Observable<Element>
}

struct Driver<Element> {
    let source: Observable<Element>
}
...
```

Вы можете думать о них как о своего рода реализации шаблона строителя для последовательностей Observable. Когда Trait построен, вызов `.asObservable()` преобразует его обратно в ванильную наблюдаемую последовательность.

---

## RxSwift traits

### Single

Single — это разновидность Observable, которая вместо серии элементов всегда гарантированно выдает либо _один элемент_, либо _ошибку_.

* Выдает ровно один элемент или ошибку.
* Не разделяет побочных эффектов.

Одним из распространенных вариантов использования Single является выполнение HTTP-запросов, которые могут возвращать только ответ или ошибку, но Single можно использовать для моделирования любого случая, когда вам нужен только один элемент, а не бесконечный поток элементов.

#### Создание Single
Создание Single похоже на создание Observable.
Простой пример будет выглядеть так:

```swift
func getRepo(_ repo: String) -> Single<[String: Any]> {
    return Single<[String: Any]>.create { single in
        let task = URLSession.shared.dataTask(with: URL(string: "https://api.github.com/repos/\(repo)")!) { data, _, error in
            if let error = error {
                single(.error(error))
                return
            }

            guard let data = data,
                  let json = try? JSONSerialization.jsonObject(with: data, options: .mutableLeaves),
                  let result = json as? [String: Any] else {
                single(.error(DataError.cantParseJSON))
                return
            }

            single(.success(result))
        }

        task.resume()

        return Disposables.create { task.cancel() }
    }
}
```

После чего вы можете использовать его следующим образом:

```swift
getRepo("ReactiveX/RxSwift")
    .subscribe { event in
        switch event {
            case .success(let json):
                print("JSON: ", json)
            case .error(let error):
                print("Error: ", error)
        }
    }
    .disposed(by: disposeBag)
```

Or by using `subscribe(onSuccess:onError:)` as follows: 
```swift
getRepo("ReactiveX/RxSwift")
    .subscribe(onSuccess: { json in
                   print("JSON: ", json)
               },
               onError: { error in
                   print("Error: ", error)
               })
    .disposed(by: disposeBag)
```

Подписка предоставляет перечисление `SingleEvent`, которое может быть либо `.success`, содержащим элемент типа Single, либо `.error`. Никакие другие события не будут генерироваться за пределами первого.

Также возможно использовать `.asSingle()` в необработанной последовательности Observable, чтобы преобразовать ее в Single.

### Completable

Completable — это вариант Observable, который может только _complete_ или выдать error_. Гарантированно не излучает никаких элементов.

* Излучает ноль элементов.
* Выдает событие завершения или ошибку.
* Не разделяет побочных эффектов.

Полезным вариантом использования Completable может быть моделирование любого случая, когда нам важен только факт завершения операции, но не важен элемент, полученный в результате этого завершения.
Вы можете сравнить это с использованием `Observable<Void>`, который не может испускать элементы.

#### Creating a Completable
Создание Completable похоже на создание Observable. Простой пример будет выглядеть так:

```swift
func cacheLocally() -> Completable {
    return Completable.create { completable in
       // Store some data locally
       ...
       ...

       guard success else {
           completable(.error(CacheError.failedCaching))
           return Disposables.create {}
       }

       completable(.completed)
       return Disposables.create {}
    }
}
```
После чего вы можете использовать его следующим образом:
```swift
cacheLocally()
    .subscribe { completable in
        switch completable {
            case .completed:
                print("Completed with no error")
            case .error(let error):
                print("Completed with an error: \(error.localizedDescription)")
        }
    }
    .disposed(by: disposeBag)
```

Или с помощью `subscribe(onCompleted:onError:)` следующим образом:
```swift
cacheLocally()
    .subscribe(onCompleted: {
                   print("Completed with no error")
               },
               onError: { error in
                   print("Completed with an error: \(error.localizedDescription)")
               })
    .disposed(by: disposeBag)
```

Подписка предоставляет перечисление `CompletableEvent`, которое может быть либо `.completed`, указывающим на завершение операции без ошибок, либо `.error`. Никакие другие события не будут генерироваться за пределами первого.

### Maybe
A Maybe — это вариант Observable, который находится между Single и Completable. Он может либо выдать один элемент, завершиться без выдачи элемента, либо выдать ошибку.

**Примечание.** Любое из этих трех событий приведет к прекращению действия «Может быть», а это означает, что завершенное «возможно» не может также выдать элемент, а «может быть», выпустившее элемент, не может также отправить событие «Завершение».

* Выдает завершенное событие, отдельный элемент или ошибку.
* Не разделяет побочных эффектов.

Вы можете использовать Maybe для моделирования любой операции, которая **может** испускать элемент, но не обязательно **должна** испускать элемент.

#### Создание Maybe
Создание Maybe похоже на создание Observable. Простой пример будет выглядеть так:

```swift
func generateString() -> Maybe<String> {
    return Maybe<String>.create { maybe in
        maybe(.success("RxSwift"))

        // ИЛИ

        maybe(.completed)

        // ИЛИ

        maybe(.error(error))

        return Disposables.create {}
    }
}
```

После чего вы можете использовать его следующим образом:
```swift
generateString()
    .subscribe { maybe in
        switch maybe {
            case .success(let element):
                print("Completed with element \(element)")
            case .completed:
                print("Completed with no element")
            case .error(let error):
                print("Completed with an error \(error.localizedDescription)")
        }
    }
    .disposed(by: disposeBag)
```

Или с помощью `subscribe(onSuccess:onError:onCompleted:)` следующим образом:

```swift
generateString()
    .subscribe(onSuccess: { element in
                   print("Completed with element \(element)")
               },
               onError: { error in
                   print("Completed with an error \(error.localizedDescription)")
               },
               onCompleted: {
                   print("Completed with no element")
               })
    .disposed(by: disposeBag)
```

Также возможно использовать `.asMaybe()` в необработанной последовательности Observable, чтобы преобразовать ее в `Maybe`.

---



















## RxCocoa traits

### Driver

This is the most elaborate trait. Its intention is to provide an intuitive way to write reactive code in the UI layer, or for any case where you want to model a stream of data _Driving_ your application.

* Can't error out.
* Observe occurs on main scheduler.
* Shares side effects (`share(replay: 1, scope: .whileConnected)`).

#### Why is it named Driver

Its intended use case was to model sequences that drive your application.

E.g.
* Drive UI from CoreData model.
* Drive UI using values from other UI elements (bindings).
...


Like normal operating system drivers, in case a sequence errors out, your application will stop responding to user input.

It is also extremely important that those elements are observed on the main thread because UI elements and application logic are usually not thread safe.

Also, a `Driver` builds an observable sequence that shares side effects.

E.g.

#### Practical usage example

This is a typical beginner example.

```swift
let results = query.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
    }

results
    .map { "\($0.count)" }
    .bind(to: resultCount.rx.text)
    .disposed(by: disposeBag)

results
    .bind(to: resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```

The intended behavior of this code was to:
* Throttle user input.
* Contact server and fetch a list of user results (once per query).
* Bind the results to two UI elements: results table view and a label that displays the number of results.

So, what are the problems with this code?:
* If the `fetchAutoCompleteItems` observable sequence errors out (connection failed or parsing error), this error would unbind everything and the UI wouldn't respond any more to new queries.
* If `fetchAutoCompleteItems` returns results on some background thread, results would be bound to UI elements from a background thread which could cause non-deterministic crashes.
* Results are bound to two UI elements, which means that for each user query, two HTTP requests would be made, one for each UI element, which is not the intended behavior.

A more appropriate version of the code would look like this:

```swift
let results = query.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .observeOn(MainScheduler.instance)  // results are returned on MainScheduler
            .catchErrorJustReturn([])           // in the worst case, errors are handled
    }
    .share(replay: 1)                           // HTTP requests are shared and results replayed
                                                // to all UI elements

results
    .map { "\($0.count)" }
    .bind(to: resultCount.rx.text)
    .disposed(by: disposeBag)

results
    .bind(to: resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```

Making sure all of these requirements are properly handled in large systems can be challenging, but there is a simpler way of using the compiler and traits to prove these requirements are met.

The following code looks almost the same:

```swift
let results = query.rx.text.asDriver()        // This converts a normal sequence into a `Driver` sequence.
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .asDriver(onErrorJustReturn: [])  // Builder just needs info about what to return in case of error.
    }

results
    .map { "\($0.count)" }
    .drive(resultCount.rx.text)               // If there is a `drive` method available instead of `bind(to:)`,
    .disposed(by: disposeBag)              // that means that the compiler has proven that all properties
                                              // are satisfied.
results
    .drive(resultsTableView.rx.items(cellIdentifier: "Cell")) { (_, result, cell) in
        cell.textLabel?.text = "\(result)"
    }
    .disposed(by: disposeBag)
```

So what is happening here?

This first `asDriver` method converts the `ControlProperty` trait to a `Driver` trait.

```swift
query.rx.text.asDriver()
```

Notice that there wasn't anything special that needed to be done. `Driver` has all of the properties of the `ControlProperty` trait, plus some more. The underlying observable sequence is just wrapped as a `Driver` trait, and that's it.

The second change is:

```swift
.asDriver(onErrorJustReturn: [])
```

Any observable sequence can be converted to `Driver` trait, as long as it satisfies 3 properties:
* Can't error out.
* Observe on main scheduler.
* Sharing side effects (`share(replay: 1, scope: .whileConnected)`).

So how do you make sure those properties are satisfied? Just use normal Rx operators. `asDriver(onErrorJustReturn: [])` is equivalent to following code.

```swift
let safeSequence = xs
  .observeOn(MainScheduler.instance)        // observe events on main scheduler
  .catchErrorJustReturn(onErrorJustReturn)  // can't error out
  .share(replay: 1, scope: .whileConnected) // side effects sharing

return Driver(raw: safeSequence)            // wrap it up
```

The final piece is using `drive` instead of using `bind(to:)`.

`drive` is defined only on the `Driver` trait. This means that if you see `drive` somewhere in code, that observable sequence can never error out and it observes on the main thread, which is safe for binding to a UI element.

Note however that, theoretically, someone could still define a `drive` method to work on `ObservableType` or some other interface, so to be extra safe, creating a temporary definition with `let results: Driver<[Results]> = ...` before binding to UI elements would be necessary for complete proof. However, we'll leave it up to the reader to decide whether this is a realistic scenario or not.

### Signal

A `Signal` is similar to `Driver` with one difference, it does **not** replay the latest event on subscription, but subscribers still share the sequence's computational resources.

It can be considered a builder pattern to model Imperative Events in a Reactive way as part of your application.

A `Signal`:

* Can't error out.
* Delivers events on Main Scheduler.
* Shares computational resources (`share(scope: .whileConnected)`).
* Does NOT replay elements on subscription.

## ControlProperty / ControlEvent

### ControlProperty

Trait for `Observable`/`ObservableType` that represents a property of UI element.
 
Sequence of values only represents initial control value and user initiated value changes. Programmatic value changes won't be reported.

It's properties are:

- it never fails
- `share(replay: 1)` behavior
    - it's stateful, upon subscription (calling subscribe) last element is immediately replayed if it was produced
- it will `Complete` sequence on control being deallocated
- it never errors out
- it delivers events on `MainScheduler.instance`

The implementation of `ControlProperty` will ensure that sequence of events is being subscribed on main scheduler (`subscribeOn(ConcurrentMainScheduler.instance)` behavior).

#### Practical usage example

We can find very good practical examples in the `UISearchBar+Rx` and in the `UISegmentedControl+Rx`:

```swift 
extension Reactive where Base: UISearchBar {
    /// Reactive wrapper for `text` property.
    public var value: ControlProperty<String?> {
        let source: Observable<String?> = Observable.deferred { [weak searchBar = self.base as UISearchBar] () -> Observable<String?> in
            let text = searchBar?.text
            
            return (searchBar?.rx.delegate.methodInvoked(#selector(UISearchBarDelegate.searchBar(_:textDidChange:))) ?? Observable.empty())
                    .map { a in
                        return a[1] as? String
                    }
                    .startWith(text)
        }

        let bindingObserver = Binder(self.base) { (searchBar, text: String?) in
            searchBar.text = text
        }
        
        return ControlProperty(values: source, valueSink: bindingObserver)
    }
}
```

```swift
extension Reactive where Base: UISegmentedControl {
    /// Reactive wrapper for `selectedSegmentIndex` property.
    public var selectedSegmentIndex: ControlProperty<Int> {
        value
    }
    
    /// Reactive wrapper for `selectedSegmentIndex` property.
    public var value: ControlProperty<Int> {
        return UIControl.rx.value(
            self.base,
            getter: { segmentedControl in
                segmentedControl.selectedSegmentIndex
            }, setter: { segmentedControl, value in
                segmentedControl.selectedSegmentIndex = value
            }
        )
    }
}
```

### ControlEvent

Trait for `Observable`/`ObservableType` that represents an event on a UI element.

It's properties are:

- it never fails
- it won't send any initial value on subscription
- it will `Complete` sequence on control being deallocated
- it never errors out
- it delivers events on `MainScheduler.instance`

The implementation of `ControlEvent` will ensure that sequence of events is being subscribed on main scheduler (`subscribeOn(ConcurrentMainScheduler.instance)` behavior).

#### Practical usage example

This is a typical case example in which you can use it:

```swift
public extension Reactive where Base: UIViewController {
    
    /// Reactive wrapper for `viewDidLoad` message `UIViewController:viewDidLoad:`.
    public var viewDidLoad: ControlEvent<Void> {
        let source = self.methodInvoked(#selector(Base.viewDidLoad)).map { _ in }
        return ControlEvent(events: source)
    }
}
```

And in the `UICollectionView+Rx` we can found it in this way:

```swift

extension Reactive where Base: UICollectionView {
    
    /// Reactive wrapper for `delegate` message `collectionView:didSelectItemAtIndexPath:`.
    public var itemSelected: ControlEvent<IndexPath> {
        let source = delegate.methodInvoked(#selector(UICollectionViewDelegate.collectionView(_:didSelectItemAt:)))
            .map { a in
                return a[1] as! IndexPath
            }
        
        return ControlEvent(events: source)
    }
}
```