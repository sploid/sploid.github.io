<p align="right" width="100%"><a href="https://sploid.github.io/">To the begining</a></p>
<p align="right" width="100%"><a href="https://sploid.github.io/ptrs/">Russian version of this article</a></p>

# Smart-pointers and memory management in Qt4

Hi everybody.

Smart points are very important for managing the lifetime of objects. Qt has a model for managing the lifetime of objects, when objects are inherited from the base class QObject and the ”relationship” parent/child is set. It deletes all children of the project. This model of object lifetime management is successfully combined with the technology of interaction between “Signals & Slots” objects, and when using smart pointers, hard-to-debug bugs may occur.

This article will review the QScopedPointer, a “lightweight” version of QSharedPointer. QScopedPointer is introduced in version 4.6.

Boost has analogues to Qt-based smart pointers:
QSharedPointer — boost::shared_ptr
QWeakPointer — boost::weak_ptr
QScopedPointer — boost::scoped_ptr

## Example 1

A simple program that does not delete the m_socket object and, accordingly, when the application is terminated, the socket is not disconnected:

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

After starting the program, we get the following output:

```txt
QAbstractSocket::ConnectingState
QAbstractSocket::ConnectedState
~test_scop_ptr_obj
```

## Example 2

Now let us change the line:

```cpp
    : m_socket( new test_socket( 0 ) )
```

per line:

```cpp
    : m_socket( new test_socket( this ) )
```

and we will get the output:

```txt
QAbstractSocket::ConnectingState
QAbstractSocket::ConnectedState
~test_scop_ptr_obj
~test_socket
```

## Example 3

Let us add a smart pointer and now the test_scop_ptr_obj class will look like this:

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

We get the output:

```txt
QAbstractSocket::ConnectingState
QAbstractSocket::ConnectedState
~test_scop_ptr_obj
~test_socket
QAbstractSocket::UnconnectedState
```

## Analysis

Example 1 is not interesting to us because memory leaks occur in this example.
Examples 2 and 3 differ in the technology of freeing previously allocated memory. In both examples, the object of the test_socket class is deleted, but in the 3rd example, the on_state_changed function is called after calling the ~test_scop_ptr_obj destructor, which is an error. Before deleting children, the parent object disconnects its signals and slots, which is why in Example 2 the stateChanged signal does not arrive to the test_scop_ptr_obj object.

## Results

To correct the error in Example 3, it is necessary to add disabling signals of the deleted object to the test_scop_ptr_obj destructor:

```cpp
virtual ~test_scop_ptr_obj( )
{
    m_socket->disconnect( );
    qDebug( ) << "~test_scop_ptr_obj";
}
```

In this case, the output will be the same as in Example 2.
