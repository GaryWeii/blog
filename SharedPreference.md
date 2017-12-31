# SharedPreference

OnSharedPreferenceChangeListener
可以实现对某个shardPreferences实例的监听。当监听对象修改、添加、删除时，该监听的方法将在主线程中被调用。


Editor
用于批处理sharedPrefences的修改操作，需要调用commit() 或是apply() 方法才会生效。

SharedPreferecnes实例由ContextImpl持有，对于每个name所对应的sp实例，在整个应用中只有一个实例。

commit()
        commit方法直接在当前线程完成，因此可以返回写操作是否成功。但如果数据量大，则当前线程可能需要等待较长的时间，直到文件写入完成。
当有多个任务并行时，即之前有未完成的apply提交时，会将任务放到QueuedWork任务队列中，并堵塞当前线程直至写文件操作完成。

apply()
    apply操作将任务提交至QueuedWork的任务队列中，由一个单线程池来负责执行文件的写入操作。
因此，该操作不会堵塞当前线程。但是因为写入文件是异步操作，因此该方法不会返回操作是否成功。
也正是因为写入操作是异步的，如果在写入任务执行之前，应用关闭了，则所有未开始执行的任务状态都会丢失。

为了解决这一问题，apply方法将一个awaitCommit的任务放入QueuedWork中，该任务只是一个countDownLatch用来阻塞线程。
当ActivityThread中的handlePauseActivity、handleStopActivity等一系列生命周期相关方法被调用时，将会导致主线程阻塞以等待sp写文件任务完成。


结论，不管是commit还是apply方法，在写入sp的数据量较大时，都会或是说可能会阻塞线程，甚至引起ANR。所以SharedPreferences不适合存储过多过复杂的数据。
