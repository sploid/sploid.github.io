<p align="right" width="100%"><a href="https://sploid.github.io/">В начало</a></p>
<p align="right" width="100%"><a href="https://sploid.github.io/ptrs/">Английская версия этой статьи</a></p>

<p align="center" width="100%">Cмарт-поинтеры и управлению памятью в Qt4</p>

Привет всем.

Смарт-поинтеры является очень важным механизмом управления временем жизни объектов. В Qt присутствует модель управления временем жизни объектов, когда объекты наследуются от базового класса QObject и задается “родство” — parent/child. При удалении объекта, он удаляет всех своих child. Эта модель управления временем жизни объектов очень хорошо сочетается с технологией взаимодействия между объектами “Signals & Slots” и при использовании смарт-поинтеров могут возникнуть тяжело отлаживаемые баги.

В данной статье будет рассмотрен смарт-поинтер QScopedPointer, который является “облегченной” версией QSharedPointer. QScopedPointer введен в версии 4.6.

В boost присутствуют аналоги Qt-шным смарт-поинтерам:
QSharedPointer — boost::shared_ptr
QWeakPointer — boost::weak_ptr
QScopedPointer — boost::scoped_ptr

# Пример 1

Простая программа, у которой не удаляется объект m_socket и соответственно, при завершении приложения, не происходит отключения сокета:

```cpp
class test_socket : public QTcpSocket
{
public:
   test_socket( QObject * parent )
       : QTcpSocket( parent )
   {
   }
   virtual ~test_socket( )
   {
       qDebug( ) << "~test_socket";
   }
};

class test_scop_ptr_obj : public QObject
{
   Q_OBJECT
   test_socket * m_socket;
public:
   test_scop_ptr_obj( )
       : m_socket( new test_socket( 0 ) )
   {
       m_socket->connectToHost( "www.ru", 80 );
       connect( m_socket, SIGNAL( stateChanged ( QAbstractSocket::SocketState ) ),
               SLOT( on_state_changed( QAbstractSocket::SocketState ) ) );
   }
   virtual ~test_scop_ptr_obj( )
   {
       qDebug( ) << "~test_scop_ptr_obj";
   }
private Q_SLOTS:
   void on_state_changed( QAbstractSocket::SocketState socket_state )
   {
       qDebug( ) << socket_state;
   }
};

int main( int argc, char** argv )
{
   QCoreApplication a( argc, argv );
   test_scop_ptr_obj obj;
   QTimer::singleShot( 3000, &a, SLOT( quit( ) ) );
   return a.exec( );
}
```

После запуска программы получаем такой вывод:

```txt
QAbstractSocket::ConnectingState
QAbstractSocket::ConnectedState
~test_scop_ptr_obj
```

# Пример 2

Теперь изменим строку:

```cpp
    : m_socket( new test_socket( 0 ) )
```

на строку:

```cpp
    : m_socket( new test_socket( this ) )
```

и получим вывод:

```txt
QAbstractSocket::ConnectingState
QAbstractSocket::ConnectedState
~test_scop_ptr_obj
~test_socket
```

# Пример 3

Добавим смарт-поинтер и теперь класс test_scop_ptr_obj будет выглядеть так:

```cpp
class test_scop_ptr_obj : public QObject
{
   Q_OBJECT
   QScopedPointer< test_socket > m_socket;
public:
   test_scop_ptr_obj( )
       : m_socket( new test_socket( 0 ) )
   {
       m_socket->connectToHost( "www.ru", 80 );
       connect( m_socket.data( ), SIGNAL( stateChanged ( QAbstractSocket::SocketState ) ),
                SLOT( on_state_changed( QAbstractSocket::SocketState ) ) );
   }
   virtual ~test_scop_ptr_obj( )
   {
       qDebug( ) << "~test_scop_ptr_obj";
   }
private Q_SLOTS:
   void on_state_changed( QAbstractSocket::SocketState socket_state )
   {
       qDebug( ) << socket_state;
   }
};
```

Получаем вывод:

```txt
QAbstractSocket::ConnectingState
QAbstractSocket::ConnectedState
~test_scop_ptr_obj
~test_socket
QAbstractSocket::UnconnectedState
```

# Анализ

Пример 1 не интересен, так как в этом примере происходят утечки памяти.
Пример 2 и 3 отличаются технологией освобождения ранее выделенной памяти. И в том и другом примере происходит удаление объекта класса test_socket, но в 3-ем примере происходит вызов функции on_state_changed уже после вызова деструктора ~test_scop_ptr_obj, что является ошибкой. Перед удалением своих child, объект parent делает disconnect своим сигналам и слотам, поэтому в примере 2 и не приходит сигнал stateChanged к объекту test_scop_ptr_obj.

# Итоги

Для исправления ошибки в примере 3 необходимо в деструктор test_scop_ptr_obj дописать отключение сигналов удаляемого объекта:

```cpp
virtual ~test_scop_ptr_obj( )
{
    m_socket->disconnect( );
    qDebug( ) << "~test_scop_ptr_obj";
}
```

В этом случае вывод будет таким-же как и в примере 2.
