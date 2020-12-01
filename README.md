# Виктор Куринный об iOS разработке

## [Почему переменную delegate лучше делать strong когда придерживаешься принципов SOLID](strong-delegate.md)

Мы все привыкли что ссылаться на делегата в iOS нужно через `weak` переменную. Но что делать если, например, делегат `UITableView` это отдельный объект и на него ни что кроме таблицы не ссылается? Прийдется делать `strong` переменную в другом объекте или использовать `objc_setAssociatedObject` чтобы держать делегата в оперативной памяти. Такой подход выглядит не совсем логичным и правильным. В этой статье попытаемся разобраться в причинах проблемы и узнаем что можно с этим сделать.
