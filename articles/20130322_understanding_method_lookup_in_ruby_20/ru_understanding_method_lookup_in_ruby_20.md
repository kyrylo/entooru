О поиске методов в Руби 2.0
===========================

| Проект:        | [Энту́ру](https://www.github.com/kyrylo/entooru/)
|:---------------|:-----------------------------------------------------------------
| Автор:         | Марк-Андре Лафортюн <github_rocks@marc-andre.ca>
| Дата:          | 18 марта 2013 года
| URI:           | [http://tech.pro/tutorial/1149/understanding-method-lookup-in-ruby-20][0]
| Перевел:       | Кирилл Силин <kyrylosilin@gmail.com>
| Дата перевода: | 22 марта 2013 года


Официальное представление `prepend` в Руби 2.0 — это отличная возможность
рассмотреть, как именно Руби поступает с вызовами методов.

Для того, чтобы понять поиск методов (англ. method lookup), крайне необходимо
усвоить иерархию классов в Руби. Я приправил эту статью многими примерами кода.
Вам понадобится Руби 1.9.2 или выше, если вы пожелаете запускать примеры сами.
Там есть один пример, который использует `prepend`: для него нужен Руби 2.0.

Иерархия классов
----------------

Начнём с классического примера наследования классов:

```ruby
class Animal
  def initialize(name)
    @name = name
  end

  def info
    puts "Ya #{self.class}."
    puts "Menya zovut '#{@name}'."
  end
end

class Dog < Animal
  def info
    puts "Ya #{make_noise}."
    super
  end

  def make_noise
    'gavkayu "Gav-gav"'
  end
end

lassie = Dog.new "Lessi"
lassie.info
# => Ya gavkayu "Gav-gav".
#    Ya Dog.
#    Menya zovut 'Lessi'.
```

[Хочу попробовать запустить код!][1]

В данном примере `Dog` наследует класс `Animal`. Мы говорим, что `Animal` — это
[базовый класс (суперкласс)][2] класса `Dog`:

```ruby
Dog.superclass # => Animal
```

Обратите внимание, что `Dog#info` вызывает `super`. Это специальное ключевое
слово исполняет следующее определение метода `info` в иерархии. В нашем случае —
`Animal#info`.

Иерархию любого класса можно посмотреть с помощью метода `ancestors` (рус.
предки):

```ruby
Dog.ancestors # => [Dog, Animal, Object, Kernel, BasicObject]
```

Интересный факт: предки не заканчиваются на `Animal`.

```ruby
Animal.superclass # => Object
```

Объявление `class Animal` эквивалентно `Animal < Object`.

Вот почему «животное» имеет больше методов, чем просто `info` и `make_noise`. В
частности, также имеются такие методы, как `respond_to?`, `methods` и т.д.:

```ruby
lassie.respond_to? :upcase # => false
lassie.methods
# => [:nil?, :===, :=~, :!~, :eql?, :hash, :<=>, :class, :singleton_class, ...]
```

Так что же это за `Kernel` (рус. ядро) и `BasicObject` (рус. базовый объект)?
Про `Kernel` я расскажу как-нибудь потом, а вот про `BasicObject` и
рассказывать-то особо и нечего окромя того, что у него есть всего лишь
незначительное количество методов. Ну и ещё то, что он является концом иерархии
всех классов:

```ruby
# Дальше BasicObject только высота:
Object.superclass # => BasicObject
BasicObject.superclass # => nil
# А ещё у него мало методов:
Object.instance_methods.size # => 54
BasicObject.instance_methods.size # => 8
```

> Классы формируют _корневое дерево_, где `BasicObject` — корень.

Примеси
-------

Руби хоть и поддерживает только простое наследование (то есть класс может иметь
только один родительский класс), но он также обладает поддержкой [примесей][3].
Примесь — это набор методов, который может быть включён в классы. В Руби — это
экземпляры класса `Module`:

```ruby
module Mammal
  def info
    puts "Ya mlekopitayuschee"
    super
  end
end
Mamal.class # => Module
```

Чтобы включить данную функциональность в наш класс `Dog`, мы можем использовать
или `include` (рус. включить), или новый `prepend` (рус. предварить). Эти методы
вставят модуль либо после, либо перед самим классом.

```ruby
class Dog
  prepend Mammal
end
lassie = Dog.new "Lessi"
lassie.info
# => Ya mlekopitayuschee.
#    Ya gavkayu "Gav-gav".
#    Ya Dog.
#    Menya zovut 'Lessi'.
Dog.ancestors # => [Mammal, Dog, Animal, Object, ...]
```

Если бы модуль был включён, а не предварён, то эффект был бы точно таким же.
Только лишь порядок был бы другим. Догадаетесь, какими будут вывод и предки?
[Догадаться можно тут][4].

Вы можете включать и предварять сколько угодно модулей, а модули даже могут быть
включены или предварены в другие модули<a name="sub1-r"></a><a href="#sub1">¹</a>.
Не стесняйтесь вызывать `ancestors`, чтобы лишний раз проверить иерархию модулей
и классов.

Метаклассы
----------

В Руби существует всего один дополнительный уровень иерархии модулей и классов.
Любой объект может иметь специальный класс (только лишь для себя), который имеет
преимущество перед всем: метакласс (англ. singleton class).

Вот простой пример, основанный на предыдущем:

```ruby
scooby = Dog.new "Skubi-Du"

class << scooby
  def make_noise
    'voyu "Skubi-Dubi-Du!"'
  end
end
scooby.info
# => Ya mlekopitayuschee.
#    Ya voyu "Skubi-Dubi-Du!".
#    Ya Dog.
#    Menya zovut 'Skubi-Du'.
```

Обратите внимание на то, как гавканье было заменено на особенный вой Скуби-Ду.
Это не повлияет ни на какие другие экземпляры класса `Dog`.

Запись `class << scooby` — это специальное обозначение, которе открывает
метакласс объекта. Существуют и другие способы определять методы метаклассов:

```ruby
# Делает то же самое, что и предыдущий пример:
def scooby.make_noise
  'voyu "Skubi-Dubi-Du!"'
end
```

Метакласс — это настоящий `Class`, и он может быть доступен посредством вызова
`singleton_class`:

```ruby
# У метаклассов странные имена:
scooby.singleton_class # => #<Class:#<Dog:0x00000100a0a8d8>>
# Метаклассы - настоящие классы:
scooby.singleton_class.is_a?(Class) # => true
# Мы можем получить список его методов экземпляра:
scooby.singleton_class.instance_methods(false) # => [:make_noise]
```

Все объекты Руби могут иметь метаклассы<a name="sub2-r"></a><a href="#sub2">²</a>,
включая непосредственно сами классы. И да, их могут иметь даже метаклассы.

Звучит немного безумно… не потребовалось ли бы для этого бесконечное число
метаклассов? В каком-то смысле да, но Руби создаст метаклассы по мере
необходимости.

Не смотря на то, что в предыдущем примере использовался метакласс экземпляра
класса `Dog`, метаклассы чаще используются для классов. Действительно, [методы
класса][5], по сути, — это методы метакласса. К примеру, `attr_accessor`
является методом экземпляра метакласса класса `Module`. `ActiveRecord::Base` из
Рэйлс обладает многими такими методами: `has_many`, `validates_presence_of`, и
т.д. Это — методы метакласса класса `ActiveRecord::Base`:

```ruby
Module.singleton_class
      .private_instance_methods
      .include?(:attr_accessor) # => true

require 'active_record'
ActiveRecord::Base.singleton_method
                  .instance_methods(false)
                  .size  # => 170
```

Метаклассы, также известные как классы-одиночки, получили такое название из-за
того, что они могут иметь только один экземпляр:

```ruby
scooby2 = scooby.singleton_class.new
  # => TypeError: can't create instance of singleton class
```

С другой стороны, метаклассы обладают полной иерархией классов.

Для объектов у нас есть:

```ruby
scooby.singleton_class.superclass == scooby.class == Dog
# => true, dlya bolshinstva objektov
```

Для классов, Руби автоматически задаст дочерний класс так, что его метакласс
будет равен метаклассу родительского класса.

```ruby
Dog.singleton_class.superclass == Dog.superclass.singleton_class
# => true, для любого экземпляра `Class`
```

Это означает, что `Dog` наследует как методы экземпляра `Animal`, так и методы
его метакласса.

Чтобы запутать всех ещё больше, я закончу абзац заметкой об `extend` (рус.
расширить). Этот метод может рассматриваться, как короткий путь, с помощью
которого можно включить модуль в метакласс получателя<a name="sub3-r"></a><a href="#sub3">³</a>:

```ruby
class << obj
  include MyModule
end

# Делает то же самое, что и код вверху, только короче
obj.extend MyModule
```

> Метаклассы Руби следуют [модели метаклассов (англ.)][6]

Поиск методов и отсутствие методов
----------------------------------

Почти закончили!

Богатая родительская цепь (англ. ancestor chain), поддерживаемая Руби — это
залог поиска методов.

По достижении последнего родительского класса (`BasicObject`), Руби
предоставляет дополнительную возможность для обработки вызова в лице
`method_missing` (рус. отсутствие метода).

```ruby
lassie.woof # => NoMethodError: undefined method
# `woof' for #<Dog:0x00000100a197e8 @name="Lessi">

class Animal
  def method_missing(method, *args)
    if make_noise.include? method.to_s
      puts make_noise
    else
      super
    end
  end
end

lassie.layu # => layu "Gav-gav"
scooby.layu # => NoMethodError
scooby.voyu # => voyu "Skubi-Dubi-Du!"
```

[Хочу попробовать запустить код!][7]

В этом примере мы вызываем `super` только если имя метода не является звуком
который издаёт животное. `super` будет двигаться по родительской цепи, пока не
достигнет `BasicObject#method_missing`, который возбудит исключение `NoMethodError`.

Итог
----

В итоге, вот, что происходит, когда вы вызываете в Руби
`получатель_сообщения.сообщение`:

* сообщение отправляется по родительской цепи `получатель_сообщения.метакласс`
* затем по этой же цепочке отправляется `отсутствие_метода(сообщение)`

Первые методы, которые встречаются в ходе поиска исполняются, а их результат
возвращается. Любой вызов `super` ставит поиск обратно на колею и процесс
нахождения следующего метода продолжается.

Родительская цепь для модуля `mod` выглядит так:

* родительская цепь каждого из предварённых модулей (последный модуль,
  предварённый первым)
* `mod` сам по себе
* родительская цепь каждого из включенных модулей (последний модуль,
  включенный первым)
* если `mod` — это класс, тогда родительская цепь его родительского класса

Можно написать псевдокод `ancestors`, похожий на Руби<a name="sub4-r"></a><a href="#sub4">⁴</a>:

```ruby
class Module
  def ancestors
    prepended_modules.flat_map(&:ancestors) +
    [self] +
    included_modules.flat_map(&:ancestors) +
    is_a?(Class) ? superclass.ancestors : []
  end
end
```

Написание настоящего поиска запутанно слегка сильнее. Код бы выглядел так:

```ruby
class Object
  def send(method, *args)
    singleton_class.ancestors.each do |mod|
      if mod.defines? method
        execute(method, for: self, arguments: args,
          if_super_called: resume_lookup_at(mod))
      end
    end
    send :method_missing, method, *args
  end

  def method_missing(method, *args)
    # На этом месте поиск заканчивается.
    # Если мы оказались тут, то либо метод не был определён нигде в предках и не
    # было определено ни одного метода под названием `method_missing`, не считая
    # этого, либо какие-то другие методы с таким именем были найдены, но вызвали
    # `super`, когда больше не удалось найти другие методы кроме этого.
    raise NoMethodError, "undefined method `#{method}' for #{self}"
  end
end
```

Технические сноски
------------------

<a name="sub1"></a>
<a href="#sub1-r">¹</a> Существуют некоторые ограничения для примесей:

* Руби не очень хорошо поддерживает иерархии, где один и тот же модуль появляется
  более одного раза в иерархии. `ancestors`, как правило, выведет модуль лишь
  единожды, даже если он и включён на [различных (англ.)][8] уровнях в цепочке
  родителей. Я до сих пор надеюсь, что это изменится в будущем (в частности,
  чтобы [избежать сумбурных (англ.)][9] багов).
* Включение подмодуля в модуль `M` не будет никак влиять ни на какой класс,
  который уже включил в себя `M`. Однако он будет влиять на классы, в которые
  включили `M` позже. [Ознакомиться поближе (англ.)][10].
* Также, Руби не разрешает циклы в родительской цепи.

<a name="sub2"></a>
<a href="#sub2-r">²</a> Руби, по сути, запрещает доступ к метаклассам лишь
для нескольких классов: `Fixnum` и `Symbol` (а в Руби 2.0 ещё и для `Bignum` и
`Float`):

```ruby
42.singleton_class # => TypeError: can't define singleton
```

_Немедленные объекты_ (англ. immediates) не могут иметь метаклассы — это
эмпирическая закономерность. Поскольку `(1 << 42).class` вернёт `Fixnum`, если
вы на 64-битной платформе или `Bignum`, если на другой, то удобнее трактовать
`Bignum` так же. В Руби 2.0 то же самое объяснение применяется к `Float`,
поскольку некоторые числа с плавающей запятой [могут быть немедленными (англ.)][11].

К исключениям относятся только: `nil`, `true` и `false`, которые являются
объектами-одиночками своих классов: `nil.singleton_class` вернёт `NilClass`.

<a name="sub3"></a>
<a href="#sub3-r">³</a> Если быть предельно точным, `extend` и
`singleton_class.send :include` по умолчанию имеют одинаковый эффект, однако они
дадут спуск (англ. trigger) разным функциям обратного вызова (англ. callback):
`extended` и `included`, `append_features` и `extend_object`, соответственно.
Если эти функции обратного вызова определены по-другому, тогда воздействие тоже
может быть другим.

<a name="sub4"></a>
<a href="#sub4-r">⁴</a> Учтите, что `prepended_modules` [пока ещё не существует][12].
Также, `singleton_class.ancestors` не включает в себя метакласс, но это
[поменяется в Руби 2.1][13].

[0]: http://tech.pro/tutorial/1149/understanding-method-lookup-in-ruby-20
[1]: http://rubyfiddle.com/riddles/576c1/2
[2]: http://ru.wikipedia.org/wiki/%D0%9D%D0%B0%D1%81%D0%BB%D0%B5%D0%B4%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5_(%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)#.D0.9F.D1.80.D0.BE.D1.81.D1.82.D0.BE.D0.B5_.D0.BD.D0.B0.D1.81.D0.BB.D0.B5.D0.B4.D0.BE.D0.B2.D0.B0.D0.BD.D0.B8.D0.B5
[3]: http://ru.wikipedia.org/wiki/%D0%9F%D1%80%D0%B8%D0%BC%D0%B5%D1%81%D1%8C_(%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)
[4]: http://rubyfiddle.com/riddles/2442f
[5]: http://ru.wikipedia.org/wiki/%D0%9C%D0%B5%D1%82%D0%BE%D0%B4_(%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)
[6]: http://en.wikipedia.org/wiki/Eigenclass_model
[7]: http://rubyfiddle.com/riddles/d31cc/2
[8]: https://bugs.ruby-lang.org/issues/1586
[9]: https://bugs.ruby-lang.org/issues/7844
[10]: http://rubyfiddle.com/riddles/3e690
[11]: http://blog.marc-andre.ca/2013/02/23/ruby-2-by-example/#optimizations
[12]: http://bugs.ruby-lang.org/issues/8026
[13]: http://bugs.ruby-lang.org/issues/8035
