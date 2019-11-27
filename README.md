# Understanding Syncronization using Python 3
Before diving into syncronization, let's get a quick recap at the differences between threads and processes. 

| Process | Thread |
| ------ | ------ |
| It's an executing instance of an application | It's a path of execution within a process |
| Run in independent memory space | Run in shared memory space |
| Context swithcing in involves PCB switch and MMU reconfiguration which is essentially flushing the TLB (virtual-phy memory) which makes memory access more expensive | It only has to do the PCB switch but can still use the same memory space and hence is relatively cheap. MMU doesn't need to be reconfigured as it's shared.|

We take advantage of multiple threads to concurrently complete tasks in a process. For example, in the browser process there might be one thread to take handle input in the address bar, one to handle network requests and another to paint the screen. The OS/Hardware (Hardware vs S/W context swithcing) efficiently switches between these threads to optimally utilize the CPU. For the scope of this discussion, we will limit ourselves to a single core machine (We will discuss concurrency and not parallelism).
Since threads of the same process might be accessing the same resource(Eg: the same variable), we might end up with inconsistent data. The shared segment of code which  might lead to inconsistent data is known as the critical section.

Let's take an example.
In the following code we use python's [threading library](https://docs.python.org/3/library/threading.html) to create multiple threads for the exection of `testThread` method
```python
import threading
import time
total = 0
def testThread(num):
	global total
	temp = total
	#Halt the thread for 2 seconds where num is 3
	if num==3:
		time.sleep(2)
	total = temp + num

if __name__ == '__main__':
	#Create 5 threads for testThread(num) method where num = 0, 1, 2, 3, 4
	threads = [threading.Thread(target=testThread, args=(i,)) for i in range(5)]
	#Start all threads
	for thread in threads:
		thread.start()
	#Block the current execution till all spawned threads terminate
	for thread in threads:
		thread.join()
	#print the total
	print("The total is: ", total)
```
Since we are adding each `i` to the global variable `total`, our final balance should ideally be the sum of all i's. i.e. 0+1+2+3+4 = 10
However, after executing the code above, we get the output `The total is:  6`
This discerpancy arises due to a  between threads modifying the global variable `time`. To know more about the details of how and why it is arising, please read this [link](https://en.wikipedia.org/wiki/Race_condition#Software).

Now, in order to solve the race condition we have three different ways.
#### 1. Spin Locks
Here the thread repetedly checks if a lock is available. Since the thread remains active but is not performing a useful task, the use of such a lock is a kind of busy waiting.
In the above code, we can redefine the method `testThread` using a sping lock as follows. Note that although it solves the concurrency issue, it causes a lot of overhead on the CPU to continously check if a lock is available.

```python
spin_lock = False
def testThread(num):
	global spin_lock
	#thread 'spins' in this while condition till spin_lock is made available 
	while spin_lock:
		pass
	spin_lock = True
	global total
	temp = total
	if num==3:
		time.sleep(2)
	total = temp + num
	spin_lock = False
```
#### 2. Locks (Blocking)
Locks block a thread if the lock is not available. Blocking means that the thread is suspended by the OS and will be automatically notified when it's corresponding lock becomes available. It's doesn't waste CPU cycles to keep checking if the lock is made available. It is also known as a binary semaphore. Let us look at its implimentation

```python
import threading
lock = threading.Lock()

def testThread(num):
	lock.acquire()
	global total
	temp = total
	if num==3:
		time.sleep(2)
	total = temp + num
	lock.release()
```

#### 3. Semaphores 
It's an internal counter which is decremented for each acquire() call and incremented for each release() call. It's essentially used to syncronize over a limited resource like a server with limited capacity to handle concurrent requests.

Here let's take an example of a server where we can have 5 clients concurrently. 

```python
import threading
import time

class Server:
    def __init__(self, max_connections=3):
        # Note: using print_lock to prevent multiple threading 
        #          from writing concurrently in the output buffer.
        self.sem = threading.BoundedSemaphore(max_connections)
        self.print_lock = threading.Lock()

    def handle_heavy_request(self, thread_no):
        self.sem.acquire()
        time.sleep(3)
        self.sem.release()

        self.print_lock.acquire()
        print("complete: ", thread_no)
        self.print_lock.release()

if __name__ == '__main__':
    server_obj = Server()
    threads = [threading.Thread(target=server_obj.handle_heavy_request, args=(i,)) for i in range(10)]
    for thread in threads:
        thread.start()
```
