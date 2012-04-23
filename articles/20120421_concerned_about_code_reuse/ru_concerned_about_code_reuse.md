Заинтересованы в повторном использовании кода?
==============================================

| Проект:        | [Энту́ру](https://www.github.com/kyrylo/entooru/)
|:---------------|:-----------------------------------------------------------------
| Автор:         | Ричард Шниман <richard.schneeman@gmail.com>
| Дата:          | 19 апреля 2012 года
| URI:           | [http://schneems.com/post/21380060358/concerned-about-code-reuse][0]
| Перевел:       | Кирилл Силин <kyrylosilin@gmail.com>
| Дата перевода: | 24 апреля 2012 года


Прямо со старта Руби дает нам мощные способы повторно использовать методы
экземпляра и методы класса без использования наследования. Модули в Руби могут
быть довольно легко использованы для подмешивания методов классам. К примеру,
мы можем добавить новые методы экземпляра, используя `include`:

``` ruby
module DogFort
  def call_dog
    puts "Это собака!"
  end
end

class Dog
  include DogFort
end
```

Теперь мы можем вызывать любые методы, определенные в модуле `DogFort`, как
будто бы они были объявлены в нашем классе `Dog`:

``` ruby
dog_instance = Dog.new
dog_instance.call_dog
#=> "Это собака!"
```

С помощью модулей можно довольно легко повторно использовать методы. Если же вы
хотите добавить методы непосредственно классу, то вы можете использовать
`extend`:

``` ruby
module DogFort
  def board_the_doors
    puts "Кошки - прочь!"
  end
end

class Dog
  extend DogFort
end
```

Классно! Однако что делать, если вы захотели добавить классу и метод экземпляра,
и метод класса? Мы могли бы создать два модуля: один бы подключался через
`include`, а другой — через `extend`. Это не составило бы особого труда, но было
бы неплохо использовать только лишь `include` (особено, если два модуля
логически связаны). Итак, возможно ли добавить метод экземпляра и метод класса
с помощью одного выражения `include`? Естественно…

В дело вступают Концерны
------------------------

Концерн (англ. Concern) — это модуль, который добавляет методы экземпляра
(как `Dog.new.call_dog`) и методы класса (как `Dog.board_the_doors`) классу. Если
вы пошарите по коду Рэйлс, то увидите, что он используется везде. Он
используется так часто, что в _ActiveSupport_ (Эктив Саппорт — прим. пер.) даже
добавили модуль-хэлпер для создания концернов. Чтобы воспользоваться им,
подключите _ActiveSupport_ а затем напишите `extend ActiveSupport::Concern`:

``` ruby
require 'active_support/all'

module DogFort
  extend ActiveSupport::Concern
  # ...
end
```

Теперь любые методы, которые вы напишите в этом модуле будут методами экземпляра
(методы нового экземпляра класса `Dog.new`) и любые методы, которые вы напишете
в модуле `ClassMethods` будут добавлены непосредственно в класс (такой, как
`Dog`):

``` ruby
require 'active_support/all'

module YoDawgFort
  extend ActiveSupport::Concern

  def call_dawg
    puts "Йоу, доуг, это доуг!"
  end

  # Всё, в ClassMethods становится методом класса
  module ClassMethods
    def board_the_doors
      puts "Йоу доуг, кошки - прочь!"
    end
  end
end
```

Довольно клёво, ага?

Included
--------

Это не всё. _ActiveSupport_ также дает нам особый метод, который называется
_included_ (инклудэд — прим. пер.), который мы можем использовать для вызова
методов во время подключения модулей с помощью `include`. Если вы добавите
`included` к вашему `ActiveSupport::Concern`, то всякий код, который находится
в блоке included будет вызван во время подключения вашего модуля:

``` ruby
module DogCatcher
  extend ActiveSupport::Concern

  included do
    if self.is_a? Dog
      puts "Попался!!"
    else
      puts "Можешь идти"
    end
  end
end
```

Так вот, когда мы подключаем `DogCatcher` в класс, то его блок _included_ будет
вызван тотчас:

``` ruby
class Dog
  include DogCatcher
end
# => "Попался!!"

class Cat
  include DogCatcher
end
# => "Можешь идти"
```

Пусть это и надуманный пример, но допустим, вы желаете реализовать концерн для
контроллеров Рэйлс и добавить код `before_filter` к нашему коду. Мы легко можем 
сделать это, добавив блок _included_.

Это магия?
----------

Нет, под капотом мы всего лишь используем старый добрый Руби. Если вы хотите
узнать больше о всех потешных вещах, которые вы можете делать с модулями, я
рекомендую обратиться к одной из моих любимых книг «[Метапрограммирование Руби][1]»,
а у Дэйва Томаса также есть фантастический [цикл скринкастов][2].

Попался
-------

Когда вы будете писать модули, я гарантирую, что вы совершите ошибку и случайно
попытаетесь создать метод класса, используя `self` или `class << self`, однако
это не сработает, потому что в данной ситуации вы пишете метод модуля:

``` ruby
module DogFort
  def self.call_dog
    puts "Это собака!"
  end
end
```

В примере выше контекст `self`, в действительности, — это модульный объект
`DogFort`. Так что когда мы подключаем его в другой класс, мы не видим этот
метод:

``` ruby
class Wolf
  include DogFort
end

Wolf.call_dog
# NameError: undefined local variable or method `call_dog'

wolf_instance = Wolf.new
wolf_instance.call_dog
# NameError: undefined local variable or method `call_dog'
```

Если вы желаете использовать этот метод в сложившейся ситуации, то вам нужно
вызывать модуль напрямую:

``` ruby
DogFort.call_dog
# => "this is dog!"
puts DogFort.class
# => Module
```

Конец
-----

На сегодня это всё. В своей следующей записи я собираюсь показать вам, как
подчистить ваш код, доставшийся по наследству (legacy code — прим. пер.) с
помощью концернов. Дайте мне знать, если у вас есть какие-либо вопросы
[@schneems][3]!

Возможно, вы также проявите интерес к [Заинтересовывая себя в
ActiveSupport::Concern][4], [Концерны в ActiveRecord][5] и [Улучшенные идиомы
Руби][6].

[0]: http://schneems.com/post/21380060358/concerned-about-code-reuse
[1]: http://pragprog.com/book/ppmetr/metaprogramming-Ruby
[2]: http://pragprog.com/screencasts/v-dtRubyom/the-Ruby-object-model-and-metaprogramming
[3]: http://twitter.com/schneems
[4]: http://www.fakingfantastic.com/2010/09/20/concerning-yourself-with-active-support-concern/
[5]: http://weblog.jamisbuck.org/2007/1/17/concerns-in-activerecord
[6]: http://yehudakatz.com/2009/11/12/better-Ruby-idioms/
