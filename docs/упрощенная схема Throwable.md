
Throwable
├── Error (не обрабатываемые программой, например OutOfMemoryError, StackOverflowError)
└── Exception
    ├── Checked Exceptions (требуют обработки, например, IOException, SQLException)
    |       +--- CloneNotSupportedException (класс, для объекта которого вызывается клонирование, не реализует интерфейс Cloneable)
    |       +--- InterruptedException (поток прерван другим потоком)
    |       \--- ClassNotFoundException (невозможно найти класс)
    |
    └── Unchecked Exceptions (не требуют обработки, например, RuntimeException - RuntimeException является базовым классом для так называемой группы непроверяемых исключений (unchecked exceptions))
        ├── NullPointerException (использование пустой ссылки)
        ├── ArrayIndexOutOfBoundsException (индекс вне границ массива)
        ├── ArithmeticException (исключение, возникающее при делении на ноль)
        ├── NumberFormatException (ошибка преобразования строки в число)
        ├── EntityNotFoundException
        └── IllegalArgumentException (использование неверного аргумента при вызове метода)

---

> `RuntimeException` является базовым классом для так называемой группы непроверяемых исключений (unchecked exceptions) - 
> компилятор не проверяет факт обработки таких исключений и их можно не указывать вместе с оператором throws в объявлении 
> метода. Такие исключения являются следствием ошибок разработчика, например, неверное преобразование типов или выход 
> за пределы массива.