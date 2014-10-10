Секреты «delegate.rb» из Руби
=============================

| Проект:        | [Энту́ру](https://www.github.com/kyrylo/entooru/)
|:---------------|:-----------------------------------------------------------------
| Автор:         | Джим Гей <jim@saturnflyer.com>
| Дата:          | 21 марта 2013 года
| URI:           | [http://www.saturnflyer.com/blog/jim/2013/03/21/ruby-delegate-rb-secrets/][0]
| Перевел:       | Кирилл Силин <kyrylosilin@gmail.com>
| Дата перевода: | 23 марта 2013 года


Вы видели `SimpleDelegator` (рус. простой делегатор) в действии и даже немного
им пользовались. Библиотека `delegate` — это нечто большее, чем просто модная
обёртка `method_missing`.

Простые обёртки
---------------

Первое и самое самое главное, что я хочу сказать: `SimpleDelegator` — это просто
модная обёртка `method_missing`. Я знаю, что я говорил, что это нечто большее,
но просто будьте ко мне более снисходительны.

Вот пример кода:

```ruby
vasily = Person.new # какой-то объект

class Displayer < SimpleDelegator
  def name_with_location
    "#{__getobj__.name} из #{__getobj__.city}"
  end
end

displayer = Displayer.new(vasily)

puts displayer.name_with_location #=> "Василий из города N"
```

Класс `Displayer` (рус. отображатель) инициализируется с объектом и
автоматически устанавливает его как `@delegate_sd_obj`.

Возможно, вы захотите присвоить псевдонимы для методов, чтобы они не выглядели
так уродливо: `alias_method :__getobj__, :object`

### Отсутствие метода

Вот развёрнутый пример того, как идёт обращение с `method_missing` (рус.
отсутствие метода).

```ruby
target = self.__getobj__ # Получаем целевой объект
if target.respond_to?(the_missing_method)
  target.__send__(the_missing_method, *arguments, &block)
else
  super
end
```

Настоящий код будет немного покомпактнее, но мой пример понятней.
По сути, `SimpleDelegator` так прост, что вы сами сможете его реализовать:

```ruby
class MyWrapper
  def initialize(target)
    @target = target
  end
  attr_reader :target

  def method_missing(method_name, *args, &block)
    target.respond_to?(method_name) ?
      target.__send__(method_name, *args, &block) : super
  end
end
```

Но это ещё не всё. Однако если всё, что вам нужно — это удобное использование
`method_missing`, то я уже показал, как это работает.

Методы простого делегатора
==========================

`SimpleDelegator` добавляет несколько удобных вещей для просмотра доступных
методов. Допустим, наш `vasily` обёрнут в `displayer`. Что мы можем с ним
сделать? Если мы вызовем `displayer.methods`, то получим уникальный набор
методов как Васи, так и отображателя.

Вот, что происходит:

```ruby
def methods(all=true)
  __getobj__.methods(all) | super
end
```

Определяется метод `methods` и используется дизъюнкция (метод `|` класса
`Array`), чтобы получить уникальный набор. Методы Васи группируются с методами
обёртки:

```ruby
['a','b'] | ['c','b'] #=> ['a','b','c']
```

То же можно наблюдать для `public_methods` и `protected_methods`, но _не_ для
`private_methods`. Приватные методы приватны, так что у вас не должно быть к
ним доступа снаружи.

Зачем оно это делает? Ведь нам же нужно знать, что осно

[0]: http://www.saturnflyer.com/blog/jim/2013/03/21/ruby-delegate-rb-secrets/
