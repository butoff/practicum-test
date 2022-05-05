# Ревью тестовых заданий Практикума

## Задание 1

Что ж, этот код будет работать, это уже хорошо! Давай посмотрим, как его можно улучшить.
Сейчас основная проблема в том, что эту функцию неудобно использовать из вызывающего кода, и
легко допустить ошибку в месте вызова. Что, если пользователь этой библиотеки (смежник, партнер,
случайный человек из интернета или даже сам автор) допустит ошибку в параметре: ```turnTo("Norht")```
или ```turnTo("north")```?

В этом случае функция не выполнит то, что от нее хотел клиент. И хуже всего то, что об этом мы
узнаем только в момент выполнения. А можем и вовсе не узнать, если невнимательно посмотрим на
вывод программы.

Последнюю проблему легко решить, если вместо печати ошибки мы будем бросать исключение. Аварийно
завершить программу в случае явной ошибки - почти всегда хорошая идея, так можно выявить проблему
на ранней стадии, не дожидаясь распространения ошибки и сэкономить кучу времени на отладке.

Но есть решение еще лучше, которое позволяет отсечь недопустимые значения параметра еще на этапе
компиляции. Это одно из средств языка Kotlin (и Java) - специальный вид классов с фиксированным
набором экземпляров. Прежде чем читать дальше, предлагаю вспомнить, что это за средство.

Итак, речь о enum - специальном типе с фиксированным набором значений. Enum обладает богатыми
возможностями, которые делают программы с его использованием очень выразительными, но нам в этой
задаче требуется самый минимум - просто объявить набор значений и передавать их в качестве аргумента.

```kotlin
enum class Direction { NORTH, EAST, SOUTH, WEST }

fun turnTo(direction: Direction) {
```

Теперь мы просто не можем передать в функцию неверное значение - программа просто не скомпилируется.

Еще неплохо заменить java-стайл цепочку выражений if-else на одно выражение when:

```kotlin
fun turnTo(direction: Direction) {
    when (direction) {
        Direction.NORTH -> northAction()
        Direction.SOUTH -> southAction()
        Direction.EAST -> eastAction()
        Direction.WEST -> westAction()
    }
}
```

О других возможностях enum классов можно (и нужно!) почитать в книге Effective Java, но позже, по мере
приобретения опыта.

## Задание 2

Как и предыдущее, это задание выполнено корректно, функция делает то, что требуется.
Однако здесь содержится проблема, связанная с производительностью. Прежде, чем читать
дальше, предлагаю найти эту проблему, связанную с многократным созданием и уничтожением
объектов.

Итак, проблема кроется в многократном создании и уничтожении строки output. Посмотрим
внимательно на выражение

```kotlin
output = output + "Имя: " + user.name + ", Возраст: " + user.age + "\n"
```

Здесь происходит следующее:

1) Из строки output и еще нескольких строк создается новая строка, это делается функцией-
оператором plus (https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-string/plus.html)
При этом старая строка output никуда не исчезает, не переиспользуется, не входит физически
в состав новой строки. После вызова функции в памяти одновременно содержатся два объекта:
старая строка и новая.

2) Ссылочной переменной output присваивается эта новая строка. Несмотря на кажущуюся простоту,
нужно рассмотреть этот процесс во всех подробностях. Перед выполнением присваивания переменная
output указывает на старую строку. При этом на шаге (1) сконструирован объект новой строки, он
находится в памяти и готов к использованию. Остается привязать к нему ссылку, что и делается
оператором присваивания.

3) Что произойдет со старой строкой после выполнения оператора присваивания? Единственная ссылка
output теперь указывает на новую строку, других ссылок на старую строку нет. Как мы знаем,
такие объекты, на которые не указывает ни одна ссылка, становятся добычей сборщика мусора. При
большом размере входных данных - списка users, сборщик мусора будет неоднократно вызываться в
процессе выполнения, а сборка мусора - затратный процесс и будет приводить к медленной работе.

Таким образом, падение производительности будет вызвано 1) многократным копированием строки
output и 2) большим расходом памяти и частой сборкой мусора.

Для эффективных, быстрых программ требуется свести к минимуму создание и уничтожение объектов.
К счастью, в данной задаче можно вместо неизменяемого класса String использовать специально
предназначенный для создания строк класс ```StringBuilder```
(https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/-string-builder/)

```kotlin
val output = StringBuilder()
for (user in users) {
    output.append("Имя: ")
    output.append(user.name)
    output.append(", Возраст: ")
    output.append(user.age)
    output.append("\n")
}
return output.toString()
```

Так как StringBuilder специально сделан для решения таких задач, можно надеяться, что его реализация
оптимально использует память и внутри себя не создает и не выбрасывает в мусор больших объектов.

Итак, код стал эффективнее, но менее читаем. На самом деле, для решения проблемы производительности
нам достаточно избавиться только от многократного присваивания переменной output в цикле. Правую часть
можно оставить состоящей из конкатенированных составных частей. Если конкатенация происходит не в
цикле, то компилятор сам распознает такие места и заменяет их с использованием того же StringBuilder.
А лучше пойти еще дальше и применить интерполяцию строк:

```kotlin
for (user in users) {
    output.append("Имя: ${user.name}, Возраст: ${user.age}\n");
}
```

Можно ли еще улучшить этот код? Да! Заметим, что результатом функции являются записанные в каждой
строке строковые представления объектов User. Что в Java используется для строкового представления
объекта? Это метод toString() класса Object. Если мы создаем какой-то свой класс, почти всегда
полезно переопределить для него метод toString(). В Kotlin есть data классы, которые делают это
автоматически, но иногда это не то, что нужно. Итак, в классе User добавим

```kotlin
override fun toString() = "Имя: $name, Возраст: $age"
```

Тогда цикл примет вид
```kotlin
for (user in users) {
    output.appendln(user)
}
```

Теперь небольшое открытие: функция ```createResultString()``` уже есть в библиотеке Kotlin! Это
```joinToString``` (https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/join-to-string.html)

Так будет выглядеть вызов (придется передать аргумент-сепаратор, так как по умолчанию используется
запятая, а нам нужен перевод строки): ```users.joinToString(separator = "\n")```

Итак, мы почти полностью избавились от своего кода, оставив только ```toString()``` у класса User, и
все благодаря наличию в библиотеке Kotlin подходящей функции. Отсюда можно сделать такой вывод: нужно
хорошо знать не только язык, но и входящие в него стандартные и другие используемые в твоей команде
библиотеки. Полностью изучить все библиотечные средства Java, Kotlin, Android фреймворка и популярных
сторонних библиотек невозможно, но уделять большое время изучению библиотек совершенно необходимо.
Причем не только на нашем курсе, но и постоянно в процессе работы.

На самом деле, это большая удача, что вместо простого использования библиотечной функции мы так
подробно рассмотрели ее устройство и связанные с этим проблемы. Описанное падение производительности
из-за избыточного создания объектов - частая проблема, нужно научиться распознавать такие места в
коде, это непросто без большого опыта.

## Задание 3

К сожалению, в этом коде содержaтся критические ошибки. Во-первых, код не скомпилируется:
второй вызов ```preferences.edit()``` написан с ошибкой, он почему-то принимает в качестве аргумента
лямбду. Аналогичный вызов в onboardingWasShown написан корректно.

Во-вторых, менее заметная ошибка содержится в неверном ключе в том же втором вызове метода.

В-третьих, в обоих случаях у возвращаемого вызовом ```edit()``` объекта ```Editor``` необходимо вызвать
```commit()``` или ```apply()``` для сохранения результатов.

На самом деле, первые две ошибки - следствие серьезной проблемы в дизайне этого кода, который
содержит дублирование. onboardingWasShown и promoWasShown выполняют ту же задачу и различаются
только ключом для хранения. Дублирование - наверное, самая распространенная ошибка в
проектировании, этому уделяется большое внимание в книгах и статьях по дизайну программ:
https://en.wikipedia.org/wiki/Don%27t_repeat_yourself

Предлагается не читая дальше переписать данный код, чтобы устранить дублирование. Можно сделать
это проще, отказавшись от использования Kotlin properties - на данном этапе это даже
предпочтительнее. Иногда использование мощных средств языка приводит к проблемам, которые требуют
еще более сложных решений. На начальном этапе вполне нормально в таких случаях использовать более
простые средства. Если же хочется углубиться в изучение языка, можно все же оставить properties,
тогда для решения следует использовать специальное мощное средство языка Kotlin.

Итак, специально для вынесения общего кода работы с однотипными properties в Kotlin есть механизм
делегатов (Delegated proprties, https://kotlinlang.org/docs/delegated-properties.html) Технология
довольно сложная для начинающих, но если есть время и желание, можно с ней разобраться по
приведенной документации. Также для понимания потребуется знакомство с механизмом рефлексии.
Далее приведу решение нашей проблемы с минимальными комментариями.

```kotlin
// интерфейс для доступа к именам property механизмом рефлексии
import kotlin.reflect.KProperty

// Специальный класс-делегат
class BooleanPreference(private val preferences: SharedPreferences) {

    // Этот метод вызывается при каждом get() делегируемого property
    operator fun getValue(thisRef: Any?, property: KProperty<*>): Boolean {
        // ключ к shared preference формируется автоматически по имени property!
        return preferences.getBoolean("${property.name}_key", false)
    }

    // Этот метод вызывается при каждом set()
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: Boolean) {
        // снова ключ формируется по имени property
        preferences.edit().putBoolean("${property.name}_key", value).apply()
    }
}

class AppPreferences(private val preferences: SharedPreferences) {
    // Теперь реализуем property через делагат, это занимает всего одну строку!
    var onboardingWasShown by BooleanPreference(preferences)
    var promoWasShown by BooleanPreference(preferences)
}
```

Класс-делегат довольно сложен для понимания, но он пишется один раз (возможно, опытными
коллегами). Зато использование его в ```AppPreferences``` предельно простое: всего одна строка,
в которой просто нет места для ошибки! Заметим, что попутно мы избавились от объявления
ключей для достпа к shared preferences - они формируются автоматически по имени property.
Делегаты в языке Kotlin - cложное, но необыкновенно красивое средство!

Можно пойти дальше и сделать код еще более устойчивым к ошибкам. Заметим, что такие настройки,
как ```onboardingWasShow```, меняются только один раз за жизненный цикл программы, и всегда из false
в true. Наш же интерфейс допускает возможность установить значение обратно в false, а значит,
пользователь может по ошибке это сделать. Учитывая это, будет разумно сделать property,
подобные onboardingWasShown, семантически read-only, то есть val, а не var. Если окажется, что
значение не нашлось по ключу (так будет при первом использовании), сразу запишем в настройки
значение true. Со стороны клиента использование будет предельно простое: нужно только читать
property и выполнять необходимое действие, если вернется false. Сохранение в настройки
произойдет внутри класса-делегата. Реализовать такой делегат предлагается самостоятельно.

## Задание 4

К сожалению, это неработоспособный код. Для начала, он не скомпилируется - ошибка в
```this@TimerActivityIn```, такого класса не существует. Далее, в методе ```onCreate``` не вызван
```setContentView```, без этого ```TimerActivity``` не приобретет корневой ```View``` с содержащимся в
нем ```TextView```.

Менее очевидная ошибка состоит в том, что таймер является inner class внутри
```TimerActivity```, а значит, содержит неявную ссылку на содержащий его объект:
(https://kotlinlang.org/docs/nested-classes.html#inner-classes)

Если во время работы таймера содержащая его Activity будет пересоздана
(например, при смене конфигурации - повороте экрана), то очередной callback таймера
будет обновлять уже не существующий ```TextView```. Кроме того, неявная ссылка из inner
класса таймера на outer класс ```TimerActivity``` будет препятствовать утилизации объекта
```TimerActivity``` сборщиком мусора.

Одно из решений - не делать таймер inner классом, а ссылку на ```TextView``` передавать
в таймер явно. Также для показа заключительного тоста мы передадим в таймер
ApplicationContext - это глобальный для нашего процесса контекст, не привязанный к
какой-либо Activity или другому компоненту.

```kotlin
// теперь Timer - не inner class, и не содержит неявной ссылки на TimerActivity
class Timer(time: Long, interval: Long, val applicationContext: Context) : CountDownTimer(time, interval) {

    // вместо неявной ссылки на объект outer класса TimerActivity храним явную ссылку на TextView
    public var textView: TextView? = null

    override fun onTick(millisUntilFinished: Long) {
        textView?.text = getString(R.string.millis_until_finished, millisUntilFinished.toString())
    }

    override fun onFinish() {
        // Toast показываем, используя applicationContext, так мы не зависим от недолгоживущей Activity
        Toast.makeText(applicationContext, R.string.timer_is_finished, Toast.LENGTH_SHORT).show()
    }
}

public class TimerActivity : Activity() {

    companion object {
        private const val MILLIS_IN_SECONDS = 1000L
        private const val INTERVAL = 1 * MILLIS_IN_SECONDS
        private const val TIME = 10 * MILLIS_IN_SECONDS

        private var timer: Timer? = null
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // setContentView(R.layout....)

        if (savedInstanceState == null) {
            // если это новая, а не пересозданная Activity
            timer = Timer(TIME, INTERVAL, applicationContext)
            timer?.start()
        }
        timer?.textView = findViewById<TextView>(R.id.text)
    }

    override fun onDestroy() {
        super.onDestroy()
        timer?.textView = null  // прекратить обновление уже не существующего TextView
    }
}
```

Из этой задачи нужно сделать такие выводы: быть готовым к пересозданию Activity в результате смены
конфигурации и внимательно относиться к inner классам, содержащим неявную ссылку на outer класс.

## Задание 5
