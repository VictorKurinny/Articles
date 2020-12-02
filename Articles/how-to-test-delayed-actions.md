# Тестируем отложенные вызовы

В этой статье рассмотрим возможность тестирования отложенных вызовов `DispatchQueue.asyncAfter(deadline: .now() + delay, execute: work)` с помощью быстрых тестов. Таким образом можно проверять поведение систем, а также взаимодействие пользователя с ними, с учетом течения времени. При этом используем только синхронные проверки.

## Методология
Все что мы не можем контролировать во время тестов нужно заменять фейковыми объектами, `DispatchQueue` не исключение. По мере того, как нужно тестировать те или иные ситуации в проекте, такой фейковый объект будет развиваться, обрастать нужными возможностями, хэлпер методами. Желательно иметь отдельные тесты для него. Они гарантируют правильную работу даже если удалили код из проекта который использовал ту или иную фичу. Я придерживаюсь следующей позиции: если такой объект можно создать быстро и оперативно добавлять в него фичи, то лучше не использовать сторонних решений. Это позволяет создавать быстрые тесты без лишних фич, полностью контролировать код и менять его под свои нужды в кратчайшие сроки.

## Абстракция отложенных действий
Вместо явных обращений к `DispatchQueue`
```Swift
DispatchQueue.main.asyncAfter(deadline: .now() + delay) {
...
}
```
лучше обращаться к протоколу, например добавим
```Swift
protocol Scheduling {
    func schedule(after delay: TimeInterval, block: @escaping () -> Void)
```
и его поддержку для `DispatchQueue`
extension DispatchQueue: Scheduling {
    func schedule(after delay: TimeInterval, block: @escaping () -> Void) {
        asyncAfter(deadline: .now() + delay, execute: block)
    }
}

## Реализация с помощью TDD
Допустим нужно поддерживать следующие ситуации:
1. Выполнение действия через заданное время
2. Выполнение действий через разные промежутки времени
3. Выполнение отложенного кода который будет добавлять другие события в будущем

В 3 этапа пишем тесты
```Swift
final class SchedulerTests: XCTestCase {
    func test_schedulesActionAfterDelay() {
        let delay = 4.0
        let sut = Scheduler()
        var callbackCalled = false

        sut.schedule(after: delay) {
            callbackCalled = true
        }

        sut.timeTravel(by: delay - accuracy)
        XCTAssertFalse(callbackCalled)
        sut.timeTravel(by: 2 * accuracy)
        XCTAssertTrue(callbackCalled)
    }

    func test_schedulesActionsByDelayOrder() {
        let sut = Scheduler()
        var events = [Int]()

        sut.schedule(after: 5.0) {
            events.append(2)
        }

        sut.schedule(after: 2.0) {
            events.append(1)
        }

        sut.timeTravel(by: 6.0)

        XCTAssertEqual(events, [1, 2])
    }

    func test_schedulesActionInsideScheduledAction() {
        let delay = 4.0
        let sut = Scheduler()
        var callbackCalled = false

        sut.schedule(after: delay) {
            sut.schedule(after: delay) {
                callbackCalled = true
            }
        }

        sut.timeTravel(by: 2 * delay - accuracy)
        XCTAssertFalse(callbackCalled)
        sut.timeTravel(by: 2 * accuracy)
        XCTAssertTrue(callbackCalled)
    }
}

extension SchedulerTests {
    private var accuracy: TimeInterval {
        return 1e-6
    }
}
```
и получаем реализацию
```Swift
final class Scheduler: Scheduling {
    func schedule(after delay: TimeInterval, block: @escaping () -> Void) {
        let action = Action(block: block, time: now + delay)
        actions.append(action)
        actions.sort { $0.time < $1.time }
    }

    func timeTravel(by delay: TimeInterval) {
        let targetNow = now + delay

        while let action = actions.first, action.time <= targetNow {
            now = action.time
            action.block()
            actions.removeFirst()
        }

        now = targetNow
    }

    private var now: TimeInterval = 0.0
    private var actions = [Action]()
}

extension Scheduler {
    private struct Action {
        let block: () -> Void
        let time: TimeInterval
    }
}
```

## Тестируем код с помощью `Scheduler`
Теперь протестируем отложенный вызов с помощь `Scheduler`
```Swift
final class RetryUseCase {
    init(
        delay: TimeInterval,
        scheduler: Scheduling = DispatchQueue.main,
        onRetry: @escaping () -> Void
    ) {
        self.delay = delay
        self.scheduler = scheduler
        self.onRetry = onRetry
    }

    func scheduleRetry() {
        scheduler.schedule(after: delay) { [weak self] in
            guard let self = self else { return }

            self.onRetry()
        }
    }

    private let delay: TimeInterval
    private let scheduler: Scheduling
    private let onRetry: () -> Void
}
```
```Swift
final class RetryUseCaseTests: XCTestCase {
    func test_retriesAfterDelay() {
        var callbackCalled = false
        let delay: TimeInterval = 5
        let scheduler = Scheduler()
        let sut = RetryUseCase(delay: delay, scheduler: scheduler) {
            callbackCalled = true
        }

        sut.scheduleRetry()

        scheduler.timeTravel(by: delay - accuracy)
        XCTAssertFalse(callbackCalled)
        scheduler.timeTravel(by: 2 * accuracy)
        XCTAssertTrue(callbackCalled)
    }
}
```

## Дальнейшее развитие `Scheduler`
Далее по мере работы над проектом могут появится дополнительные требования, например
* Отмена выполнения запланированного действия, аналог `DispatchWorkItem.cancel()`. Тут можно реализовать поддержку `protocol Cancelable { func cancel() }`
* Убрать дублирование проверок до и после события, например добавив хэлпер метод
```Swift
scheduler
    .timeTravel(by: timeInterval, before: {
        XCTAssertFalse(callbackCalled)
    }, after: {
        XCTAssertTrue(callbackCalled)
    })
```
* Возможность принимать решения на основе текущего времени. `Scheduler.now` делаем публичным аналогом `Date()` или `CACurrentMediaTime()`. Логика в любой момент может узнать текущее время и принять решение на основании него. Этим подходом можно симулировать поведение пользователя во времени, проверять сложные анимации которые используют текущее время для отрисовки состояния.
