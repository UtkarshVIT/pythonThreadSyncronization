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
Since we are adding each i to the global variable total, our final balance should ideally be the sum of all i's. i.e. 0+1+2+3+4 = 10
However, after executing the aobve code we get the output `The total is:  6`

