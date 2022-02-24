Subjects
========

Все ведут себя точно так же, как описано [here](http://reactivex.io/documentation/subject.html)

Relays
======
RxRelay предоставляет три вида реле: `PublishRelay`, `BehaviorRelay` и `ReplayRelay`.
Они ведут себя точно так же, как и их параллельные `Subject`ы, с двумя изменениями:

- Relays никогда не завершаются.
- Relays никогда не выдают ошибок.

По сути, реле только генерируют события `.next` и никогда не завершаются.
