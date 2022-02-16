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

Это самая проработанная черта. Его цель состоит в том, чтобы предоставить интуитивно понятный способ написания реактивного кода на уровне пользовательского интерфейса или в любом случае, когда вы хотите смоделировать поток данных, управляющий вашим приложением.

* Не могу ошибиться.
* Наблюдение происходит в основном планировщике.
* Делится побочными эффектами (`share(replay: 1, scope: .whileConnected)`).

#### Почему он называется Driver

Его предполагаемый вариант использования заключался в моделировании последовательностей, управляющих вашим приложением.

Например.
* Пользовательский интерфейс привода из модели CoreData.
* Управляйте пользовательским интерфейсом, используя значения из других элементов пользовательского интерфейса (привязки).
...


Как и обычные драйверы операционной системы, в случае ошибки последовательности ваше приложение перестанет реагировать на ввод данных пользователем.

Также чрезвычайно важно, чтобы эти элементы наблюдались в основном потоке, потому что элементы пользовательского интерфейса и логика приложения обычно не являются потокобезопасными.

Кроме того, `Driver` создает наблюдаемую последовательность, которая имеет общие побочные эффекты.

Например.

#### Практический пример использования

Это типичный пример новичка.

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

Предполагаемое поведение этого кода заключалось в следующем:
* Дроссельный пользовательский ввод.
* Связаться с сервером и получить список пользовательских результатов (один раз за запрос).
* Привяжите результаты к двум элементам пользовательского интерфейса: представлению таблицы результатов и метке, отображающей количество результатов.

Итак, какие проблемы с этим кодом?:
* Если ошибка наблюдаемой последовательности `fetchAutoCompleteItems` (ошибка соединения или ошибка синтаксического анализа) приведет к отмене привязки, и пользовательский интерфейс больше не будет отвечать на новые запросы.
* Если `fetchAutoCompleteItems` возвращает результаты в каком-то фоновом потоке, результаты будут привязаны к элементам пользовательского интерфейса из фонового потока, что может вызвать недетерминированные сбои.
* Результаты привязаны к двум элементам пользовательского интерфейса, что означает, что для каждого пользовательского запроса будет выполнено два HTTP-запроса, по одному для каждого элемента пользовательского интерфейса, что не является предполагаемым поведением.

Более подходящая версия кода будет выглядеть так:

```swift
let results = query.rx.text
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .observeOn(MainScheduler.instance) // результаты возвращаются в MainScheduler
            .catchErrorJustReturn([])           // в худшем случае обрабатываются ошибки
    }
    .share(replay: 1)                          // HTTP-запросы передаются, а результаты воспроизводятся
                                                 // ко всем элементам пользовательского интерфейса

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

Убедиться, что все эти требования должным образом обрабатываются в больших системах, может быть сложно, но есть более простой способ использовать компилятор и трейты, чтобы доказать, что эти требования выполнены.

Следующий код выглядит почти так же:

```swift
let results = query.rx.text.asDriver()        // Это преобразует обычную последовательность в последовательность `Драйвер`  
    .throttle(.milliseconds(300), scheduler: MainScheduler.instance)
    .flatMapLatest { query in
        fetchAutoCompleteItems(query)
            .asDriver(onErrorJustReturn: [])  // Строителю просто нужна информация о том, что возвращать в случае ошибки.
    }

results
    .map { "\($0.count)" }
    .drive(resultCount.rx.text)             // Если вместо `bind(to:)` доступен метод `drive`,
    .disposed(by: disposeBag)              // это означает, что компилятор доказал, что все свойства  
                                              // удовлетворены.


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

Обратите внимание, что не было ничего особенного, что нужно было сделать. `Driver` имеет все свойства трейта `ControlProperty`, а также некоторые другие. Базовая наблюдаемая последовательность просто обернута как черта `Driver`, и все.

Второе изменение:

```swift
.asDriver(onErrorJustReturn: [])
```

Любая наблюдаемая последовательность может быть преобразована в черту «Драйвер», если она удовлетворяет трем свойствам:
* Не могу ошибиться.
* Наблюдайте за основным планировщиком.
* Совместное использование побочных эффектов (`share(replay: 1, scope: .whileConnected)`).

Так как же убедиться, что эти свойства удовлетворены? Просто используйте обычные операторы Rx. `asDriver(onErrorJustReturn: [])` эквивалентен следующему коду.

```swift
let safeSequence = xs
  .observeOn(MainScheduler.instance)        // наблюдаем за событиями в главном планировщике
  .catchErrorJustReturn(onErrorJustReturn)  // не может вывести ошибку
  .share(replay: 1, scope: .whileConnected) // обмен побочными эффектами

return Driver(raw: safeSequence)            // завершаем
```

Последняя часть использует `drive` вместо `bind(to:)`.

`драйв` определен только в признаке `Driver`. Это означает, что если вы видите `drive` где-то в коде, эта наблюдаемая последовательность никогда не может привести к ошибке, и она наблюдает в основном потоке, что безопасно для привязки к элементу пользовательского интерфейса.

Однако обратите внимание, что теоретически кто-то все еще может определить метод `drive` для работы с `ObservableType` или каким-либо другим интерфейсом, поэтому для дополнительной безопасности создайте временное определение с `let results: Driver<[Results]> = .. .` перед привязкой к элементам пользовательского интерфейса было бы необходимо для полного доказательства. Тем не менее, мы оставим читателю решать, является ли это реалистичным сценарием или нет.

### Signal

`Signal` похож на`Driver` с одним отличием: он **не** воспроизводит последнее событие по подписке, но подписчики по-прежнему разделяют вычислительные ресурсы последовательности.

Его можно рассматривать как шаблон построителя для моделирования императивных событий реактивным способом как часть вашего приложения.

`Signal`:

* Не могу ошибиться.
* Доставляет события на основной планировщик.
* Разделяет вычислительные ресурсы (`share(scope: .whileConnected)`).
* НЕ воспроизводит элементы по подписке.


## ControlProperty / ControlEvent

### ControlProperty

Признак для `Observable`/`ObservableType`, который представляет свойство элемента пользовательского интерфейса.
 
Последовательность значений представляет собой только начальное значение управления и изменения значений, инициированные пользователем. Сообщения об изменениях программной ценности не поступают.

Его свойства:

- никогда не подводит
- `share(replay: 1)` поведение
    - это состояние, при подписке (вызов подписки) последний элемент немедленно воспроизводится, если он был создан
- будет завершена последовательность при освобождении управления
- никогда не выдает ошибок
- он доставляет события в `MainScheduler.instance`

Реализация `ControlProperty` гарантирует, что последовательность событий подписывается в основном планировщике (поведение `subscribeOn(ConcurrentMainScheduler.instance)`).

#### Практический пример использования

Мы можем найти очень хорошие практические примеры в «UISearchBar Rx» и в «UISegmentedControl Rx»:

```swift 
extension Reactive where Base: UISearchBar {
    /// Реактивная оболочка для свойства `text`.
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
    /// Реактивная оболочка для свойства selectedSegmentIndex.
    public var selectedSegmentIndex: ControlProperty<Int> {
        value
    }
    
    /// Реактивная оболочка для свойства selectedSegmentIndex.
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

Признак для `Observable`/`ObservableType`, который представляет событие в элементе пользовательского интерфейса.

Его свойства:

- никогда не подводит
- он не будет отправлять начальное значение при подписке
- будет завершена последовательность при освобождении управления
- никогда не выдает ошибок
- он доставляет события в `MainScheduler.instance`

Реализация `ControlEvent` гарантирует, что последовательность событий подписывается в основном планировщике (поведение `subscribeOn(ConcurrentMainScheduler.instance)`).

#### Практический пример использования

Это типичный пример случая, в котором вы можете его использовать:

```swift
public extension Reactive where Base: UIViewController {
    
    /// Реактивная оболочка для сообщения `viewDidLoad` `UIViewController:viewDidLoad:`.
    public var viewDidLoad: ControlEvent<Void> {
        let source = self.methodInvoked(#selector(Base.viewDidLoad)).map { _ in }
        return ControlEvent(events: source)
    }
}
```

И в `UICollectionView Rx` мы можем найти это так:

```swift

extension Reactive where Base: UICollectionView {
    
    /// Реактивная оболочка для сообщения делегата collectionView:didSelectItemAtIndexPath:.
    public var itemSelected: ControlEvent<IndexPath> {
        let source = delegate.methodInvoked(#selector(UICollectionViewDelegate.collectionView(_:didSelectItemAt:)))
            .map { a in
                return a[1] as! IndexPath
            }
        
        return ControlEvent(events: source)
    }
}
```