---
title: Distrbuted computation with a raspberry pi cluster(1)
author: Chester Holtz
layout: post
permalink: /blog/post/rpi_distributed_1
categories:
  - Uncategorized
---

I have had a great deal of fun reading about the many adventures of the physicist [Richard Feynman][1]. I read his semi-autobiography *Surely You're Joking Mr. Feynman!* as a kid, watched his video series on computer heuristics as a freshman taking a class on organization of computer systems, and found his physics notes to be invaluable while taking my university's freshman physics series. Recently I have also been reading about his collaboration with the great computer scientist [Dany Hillis][2] - coincidentally advised by 3 other famous computer scientists - on the [Connection Machine][3] - a parallel arangement of multiple supercomputers. Additionally, one of my mentors at The University of Rochester just retired, and while browsing his library, I found that he had worked on the development of software for the [BBN Butterfly Processor][4] - one of the largest parallel computers of the 1980s. These factors all served to exite my interest in parallel computation and distributed systems. Although it is a topic I am most interested in, I will most likely not be taking a class on the topic, but also want some sort of foundation in the subject. 

In this project series I will be experimenting with various distributed algorithms for doing various things over a network. I will be examining the efficiency of computation which can be divided among multiple processors. Algorithms which I will be looking at include the canonical mergesort, matrix arithmetic, plotting a Delaunay Triangulation, and others.

For testing and comparing the parallelized and single-processor approaches for various aglorithms, I primarily consider one important metric: average computation time. These statistics are gathered during run time by using the high precision timers in the C++ chrono library, which allows the collection of timing data accurate on the order of nanoseconds. Testing itself is performed on three raspberry Pi B+ computers. These machines feature an 700 MHz single-core ARM1176JZF-S processor with 144 KB of Cache and 512 MB of RAM. The complete code for this project can be found on on my [github][5]

Merge sort is a classic sorting algorithm used to introduce the divide and conquer algorithm design paradigm. As such, it is known to parallelize well.

Recursively, mergesort processes an unsorted list of numbers by dividing the unsorted list into n sublists, each containing 1 element (a list of 1 element is considered sorted). It then repeatedly merges sublists to produce new sorted sublists until there is only 1 sublist remaining. This will be the sorted list. The following is a short walkthrough of the server-side code written for a distributed mergesort and the results of the comparison.

An general example is given below with some merge and split steps skipped to conserve space. 
<pre class="prettyprint linenums">
Start       : 3--4--2--1--7--5--8--9--0--6
Split		: 3--4--2--1--7  5--8--9--0--6
Split		: 3  4  2  1  7  5  8  9  0  6
Merge       : 3--4  1--2  5--7  8--9  0--6
Merge       : 1--2--3--4  5--7--8--9  0--6
Merge       : 0--1--2--3--4--5--6--7--8--9
</pre>

I partition my initial unsorted array into n subarrays of equal size on the server. These subarrays will be passed to individual raspberry pi nodes to be sorted before being merged on the server. My breakarray() funtion intuitively takes the initial unsorted array and the number of processors on the network.
<pre class="prettyprint linenums">
def breakarray(array, n): 
	sectionlength = len(array)/n	#length of each section 
	result = [] 
	for i in range(n):
		if i < n - 1:
			result.append( array[ i * sectionlength : (i+1) * sectionlength ] )
		#include all remaining elements for the last section 
		else:
			result.append( array[ i * sectionlength : ] )		
	return result
</pre>

Furthermore, setting up a simple network is quite simple. I make use of Python's socket module to do this. I first provide host and port parameters and create an inet, streaming socket before binding the socket to local host and port.
<pre class="prettyprint linenums">
HOST = ''
PORT = 50007 
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) 
print '[DEBUG] Socket created'
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1) 
try:
	s.bind((HOST, PORT))
except socket.error as msg:
	print '[ERROR] Bind failed. Error Code : ' + str(msg[0]) + ' Message ' + msg[1]
	sys.exit()
</pre>

Finally, I send and recieve the data in 4KB size chunks. Once sorted, the client sends its assigned subaray back to the server to be merged.
<pre class="prettyprint linenums">
for i in range(procno - 1):	# Converts array section into string to be sent
	arraystring = repr(sections[i+1]) 
	conn.sendto(arraystring, addr_list[i])	# Sends array string 
print '[DEBUG] Data sent, sorting array...'
</pre>

<pre class="prettyprint linenums">
# Receives sorted sections from each client
for i in range(procno - 1):
	arraystring = '' 
	print '[DEBUG] Receiving data from clients...' 
	while 1:
		data = conn.recv(4096)	# Receives data in chunks 
		arraystring += data	# Adds data to array string 
		if ']' in data:	# When end of data is received
			break

	print '[DEBUG] Data received, merging arrays...'	
	array = ms.merge(array, eval(arraystring))	# Merges current array with section from client
	print '[DEBUG] Arrays merged.'
</pre>

Tests were preformed on list lengths ranging from 1,000 to 1,000,000, and in all cases, the distributed set up outpreformed the single-node settup by factor seemingly proportional to the number of nodes I distributed the unsorted list accross. Sample output is given below:

<pre class="prettyprint">
$ sudo python server.py 3 10000
[DEBUG] Waiting for client...
[DEBUG] connected to 192.168.1.2:50007
[DEBUG] connected to 192.168.1.3:50007
[DEBUG] Data sent, sorting array...
[DEBUG] Array sorted.
[DEBUG] Receiving data from clients...
[DEBUG] Data Recieved, merging arrays...
[DEBUG] Arrays merged.
[DEBUG] Time taken to sort: 21.223145 seconds.
</pre>

This concludes the first part of this project series. Expect more updates comming soon.

[1]: https://en.wikipedia.org/wiki/Richard_Feynman
[2]: https://en.wikipedia.org/wiki/Danny_Hillis
[3]: https://en.wikipedia.org/wiki/Connection_Machine
[4]: https://en.wikipedia.org/wiki/BBN_Butterfly
[5]: https://github.com/Choltz95/distributedRPI
