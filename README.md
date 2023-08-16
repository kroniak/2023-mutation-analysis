# 2023 IBS Мутационный анализ

## Литература

1. [Assessing Test Quality by David Schuler](https://d-nb.info/1051432480/34)
2. [Mutation Testing by Filip van Laenen](https://leanpub.com/mutationtesting)

## Презентация

[Presentation IBS 2023 Mutation Testing](https://disk.yandex.ru/i/NaN8fEADLs1YAQ)

## Запуск примеров

```bash
./scripts/test.coverage.sh
cd test/

dotnet tool install -g dotnet-stryker
dotnet stryker -cp stryker-config.json
```

## Мутационный анализ

Разберем самый простой пример

```c#
int Method(bool a, bool b) {
  int c;
  if (a && b) { // mutate to ||
    c = 1;
  } else {
    c = 0;
  }
  return c;
}

[Fact]
[InlineData(false, false)]
void Method_ReturnFalse(bool a, bool b) {
  // Act
  var result = Method(a, b);

  // Assert
  Assert.False(result);
}
```

Мутант выживет или нет?

Для тогда что бы тест убил мутанта должны быть соблюдены следующие условия

1. Тест должен достигнуть кода, то есть покрытие этих строк должно быть 100% - Покрытие
2. Входные данные для теста должны быть полными, что бы мутант был убит - все проверяемые граничные условия - Тестовые данные
3. Значение выходных параметров должно влиять на тесты и проверяться в тестах - Достаточное ассертированность

Вот 3 условия при котором будет убит мутант.

```c#
int Method(bool a, bool b) {
  int c;
  if (a && b) { // mutate to ||
    c = 1;
  } else {
    c = 0;
  }
  return c;
}

[Fact]
[InlineData(false, false)]
[InlineData(true, false)]
[InlineData(false, true)]
void Method_ReturnFalse(bool a, bool b) {
  // Act
  var result = Method(a, b);

  // Assert
  Assert.False(result);
}
```

## Общая модель строится мутационного тестирования строится так

Сначала анализируется код и создается AST и генерируются все возможные мутации, отбрасываются гарантированно некорректные мутации, которые не дадут, например скомпилировать код, создается словарь мутаций, далее наиболее сложная часть мутационного тестирования - это выявление эквивалентных мутация (об них мы поговорим чуть позже) - это мутации, которые невозможно убить никаким тестом. После этого ваш код собирается со всеми мутациями и результат сборки подсовывается в набор тестов.
После этого идет сбор статистики ее выдача в виде, который может проанализировать разработчик.

## Какие мутации могут быть в программном обеспечении

Какие вообще существуют мутации в ПО?

Все мутации - это не хаотичные изменения кода, а точно выверенные те, которые на самом деле могут быть из-за **человеческого** фактора применены в коде или при **рефакторинге** приложения.

Большинство этих мутаций до тестов никогда не дойдут - сработает первая защита - проверка типов и компиляция.
Другие мутации будут применены.

## Мутации

1. ООП Мутации в зависимости от языка и его реализации
    - Мутации модификаторов доступа
    - Мутации наследования - удаление указателей на классы, удаление override методов, удаление вызова base или super, удаление вызова родительского конструктора
    - Полиморфные мутации, например, изменения конкретного класса реализации интерфейса в коде
    - Мутации перегрузки методов и их аргументов
    - Общие мутации - удаление `this`, передача параметров по ссылке или по значению
2. Общие мутации
    - Арифметические операции
    - Мутации сравнения
    - Мутации логических операторов
    - Мутации bool типа
    - Мутации присвоения и арифметические
    - Унарные операции
    - Строковые мутации
    - Мутации массивов и списков
3. Специфические мутации для языка и библиотек - LINQ мутации

Теперь представьте, что этот массив операций мутаций будет применен к вашему коду, кол-во очень большое, вы еще точно уверены, что ваши тесты упадут?

Кол-во общих мутаций зависит от кол-во кода, для примера приведу пример из двух библиотек до того как применить мутационное тестирование

Flurl, 9000 строк, покрытие 85%, 300 тестов, 363 мутантов, 43 сек выполнение тестов, 30 минут выполнение прогона мутаций

Flurl Url Builder, 108 мутаций, изначально выжило 11 штук, рефакторингом доведено до 0, время прогона 7 минут, критичных мутаций 2 - ошибка в распарсинге коллекции объектов до пар ключ значение, что бы построить правильную строку поиска,
вторая критичная ошибка - при установке фрагмента в url не проставлялось значение при null и установка могла привести к неверному значению, если пользователь пользуется методом и сам не проверяет что передает внутрь метода библиотеки. Это просто забыли проверить.

Flurl Http, 255 мутаций, 40 выжило, после рефакторинга осталось 2.

Ecwid Client, 5000 строк, покрытие 90%, 200 тестов, 16 секунд выполняются тесты, 300 мутантов, 15 минут прогон всех мутаций, изначально выжило 70, после рефакторинга 10 выжило. Найдены 4 критичные ошибки.

## Распространенные примеры выживших мутаций и их влияние на тесты и на код

Теперь рассмотрим наиболее часто встречаемые примерны выживших мутаций.

Вообще бороться с мутантами можно 2 способом - честным и нечестным, честный - это когда вы изменяете код или дописываете тесты, что бы мутант был убит, нечестный - вы переносите зону ответственности за ваш код, куда мутационное тестирование не может достать.

Статистика такая

- Примерно 10% мутаций будет в мертвом коде, который необходимо отключить или все таки написать на него тесты, если вы считаете что код может понадобиться, например если это публичная библиотека.
- Примерно 50% мутаций выживших укажут вам на недостаточность тестов.
- Остальные мутации относятся к эквивалентным.

## Строковый мутант

Самый распространенный выживший мутант - строковый. Кол-во строк, которые курсируют из метода в метод, из exception в exception - очень велико.
А вы всегда проверяйте все ошибки, которые возвращаются на их корректность, а еще и в разных локалях?? А вы проверяете все значения строковые получаемого объекта?
А это самая прямая дорога получить на главной странице сайта или в мобильном приложении ошибку внутреннюю со всем stacktrace, что ведет к репутационным потерям компании.

 ```c#
private bool ExceptionMethod(int a) {
  if (a < 0)
    throw new ArgumentException("USER_DB is not path pattern uiier_ksks", "user-name");

  ...

  return true;
}

public string LibraryMethod(int a) {
  try {
    if (ExceptionMethod(a)) return "Successfully";
  }
  catch (ArgumentException e) {
    throw new Exception("Something happened", e);
  }
  return null;
}

[Fact]
public void LibraryMethod_NegativeData_ThrowException() {
  // Act
  var ex = Assert.Throws<Exception>(() => SampleClass.LibraryMethod(-1));
  // Assert
  Assert.Equal("Something happened", ex.Message);
}
```

Что мы тут забыли? если мы это не проверяем, то мы это не конторолируем.

```c#
[Fact]
public void LibraryMethod_NegativeData_ThrowException() {
  // Act
  var ex = Assert.Throws<Exception>(() => SampleClass.LibraryMethod(-1));
  // Assert

  Assert.Equal("Something happened", ex.Message);
  Assert.Equal("USER_DB is not path pattern uiier_ksks\nParameter name: user-name", ex.InnerException.Message);
}
```

- В Ecwid был найден баг, при котором сервис не мог сконфигурироваться, когда один из токенов был невалидный, а не когда все были не валидные. Тесты были, exception падали, но не проверялись тексты ошибок - тесты были написаны неверно и выявился интересный баг.

```c#
[Fact]
public void Credentials_Fail() {
  Assert.Throws<Exception>(() => new Ecwid(0, Token, Token));

  Assert.Throws<Exception>(() => new Ecwid(ShopId));

  Assert.Throws<Exception>(() => new Ecwid(ShopId, " ", Token));

  Assert.Throws<Exception>(() => new Ecwid(ShopId, Token, " "));
}
```

## Мутация сравнения

Второй по распространенности выживший мутант - это проверка условий и каких то границ внутри методов и мутация операторов сравнения внутри методов - чаще всего эти мутанты говорят нам о недостаточности входных данных для теста

```c#
public string Encode(string s, ...) {
  if (string.IsNullOrEmpty(s)) return s;

  if (s.Length > MAX_URL_LENGTH) {
    ...
  }
  ...
}
```

Здесь забыли подать на вход для проверки строку равной максимальной длине.
или

```c#
public Builder Limit(this Builder query, int limit) {
  if (limit <= 0)
    throw new ArgumentException("Limit must be greater than 0", nameof(limit));

  if (limit > 100) limit = 100;

  query.AddOrUpdate("limit", limit);
  return query;
}
```

Или в этом коде, мы проверили граничные условия, но не взяли никакой средний вариант. Потому что обычно тестируем негативные варианты.

```c#
public Builder Limit(this Builder query, int limit) {
  if (limit <= 0)
    throw new ArgumentException("Limit must be greater than 0", nameof(limit));

  if (limit > 99) limit = 100;

  query.AddOrUpdate("limit", limit);
  return query;
}
```

Вот еще распространенный пример мутации операции сравнения

```c#
public void Some<T>(object arg, IEnumerable<T> list) {
  ...
  var out = new List<T>();
  ...

  if (out.Count > 0) {
    ...
  }
  ...
}
```

И тут мы приходим ко 2 способу ухода от мутации - вынесение ее за наш код

```c#
public void Some<T>(object arg, IEnumerable<T> list) {
  ...
  var out = new List<T>();
  ...

  if (out.Any()) { // FirstOrDefault() != null
    ...
  }
  ...
}
```

## LINQ мутации

LINQ мутации распространенный считается замена First на Last, Any на All, FirstOrDefault на Default.

```c#
public T GetValue(int field) {
  var values = await GetValues(new {field});

  return values.FirstOrDefault();
}

[Fact]
public void GetValue_ReturnCorrectData() {
  //... mock data
  var result = await client.GetValue("some data");

  Assert.Equal("some data", result.ValueNumber);
  //... some other assertion
}
```

Тест корректен, данные входные тоже, можно было бы пометить этот мутант просмотренным, но появление мутанта заставило покопаться в результатах вывода API хранилища и оказалось, что при запросе не который набор данных хранилище может выдавать реальный результат из нескольких записей и в зависимости от сортировки, которая задается региональными настройками, первая запись не будет совсем той, которую мы ищем. API был изменен со стороны хранилища. Ошибка не наша, на нам не легче будет, когда это всплывет в продакшене. В итоге этот мутант закрывается расширением набора входных данных и установлением сортировки по умолчанию.

Кстати, про сортировки, настоятельно прошу тестировать ваше приложение, даже unit тестами именно в том окружении, где По будет работать. Хотите в docker - тесты прогоняйте там же. На моей практике было несколько случаев, что тесты, работающие на win, не работали в docker и наоборот. При работе с датами, и округлением денег в decimal необходимо жестко устанавливать локаль.

## Примеры эквивалентных мутаций и способы ухода от них

Самые сложные мутации это эквивалентные, те, которые не могут быть убитыми никакими тестами.

Самый простой пример это условие выхода из цикла:

```c#
int index = 0;
while (...) {
  // …;
  index++;
  if (index == 10) { // mutate to >=
    break;
  }
}
```

Но такие простые мутанты вычищаются как эквивалентные еще на этапе подготовки мутантов.

Есть такие мутации

```c#
public void Remove(SomeType<T> container, T data) {
  if (data == null) return;

  if (container.All(c => c.SomeField != data.SomeField)) return;

  container.Values.Remove(data); // очень много связанной логики и т.п.
}

[Fact]
public void Remove_NotExist_NotRemoved() {
  // Arrange
  var expected = container.Values.Count;
  var data = new Value();

  // Act
  TRepository.Remove(container, data);

  // Assert
  Assert.Equal(expected, container.Values.Count);
}
```

В чем проблема?? Дело в том, что этот код бессмысленнен. Даже если условие не выполнится, то удаление произойдет. Решение 1 - удалить проверку вообще. Решение   2 - переписать немного код и бац - мы находим проблему в другом тесте.

```c#
[Fact]
public void Remove_Exist_Removed() {
  // Arrange
  var expected = container.Values.Count;
  var data = container.Values.Get(1);

  // Act
  TRepository.Remove(container, data);

  // Assert
  Assert.Equal(expected - 1, container.Values.Count);
}
```

Это тест наш на удаление элемента уже существующего. Он работает, и мутация прошла. Рефакторим

```c#
public void Remove(SomeType<T> container, T data) {
  if (data == null) return;

  var deleteT = container.FirstOrDefault(c => c.SomeField == data.SomeField);
  if (deleteT == null) return;

  container.Values.Remove(deleteT); // очень много связанной логики и т.п.
}

[Fact]
public void Remove_Exist_Removed() {
  // Arrange
  var expected = container.Values.Count;
  var data = container.Values.Get(1);

  // Act
  TRepository.Remove(container, data);

  // Assert
  Assert.Equal(expected - 1, container.Values.Count);
}

[Fact]
public void Remove_NotExist_NotRemoved() {
  // Arrange
  var expected = container.Values.Count;
  var data = new Value();

  // Act
  TRepository.Remove(container, data);

  // Assert
  Assert.Equal(expected, container.Values.Count);
}
```

Осталось только немного поменять тест с существующим элементом

```c#
public void Remove(SomeType<T> container, T data) {
  if (data == null) return;

  var deleteT = container.FirstOrDefault(c => c.SomeField == data.SomeField);
  if (deleteT == null) return;

  container.Values.Remove(deleteT); // очень много связанной логики и т.п.
}

[Fact]
public void Remove_Exist_Removed() {
  // Arrange
  var expected = container.Values.Count;
  var someField = container.Values.Get(1).SomeField;
  var data = new Value(someField);

  // Act
  TRepository.Remove(container, data);

  // Assert
  Assert.Equal(expected - 1, container.Values.Count);
}
```

Все, мы убили мутации, отрефакторили приложение и все ок? или нет?
Нам добавилась еще одна мутация эквивалентная. Какая? SingleOrDefault()
Ага, мы совсем не подумали, что бы будем делать, если у нас в контейнере несколько объектов на удаление, хм. Рефакторим.

```c#
public void Remove(SomeType<T> container, T data) {
  if (data == null) return;

  var deleteT = container.SingleOrDefault(c => c.SomeField == data.SomeField);
  if (deleteT == null) return;

  container.Values.Remove(deleteT); // очень много связанной логики и т.п.
}

[Fact]
public void Remove_ExistDouble_ThrowsException() {
  // Arrange
  var expected = container.Values.Count;
  var someField = container.Values.Get(1).SomeField;
  var data = new Value(someField);

  // Act
  Assert.Throws<InvalidOperationException>(() => _TRepository.Remove(container, data));


  // Assert
  Assert.Equal(expected, container.Values.Count);
}
```

Все, теперь все отлично. А могли просто удалить строку =)

По опыту получилось так, что работа с каждой эквивалентной мутацией всегда заканчивается рефакторингом кода.

Значительно сложнее мутанты, результаты работы которых вы не можете проверить напрямую, т.е. ваши методы не являются чистыми или очень много каскадирования по стеку и результаты многократно трансформируются.

Как пример могу привести пример любых вариантов генерации данных, у вас есть алгоритм генерации данных, алгоритм проверки данных на валидность.

Вы генерируете данные, например номера банковских карт или карт лояльности, потом проверяете их своим же валидатором, но это не совсем верно с точки зрения тестов, юнит тестирование призывает вас тестировать модули изолированно друг от друга.

Кол-во мутаций в таких алгоритмах может быть велико. Вы ничего не сможете с этим  поделать

## Stryker для dotnet core и как он работает, настройка и запуск

Теперь рассмотрим инструмент, который позволит нам использовать мутационное тестирование.

Stryker - это порт js библиотеки на net core, open source проект, только начинающий набирать популярность и находящийся в бета версии. Ссылка на экране.

У страйкера есть особенности:

- он покрыт тестами, в том числе мутационными =)
- он работает сейчас только для .NET Сore
- он поддерживается только общие мутации и LINQ мутации, ООП мутаций нет
- есть поддержка конфигурации
- есть отключение мутаторов
- есть ограничение на проекты или на файлы
- есть флаг красного била и настроенном уровне покрытия

Сейчас в работе несколько очень полезных вещей

- Работа над а Full Framework в процессе
- Такой же html отчет как во js фреймворке
- Экспорт coverage файла для интеграции с IDE
- Блоковая мутация

Как работает страйкер?

- Самое первое что делает клиент - это запускает все тесты для проверки, что они все выполнились корректно и замеряет время выполнения
- Программа получает из проекта, которые ему передали все файлы с кодом и на каждом из них с помощью CSharpSyntaxTree или библиотеки Microsoft.CodeAnalysis.CSharp строит дерево выражений
- C помощью рекурсивной процедуры Mutate проходит по всем нодам дерева и генерирует мутации по правилам мутаций для определенных выражений, все мутированные деревья попадают в общую коллекцию мутаций
- По всем мутированным коллекциям проходит CSharpCompilation, которые собирает IL для дерева и все помещается в MemoryStream
- После данные из MemoryStream инъектируются в файловую систему в тестовый проект.
- После с помощью TPL n executors создается процесс операционной системы
bash >> dotnet test --no-build --no-restore --files ...
- От testrunner приходит ответ и он отправляется репортеру, который выводит его в места хранения
- После этого собирается статистика и проект завершается с кодом 1 или 0.

Вы можете реализовать свой мутатор, для этого необходимо реализовать интерфес через

```c#
interface IMutator {
  IEnumerable<Mutation> Mutate(SyntaxNode node);
}

abstract class MutatorBase<T> where T : SyntaxNode {
  abstract IEnumerable<Mutation> ApplyMutations(T node);

  public IEnumerable<Mutation> Mutate(SyntaxNode node) {
  if (node is T tNode) return ApplyMutations(tNode);
    else return Enumerable.Empty<Mutation>();
  }
}
```

Вот как реализован мутатор boolean

```c#
Dictionary<SyntaxKind, SyntaxKind> _kindsToMutate { get; }

BooleanMutator() {
  _kindsToMutate = new Dictionary<SyntaxKind, SyntaxKind> {
    {SyntaxKind.TrueLiteralExpression, SyntaxKind.FalseLiteralExpression },
    {SyntaxKind.FalseLiteralExpression, SyntaxKind.TrueLiteralExpression }
  };
}

override IEnumerable<Mutation> ApplyMutations(LiteralExpressionSyntax node) {
  if (_kindsToMutate.ContainsKey(node.Kind())) {
    yield return new Mutation() {
      OriginalNode = node,
      ReplacementNode = SyntaxFactory.LiteralExpression(_kindsToMutate[node.Kind()]),
            DisplayName = "Boolean mutation",
            Type = Mutator.Boolean
    };
  }
}
```

## Критерии применимости мутационного тестирования

Давайте разберем какие критерии применимости есть у мутационного тестирования

Они простые, вы помните критерии для того, что бы тесты убили мутанта

1. Первый и основной критерий - покрытие кода тестами, оно должно приближаться к 80-90%, без этого тесты не смогут покрыть мутировавший метод и вы погрязнете в живых мутациях и по факту вам придется написать тесты, которые их убьют. Вы скажете, ок - ладно, слава богу, у меня нет такого покрытия и все это не про меня. Отвечу 80-90% необходимо покрытие базового функционала, за которым стоят дорогие ошибки. Выносите этот код в отдельную библиотеку или проект и покрывайте его тестами и проверяйте их на мутации. По частям.
2. Ваши тесты должны работать очень быстро, иначе время, затраченное на мутационное тестирование будет слишком велико, так как на каждый мутант запускается всех тестов. В библиотеке мне пришлось разделить тесты на несколько проектов, что бы сделать ту часть тестов, которые покрываются мутациями, очень быстрыми.
3. Ваш код должен состоять из чистых маленьких методов, тогда мутации легче обернуть тестами и проще поддерживать
4. Маленький и немаловажный момент - если вы до сих пор используете .NET Framework - переходите на .NET Core, к сожалению основной инструмент работает только на нем. Вы можете компилировать библиотеки в net standard 1.1 для совместимости с NET 4.5.1 и выше.
5. Мутационное тестирование для парней сильных духом, эксперементаторов и энтузиастов - это обязательный параметр, иначе вам будет больно.

## А давайте воткнем мутационное тестирование в pipeline

Можно ли запустить мутационное тестирование в pipeline? Можно, конечно, инструмент предлагает счетчики при которых ваш билд будет красным, есть подробный вывод мутаций. Конечно, ваши билд сервера должны быть способны выдерживать нагрузки. И вы готовы разбираться с ошибками.
У stryker есть библиотека для работы с Azure Pipeline провайдером.

Насчет pipeline спрошу у вас. А кто из вас тестирует pipeline? Звучит странно, неправда ли?

Но иногда это необходимо делать. Для билда проектов на гитхабе я использую AppVeyor, который позволяет подключать собственные скрипты сборки, каждая ветка билдится и тестируется. Все была настроено, но в один момент на билд агентах поменяли логику обработки ошибок и для кастомных скриптов необходимо было включить флаг. А его не было. Воткнув мутационное тестирование для тестов в pipeline обнаружилось, что 4 месяца примерно Flurl собирался, тестировался и выкатывался в nuget при сломанных 4 тестах. Помните я говорил, про окружение linux и windows? А статус билда был зеленый. Так что и ваши сервера сборки необходимо тестировать.

А вот для интеграционных сложных тестов, мутационное тестирование, скорее всего, не взлетит, так как потребует адских ресурсов для вычислений и прогона по 100 раз всех автотестов. Хотя может у кого хватит умения это применить.

## Выводы. Какие я результаты получил при применении MDD

Мутационное тестирование можно использовать для

- мутационное тестирование классно подходит при работе с парадигмой TDD при написании новых тестов и единократного аудита текущей системы
- для поиска неочевидных проблемных мест и оптимизации самых юнит тестов.
- аудита кода внешних разработчиков, когда к вам компанию код приходит как белая коробка. Как правило для кода, написанного под заказ ставится условие написания тестов с большим покрытием.
- для тестирования собственных разработчиков и проверки себя как разработчика тестов, насколько вы качественно пишите код
- для погружения новичков в проект, дает возможность разобраться с многими участками кода на практике.
