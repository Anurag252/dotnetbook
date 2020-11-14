# Практика отпимизаций по работе с памятью

В своей практике оптимизаций я сделал интересный, хотя и логичный вывод. Если мы избавлемся от срабатываний GC, приложение работает быстрее. Звучит как: "ну естествено! Кто бы спорил-то?". Однако, далеко не каждый ответит на вопрос: "на сколько"? 

Мой пример, конечно же, будет несколько наивен.. Но когда я работал над протоколом Samba и мной была переписана библиотека `SMBLibrary`, то результат при той же структуре кода, но другими способами по работе с памятью составил `-41%` к времени исполнения. Вдумайтесь: `41%` на аллокацию, GC и неоптимальный выбор структур данных.

Конечно же, основная масса времени уходила на работу со структурами данных типа `Dictionary`, `List`, `Queue`, т.к. заполнение шло без выставленного `capacity`, однако и на траффик мелких объектов уходило много времени.

Траффик объектов приводит, как мы помним, к GC0. Однако если траффик реально плотный, то приложение попросту не успеет освободить ссылки и объекты вместо ухода на покой в GC0 сделают это в GC1. К чему это приведет? К тому, что GC будет работать дольше. Т.е. ему придется переработать большой участок памяти для понимания, на какие объекты более нет ссылок. Плюс к этому, если на эти обекты будут ссылаться какие-то старые коллекции или старые объекты (из мира второго поколения), то в игру вступает таблица карт. Что включает еще большие расходы на анализ. Звучит как-то не очень. Что же делать? Звучит иной раз как безнадёга: ведь и сущность надо выделить. Причем не структуру, а именно класс. И память лупит по полной программе. Однако, если вспомнить главу "Время жизни сущностей", то мы вспомним, что по сути время жизни -- это несколько выртуальоне понятие и мы можем сделать еще один слой виртуализации понятия времени жизни сущности. Например, так: 

```csharp
public static class Pool<T> where T : class, new()
{
    private Queue<T> objects;
    private const MinPoolSize = 32;

    static Pool()
    {
        objects = new Queue<T>(MinPoolSize);
    }

    public static T Get()
    {
        if(objects.TryDeqeue(out var instance))
        {
            return instance;
        }
        return new T();
    }

    public static void Return(T instance) => objects.Enqeue(instance);
}

var instance = Pool<Foo>.Get();
Pool<Foo>.Return(instance)
```