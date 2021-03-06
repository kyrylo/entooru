Стандартные классы Руби небезопасны для использования во многопоточных программах
=================================================================================

| Проект:        | [Энту́ру](https://www.github.com/kyrylo/entooru/)
|:---------------|:-----------------------------------------------------------------
| Автор:         | Джесси Стоример <jesse@jstorimer.com>
| Дата:          | 25 февраля 2013 года
| URI:           | [http://www.jstorimer.com/newsletter/ruby-core-classes-arent-thread-safe.html][0]
| Перевел:       | Кирилл Силин <kyrylosilin@gmail.com>
| Дата перевода: | 5 марта 2013 года


Несколько дней назад я поделился цитатой в Твиттере:

> Массивы Руби небезопасны для использования во многопоточных программах, [...]
> реализация потоков в РРМ (англ. MRI; рус. реализация Руби Маца) нечаянно
> делает их безопасными для использования во многопоточных программах.

Эта фраза получила кучу ретвитов, завелась дискуссия. Однако, посмею сказать,
что нюансы данного утверждения были неочевидны для многих разработчиков. Черт
побери, несколько месяцев назад я бы даже не был в курсе, о чём всё это. Да и
звучит как-то нездорово. Значит ли это, что я не должен использовать массивы?
Или, быть может, я должен использовать только РРМ? Или, быть может, я не должен
использовать потоки? Что, чёрт возьми, эта проблема безопасности потоков во
многопоточных программах на самом деле значит для моего кода?

Это серьезный вопрос, и я думаю, ответ лежит в обучении. Если мы не понимаем,
что значит безопасность потоков во многопоточных программах, или не понимаем
особенности реализации потоков в РРМ, тогда тот твит и выеденного яйца не стоит.
Так вот, я хочу убедиться, что все мы понимаем, о чём идёт речь.

Давайте начнем с примера.

Вот очень простой класс, который учитывает инвентарь:

```ruby
class Inventory
  def initialize(stock_levels)
    @stock = stock_levels
  end

  def decrease(item)
    @stock[item] -= 1
  end

  def [](item)
    @stock[item]
  end
end
```

Пример довольно банален. Вы даете ему ваш уровень запасов, а затем сокращаете
инвентарь по мере продаж.

```ruby
inventory = Inventory.new(:tshirt => 200,
                          :pants => 340,
                          :hats => 4000)

inventory.decrease(:hats)
inventory.decrease(:pants)
```

Теперь я хочу у вас спросить: безопасен ли код для использования во
многопоточных программах? Что вас настораживает?

Давайте-ка опробуем этот код, чтобы не гадать. У меня есть 4000 шляп, и если я
сокращу инвентарь 4000 раз, то шляп у меня должно остаться 0.

```ruby
4000.times do
  inventory.decrease(:hats)
end

puts inventory[:hats]
```

Результат проверки этого кода на РРМ 1.9.3 даст предсказуемый результат — 0. Но
процесс списания товара кажется слишком медленным. Ваш коллега решает, что
каждое списание товара должно быть обёрнуто в поток так, чтобы работа шла
параллельно.

Теперь давайте проверим код ещё разок. На этот раз с потоками.

```ruby
threads = Array.new
40.times do
  threads << Thread.new do
    100.times do
      @inventory.decrease(:hats)
    end
  end
end

threads.each(&:join)
puts @inventory[:hats]
```

В этот раз мы вызвали 40 потоков, каждый из которых сокращает инвентарь 100 раз.
В сумме получается 4000, так что результат опять должен равняться 0. В примере
выше есть ещё немного кода, так как нам нужно отслеживать каждый поток и
вызывать для него `join`, чтобы убедиться, что он закончил своё выполнение. И
вновь я вопрошаю: безопасен ли код для использования во многопоточных
программах?

Вот результаты выполнения кода на разных реализациях Руби: РРМ, ДжейРуби и
Рубиниусе:

```
MRI 1.9.3    :  0
JRuby 1.7.2  :  486
RBX 2.0.0rc1 :  2
```

Кажется, если я прогоняю код на РРМ, то он безопасен для использования во
многопоточных программах. Однако в других реализациях Руби обнажается его
небезопасность. Вот об этом-то и шла речь в том твите. Наш опыт использования
РРМ наводит на мысли, что стандартные классы безопасны, но это всего лишь
«забавное совпадение» РРМ, которое на самом деле оборачивается совсем незабавной
катастрофой в других средах выполнения кода.

Заметка: если вы запустите этот код сами, то почти наверняка вы получите другие
ненулевые результаты. Это и есть природа проблемы. Результаты непредсказуемы и
недетерминистичны.

Так что же нам делать? Ну, для каждого языка существуют общие правила, следуя
которым код будет безопасен во многопоточных программах.

Если мы хотим, чтобы наш код был потокобезопасен, параллельные изменения должны
быть синхронизированы. То бишь, _все_ параллельные изменения должны быть
синхронизированы. Это есть основополагающее правило безопасного для
использования во многопоточных программах кода. Звучит страшновато, но я вам
сейчас конкретно покажу, как это сделать.

Короче говоря, самый распространенный способ, когда потоковая безопасность
рушится — это когда несколько потоков пользуются одними и теми же переменными и
пытаются менять их одновременно. Самым очевидным примером для Руби будет
глобальная переменная или переменная класса, которую пытаются изменить разные
потоки. Но это также легко может произойти и с общей переменной экземпляра или
локальной переменной.

Я _настоятельно_ рекомендую вам прочесть [4 правила безопасной многопоточности в
Руби (англ.)][1] (или для любой другой платформы). Следование тем правилам облегчит
вашу программистскую участь.

Так как же нам сделать этот код `Inventory` безопасным для многопоточных
программ? И почему же этот код безопасен в РРМ? Для начала сделаем его
потокобезопасным. Самым лёгким тут будет использование класса `Mutex`, который
позволит синхронизировать доступ к `@inventory`.

```ruby
threads = Array.new
lock = Mutex.new

400.times do
  threads << Thread.new do
    lock.synchronize do
      10.times do
        @inventory.decrease(:hats)
      end
    end
  end
end

threads.each(&:join)
puts @inventory[:hats]
```

Опять прогоним этот код через все три реализации Руби. Я вижу вполне ожидаемый
результат:

```
MRI 1.9.3    :  0
JRuby 1.7.2  :  0
RBX 2.0.0rc1 :  0
```

Мы только что пробежались по новому коду и прошли мимо разговора о некоторых
нюансах, к которым я аппелировал ранее. Давайте-ка начнём с кода.

Концепт «мьютекса» (англ. mutex) не относится непосредственно к Руби. Мьютексы,
как и потоки, предоставляются ядром вашей ОС. «Mutex» — это сокращение
словосочетания «mutual exclusion», что в переводе на русский означает «взаимное
исключение». Мьютекс позволяет вам синхронизировать доступ к какой-либо части
кода. Другими словами, один и только один поток может выполнять код, который
синхронизирован мьютексом в любой момент времени. Вот почему он также зовется
замкòм (англ. lock). Когда какой-то поток сокращает инвентарь, он вешает замок
на этот кусочек кода. Другие потоки, прежде чем начать действовать, должны
подождать в очереди, пока этот поток не отворит мьютекс.

Мьютексы защищают доступ к общему состоянию. Вы сами видели, что произошло с
запасами в инвентаре, когда мы не использовали мьютекс: данные были повреждены.
Чтобы избегать таких ситуаций, придерживайтесь 4-х правил, которые я упомянул
выше. Будет очень досадно видеть поврежденные данные, если вы забудете
использовать замок.

Так почему же у РРМ получается выдавать правильный результат без мьютекса?

Подумайте минутку-другую, я не спешу. Ответ — это широко известный среди
рубистов скверный набор прописных букв… дзынь-дзынь-дзынь, вы догадались!
ГЗИ (англ. GIL; рус. глобальный замòк интерпретатора), также известный как
Глобальный замòк.

Глобальный замок — это особенность РРМ, которая, по сути, оборачивает весь ваш
код в большой мьютекс. Да-да, даже если у вас многоядерный процессор, и вы
прогоняете код, использующий несколько потоков на РРМ, то он не будет
выполняться параллельно. Существуют несколько конкретных оговорок, которыми
пользуется РРМ для параллельного ввода-вывода. Если у вас есть один поток,
который ожидает ввод-вывод (например, ожидает ответ от базы данных или
веб-запрос), РРМ позволит другому потоку работать параллельно. Однако в
остальных случаях ваш код обернут в большой замок.

ДжейРуби и Рубиниус (2.0 и выше) не имеют такого глобального замка. Этим и
объясняются неверные результаты в самом первом примере. Вместо глобального
замка, их ключевые части защищены мелкими замками так, что их внутренности
безопасны для использования во многопоточных программах. И это позволяет быть
вашему коду в действительности параллельным. Как мы видели ранее, настоящий
параллельный код — это благо. Но если быть невнимательным, то это также и палка
о двух концах. Но разве это не относится почти ко всему, что мы делаем?

Ещё одно: если вы замерите скорость выполнения двух наших решений и сравните их,
вы обнаружите, что многопоточная версия с мьютексом окажется медленней, чем
однопоточная. В большей степени это вина надуманного примера, который мы
использовали (но обозначить это всё равно необходимо). По определению, код во
мьютексе не может быть запущен параллельно, поскольку только один поток может
выполнять его в любой момент времени. В нашем примере весь код был внутри
мьютекса, так что дополнительные потоки только лишь создавали накладные расходы,
а не ускоряли всё. Помните, что бездумное выделение парочки потоков для решения
задачи ничего автоматически не ускорит.

Если представить наш притянутый за уши пример в реальном мире, то каждый поток
имел бы бòльший вес при процессе списания товара. Ему бы пришлось собирать
платежи, осуществлять погрузку, оповещать кого-то по СМС, ну и сокращать
инвентарь. В таком случае, я более уверен, что подход с использованием
многопоточности действительно бы был быстрее по скорости выполнения.

Можно ещё много говорить на эту тему, поэтому моя следующая книга будет
посвящена этим вопросам. Не теряйтесь, вскоре у меня будет больше информации для
вас. [Обновление: [работает сайт][2], на котором больше информации]

Если вы узнали о чём-то новом, или если у вас остались неотвеченные вопросы,
дайте мне знать. Я всегда их читаю.

До скорого! Джесси [http://workingwithcode.com][3]

[0]: http://www.jstorimer.com/newsletter/ruby-core-classes-arent-thread-safe.html
[1]: https://github.com/jruby/jruby/wiki/Concurrency-in-jruby#wiki-concurrency_basics
[2]: http://workingwithrubythreads.com/
[3]: http://workingwithcode.com
