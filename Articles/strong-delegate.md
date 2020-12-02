# Почему переменную delegate лучше делать strong когда придерживаешься принципов SOLID

Мы все привыкли что ссылаться на делегата в iOS нужно через `weak` переменную. Но что делать если, например, делегат `UITableView` это отдельный объект и на него ни что кроме таблицы не ссылается? Прийдется делать `strong` переменную в другом объекте или использовать `objc_setAssociatedObject` чтобы держать делегата в оперативной памяти. Такой подход выглядит не совсем логичным и правильным. В этой статье попытаемся разобраться в причинах проблемы и узнаем что можно с этим сделать.

## Привычный подход к решению проблемы retain цикла
Допустим нам нужно инициализировать два объекта (`MainScreenViewController` и `MainScreenPresenter`). Они ссылаются друг на друга. Часто можно встретить в похожей ситуации такую `weak` переменную в одном из классов.

```Swift
final class MainScreenPresenter {
    weak var view: MainScreenViewController?

    func viewDidAppear() {
        view?.render(status: "Some status")
    }
}
```
```Swift
final class MainScreenViewController: UIViewController {
    required init(presenter: MainScreenPresenter) {
        self.presenter = presenter
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        presenter.viewDidAppear()
    }

    func render(status: String) {
        //some logic
    }

    private let presenter: MainScreenPresenter
}
```
Этот подход решает проблему retain цикла между объектами. А также обходит ограничение Swift требующее сначала инициализировать один объект и только после этого второй.
```Swift
final class MainScreenComposer {
    func composeMainScreen() -> UIViewController {
        let presenter = MainScreenPresenter()
        let viewController = MainScreenViewController(presenter: presenter)
        presenter.view = viewController
        return viewController
    }
}
```
Рассмотрим `weak var view: MainScreenViewController?` в `MainScreenPresenter`. С точки зрения описания `MainScreenPresenter` `view` может стать `nil` в любой момент. В будущем придется постоянно следить за тем чтобы разработчики не меняли значение `view` после инициализации. В коде нужно работать с опциональной переменной, что неудобно. Лучше чтобы у `MainScreenPresenter` было описание интерфейса, которое не дает возможность клиенту сделать неправильное действие (в нашем случае менять значение `view`). В идеале чтобы это было гарантировано на уровне компилятора Swift.

## Улучшаем решение с помощью принципов SOLID
`weak view` это нарушение принципа открытости/закрытости. Например, в тестах для `MainScreenPresenter` может быть не нужна `weak` переменная. Через некоторое время может возникнуть ситуация что не `MainScreenViewController` будет держать в памяти `MainScreenPresenter`, а наоборот. Тогда нужно переписать эти классы, перенести `weak` переменную, поправить тесты. Хотя сама логика не меняется, а значит и не нужно менять код этих классов.

Это нарушает принцип единой ответсвенности. `MainScreenPresenter` занимается не только прямыми обязанностями, но и решает задачу как связать объекты между собой.

Также нарушен принцип инверсии зависимостей: объекты ссылаются на конкретные реализации, а не на абстракции.

## Делаем следующие шаги:
1. Создаем протоколы `MainScreenViewable`, `MainScreenPresentation`
2. Правим классы чтобы они ссылались друг на друга через протоколы
3. `weak` переменную выносим в отдельный класс который решает проблему `retain` цикла

```Swift
protocol MainScreenPresentation {
    func viewDidAppear()
}
```
```Swift
protocol MainScreenViewable {
    func render(status: String)
}
```
```Swift
final class MainScreenViewController: UIViewController, MainScreenViewable {
    required init(presenter: MainScreenPresentation) {
        self.presenter = presenter
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        presenter.viewDidAppear()
    }

    func render(status: String) {
        //some logic
    }

    private let presenter: MainScreenPresentation
}
```
```Swift
final class MainScreenPresenter: MainScreenPresentation {
    init(view: MainScreenViewable) {
        self.view = view
    }

    func viewDidAppear() {
        view.render(status: "Some status")
    }

    private let view: MainScreenViewable
}
```
```Swift
final class MainScreenComposer {
    func composeMainScreen() -> UIViewController {
        let viewWeakifyAdapter = ViewWeakifyAdapter()
        let presenter = MainScreenPresenter(view: viewWeakifyAdapter)
        let viewController = MainScreenViewController(presenter: presenter)
        viewWeakifyAdapter.adaptee = viewController
        return viewController
    }
}
```
```Swift
final class ViewWeakifyAdapter: MainScreenViewable {
    weak var adaptee: (MainScreenViewable & AnyObject)?

    func render(status: String) {
        adaptee?.render(status: status)
    }
}
```

## Выводы
SOLID принципы дают возможность по новому взглянуть на подходы к которым мы все привыкли. Декомпозировать код на небольшие объекты которые легко переиспользовать. Таким образом даже привычный нам `weak delegate` превращается в `strong delegate`.
