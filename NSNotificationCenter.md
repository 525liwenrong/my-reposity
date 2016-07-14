## NSNotificationCenter

### 获取通知中心

- 每个程序都会有一个默认的通知中心。为此，NSNotificationCenter提供了一个类方法来获取这个通知中心：
(NSNotificationCenter *)defaultCenter
获取了这个默认的通知中心对象后，我们就可以使用它来处理通知相关的操作了，包括注册观察者，移除观察者、发送通知等。

###添加观察者

- 如果想让对象监听某个通知，则需要在通知中心中将这个对象注册为通知的观察者。早先，NSNotificationCenter提供了以下方法来添加观察者：
-(void)addObserver:(id)notificationObserver
           selector:(SEL)notificationSelector
               name:(NSString *)notificationName
             object:(id)notificationSender

这个方法带有4个参数，分别指定了通知的观察者、处理通知的回调、通知名及通知的发送对象。这里需要注意几个问题：

- notificationObserver不能为nil。
- notificationSelector回调方法有且只有一个参数(NSNotification对象)。
- 如果notificationName为nil，则会接收所有的通知
- 如果notificationSender不为空，则接收所有来自于notificationSender的所有通知)。如果notificationSender为nil，则会接收所有notificationName定义的通知；否则，接收由notificationSender发送的通知。
- 监听同一条通知的多个观察者，在通知到达时，它们执行回调的顺序是不确定的，所以我们不能去假设操作的执行会按照添加观察者的顺序来执行。

- (id<NSObject>)addObserverForName:(NSString *)name
                            object:(id)obj
                             queue:(NSOperationQueue *)queue
                        usingBlock:(void (^)(NSNotification *note))block

- name和obj为nil时的情形与前面一个方法是相同的。
如果queue为nil，则消息是默认在post线程中同步处理，即通知的post与转发是在同一线程中；但如果我们指定了操作队列，情况就变得有点意思了，我们一会再讲。
- block块会被通知中心拷贝一份(执行copy操作)，以在堆中维护一个block对象，直到观察者被从通知中心中移除。所以，应该特别注意在block中使用外部对象，避免出现对象的循环引用，这个我们在下面将举例说明。
如果一个给定的通知触发了多个观察者的block操作，则这些操作会在各自的Operation Queue中被并发执行。所以我们不能去假设操作的执行会按照添加观察者的顺序来执行。
该方法会返回一个表示观察者的对象，记得在不用时释放这个对象。
下面我们重点说明一下第2点和第3点。

- 当我们指定一个Operation Queue时，不管通知是在哪个线程中post的，都会在Operation Queue所属的线程中进行转发.

![] (/Users/liwenrong/Desktop/1.png)

- 可以看到，消息的post与接收处理并不是在同一个线程中。如上面所提到的，如果queue为nil，则消息是默认在post线程中同步处理，大家可以试一下。
- 由于使用的是block，所以需要注意的就是避免引起循环引用的问题.

![](/Users/liwenrong/Desktop/3.png)


###移除观察者

- Notice:在iOS9之后，NSObject也会像ViewController一样在dealloc时挪用[[NSNotificationCenter defaultCenter]removeObserver:self]办法，在iOS9之前的不会挪用，需要本人写。

- 与注册观察者相对应的，NSNotificationCenter为我们提供了两个移除观察者的方法。它们的定义如下：

- -(void)removeObserver:(id)observer;
- -(void)removeObserver:(id)observer name:(nullable NSString *)aName object:(nullable id)anObject;

- 前一个方法会将notificationObserver从通知中心中移除，这样notificationObserver就无法再监听任何消息。而后一个会根据三个参数来移除相应的观察者。

 这两个方法也有几点需要注意：

- 由于注册观察者时(不管是哪个方法)，通知中心会维护一个观察者的弱引用，所以在释放对象时，要确保移除对象所有监听的通知。否则，可能会导致程序崩溃或一些莫名其妙的问题。

- 对于第二个方法，如果notificationName为nil，则会移除所有匹配notificationObserver和notificationSender的通知，同理notificationSender也是一样的。而如果notificationName和notificationSender都为nil，则其效果就与第一个方法是一样的了。大家可以试一下。

- –removeObserver:适合于在类的dealloc方法中调用，这样可以确保将对象从通知中心中清除；而在viewWillDisappear:这样的方法中，则适合于使用-removeObserver:name:object:方法，以避免不知情的情况下移除了不应该移除的通知观察者。例如，假设我们的ViewController继承自一个类库的某个ViewController类(假设为SKViewController吧)，可能SKViewController自身也监听了某些通知以执行特定的操作，但我们使用时并不知道。如果直接在viewWillDisappear:中调用–removeObserver:，则也会把父类监听的通知也给移除。

### post消息
- -(void)postNotification:(NSNotification *)notification;
- -(void)postNotificationName:(NSString *)aName object:(nullable id)anObject;
- -(void)postNotificationName:(NSString *)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;
- 我们可以根据需要指定通知的发送者(object)并附带一些与通知相关的信息(userInfo)，当然这些发送者和userInfo可以封装在一个NSNotification对象中，由- postNotification:来发送。注意，- postNotification:的参数不能为空，否则会引发一个异常.
- 每次post一个通知时，通知中心都会去遍历一下它的分发表，然后将通知转发给相应的观察者。
- 通知的发送与处理是同步的，在某个地方post一个消息时，会等到所有观察者对象执行完处理操作后，才回到post的地方，继续执行后面的代码.

![](/Users/liwenrong/Desktop/2.png)

### 对于使用NSNotificationCenter，最后总结一些小建议：

- 在需要的地方使用通知。
- 注册的观察者在不使用时一定要记得移除，即添加和移除要配对出现。
- 尽可能迟地去注册一个观察者，并尽可能早将其移除，这样可以改善程序的性能。因为，每post一个通知，都会是遍历通知中心的分发表，确保通知发给每一个观察者。
- 记住通知的发送和处理是在同一个线程中。
- 使用-addObserverForName:object:queue:usingBlock:务必处理好内存问题，避免出现循环引用。
- NSNotificationCenter是线程安全的，但并不意味着在多线程环境中不需要关注线程安全问题。不恰当的使用仍然会引发线程问题。
