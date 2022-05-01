# BlockingQueue家族(常用系列)
## ArrayBlockingQueue
## LinkedBlockingQueue
## SynchronousQueue
只能容纳一个元素，同时使用了两种模式来管理元素，一种是使用先进先出的队列，一种是使用后进先出的栈，使用哪种模式可以通过构造函数来指定。
## PriorityBlockingQueue
## DelayQueue  
延迟队列，使得插入其中的元素在延迟一定时间后，才能获取到，插入其中的元素需要实现`java.util.concurrent.Delayed`接口。该接口需要实现`getDelay()`和`compareTo()`方法。`getDealy()`返回0或者小于0的值时，`delayedQueue`通过其`take()`方法就可以获得此元素。`compareTo()`方法用于实现内部元素排序，一般情况，按元素过期时间的优先级进行排序是比较好的选择。


# BlockingDeque
## LinkedBlockingDeque
