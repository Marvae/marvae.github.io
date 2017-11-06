# <center>IEMS5709 Fall 2017 Homework 1</center>

<br>

**Every Student MUST include the following statement, together with his/her signature in the submitted homework.**

<font size=3>*I declare that the assignment submitted on Elearning system is original except for source material explicitly acknowledged,  and that the same or related material has not been previously submitted for another course. I also acknowledge that I am aware of University policy and regulations on honesty in academic work, and of the disciplinary guidelines and procedures applicable to breaches of such policy and regulations, as contained in the website http://www.cuhk.edu.hk/policy/academichonesty/ .*</font>

Signed (Student <u>   MA, Hongwei&emsp;</u> )&emsp;Date:<u>   26/9/2017&emsp;</u>

Name  <u>  MA, Hongwei&emsp;</u>&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;SID:  <u>  1155099617&emsp;</u> 

<font color=red>**Submission notice:**

​        ● Submit your homework via the blackboard system</font>

**General homework policies:**

A student may discuss the problems with others. However, the work a student turns in must be created COMPLETELY by oneself ALONE. A student may not share ANY written work or pictures, nor may one copy answers from any source other   than one’s own brain.

Each student  **MUST LIST**  on the homework paper the **name of every person he/she has discussed or worked with**. If   the answer includes content from any other source, the student **MUST STATE THE SOURCE**. Failure to do so is cheating and   will result in sanctions. Copying answers from someone else is cheating even if one lists their name(s) on the homework.

If there is information you need to solve a problem but the information is not stated in the problem, try to find the data somewhere. If you cannot find it, state what data you need, make a reasonable estimate of its value, and justify any assumptions you make. You will be graded not only on whether your answer is correct, but also on whether you have done   an intelligent analysis.

​     

## Q1:Friendship Recommendation using MapReduce

### 1、Mapreduce Code		

- #### Overview:


For MapReduce problem, since there is already a framework to deal with the dataset, the major job is to define appropriate \<key, value> pair for the map and reduce phase.                              <center>\<User>\<TAB>\<Friends></center>

In this question, it is required to calculate the Jaccard Similarity for each given user i and recommended user j, and it can be computed as follows[1] :

```python
def jaccard(ij_mutual_friends, i_total_friends, j_total_friends):
    '''
    The Jaccard Similarity between the user_pair(i,j)
        |Intersection(i, j)| / |Union(i, j)|
    '''
    union = i_total_friends + j_total_friends - ij_mutual_friends
    return (ij_mutual_friends / (float(union)))
```

So the key idea is to computing the mutual friends of given user i and recommonded friend j. There would be two ways for this problem. 

One direct idea is using the whole dataset to minus the friends of a given user and get user list who is not there friends, and then compute the mutual friends. There are N users in total and the number of every user's friends n is likely to much less than N. Thus if using the input dataset directly to compute the number of users who are not friends of a given user j, it means more calculation and increases the complexity[2]. 

And the other one is more tricky. For a given user i, each two friends of i would have mutual friend i, whether they are friends or not. Hence, for every line in the input dataset, I can get n^2 pairs of recommended friends and n pairs of current friends, which are also similar to the required output formula. This MapReduce code is based on this idea.

I use two map reduce phase to solve this problem, the key point are showing below.

#####    map phase 1

1. Read the original input files line by line;
2. Emit \<user,friend> pair for every line;

#####    reduce phase 1

1. Just sum the pairs of each user to calculate how many friends of them.

#####    map phase 2

1. Read the original input files line by line;
2. Emit \<user,friend>\<tab>\<!> for a user's friends. '!' is a flag to mark that they are already friends
3. Emit \<friend1,friend2>\<tab>\<user> , which means friend1 and friend2 have mutual friend user.

#####    reduce phase 2

1. Read file from map line by line and another file from mapreduce1 to get the number of friends[3].
2. Just sum how many mutual friends of each pairs. If the flag is ! , we ignore them because they are already friends.
3. Calculate Jaccard similarity by mutual friends and union, then Output.

- #### code[4]

mapper1.py

```python
#!/usr/bin/env python

'''
input formate: <user><tab><friends>
output formate: <fromUser><tab><toUser>
'''

import sys,os

def mapper():
    for line in sys.stdin:
        line = line.strip().split('\t')
		#print len(line)
        #exclude users who do not have friends,they are also not friends of others
        if len(line) > 1:
            fromUser, toUsers = line
            toUsers = toUsers.split(',')
            for toUser in toUsers:
                print '%s\t%s' % (fromUser,toUser)
       

if __name__ == "__main__":
    mapper()
```

reducer1.py

```python
#!/usr/bin/env python

'''
input formate: <fromUser><tab><toUser>
output formate: <user><tab><count>
'''

import sys,os

def reducer():
    last_user, count = None, 0
    for line in sys.stdin:
        user, flag = line.strip().split('\t')
        if last_user != user and last_user is not None:
            print '%s\t%s' % (last_user,count)
            last_user, count = None, 0
        last_user = user
        count += 1
    print '%s\t%s' % (last_user,count)

if __name__ == "__main__":
    reducer()
```

mapper.py

```python
#!/usr/bin/env python

'''
input formate: <user><tab><friends>
output formate: <fromUser,toUser><tab><!>
			 or <toUser,toUser><tab><mutual_friend>
'''

import sys,os

def mapper():
    for line in sys.stdin:
        line = line.strip().split('\t')
        # exclude users who do not have friends
        if len(line) > 1:
            fromUser, toUsers = line
            toUsers = toUsers.split(',')
            for toUser in toUsers:
                print '%s,%s\t!' % (fromUser,toUser)
            if len(toUsers)>1:
                for i in range(len(toUsers)):
                    for j in range(i+1,len(toUsers)):
                        print '%s,%s\t%s' % (toUsers[i],toUsers[j],fromUser)
                        print '%s,%s\t%s' % (toUsers[j],toUsers[i],fromUser)
       

if __name__ == "__main__":
    mapper()
```

reducer.py [5]

```python
#!/usr/bin/env python

'''
input formate: <fromUser,toUser><tab><!>
			or <toUser,toUser><tab><mutual_friend>
		   and a dict {userid:count}
output formate: <i,j><Tab><Jaccard_similarity>
'''

import sys,os
import csv

def Jaccard(ij_mutual_friends, i_total_friends, j_total_friends):
    ''' 
    The Jaccard Similarity between the user_pair(i,j)   
    |Intersection(i, j)| / |Union(i, j)|
    '''
    union = i_total_friends + j_total_friends - ij_mutual_friends
    return (ij_mutual_friends / (float(union)))

def reducer(friend_count_file):
    '''
    Read in friend_count_file from cache and for each user,
    calulate their union and mutual 
    '''
    count_reader = csv.reader(open(friend_count_file,'rb'),delimiter='\t')
    friend_count = {}
    for userid,user_count in count_reader :
        friend_count[userid] = user_count
    last_from_to, count = None, 0
    current_friends = None # pairs who are already friends
    for line in sys.stdin:
        #'%s,%s\t%s\n'
        from_to, flag = line.strip().split('\t')
        from_user, to_user = from_to.split(',')
        if flag == '!': 
            current_friends = from_to
            continue
        if from_to == current_friends:
            continue
        if last_from_to != from_to and last_from_to is not None:
            from_user, to_user = last_from_to.split(',')
            print '%s\t%.2f' % (last_from_to,Jaccard(count,int(friend_count[from_user]),int(friend_count[to_user])))
            last_from_to, count = None, 0
        last_from_to = from_to
        count += 1
    from_user, to_user = last_from_to.split(',')
    print '%s\t%.2f' % (last_from_to,Jaccard(count,int(friend_count[from_user]),int(friend_count[to_user])))

if __name__ == "__main__":   
    reducer(sys.argv[1])
```

### 2、Frinend Recommendation Results

- #### Preparation 

  ```sh
  myuser@master:~$ hadoop dfs -ls hdfs://master:9000/
  myuser@master:~$ hadoop dfs -put SmallDataset.txt hdfs://master:9000/input
  myuser@master:~$ hadoop dfs -put LargeDataset.txt hdfs://master:9000/input1
  myuser@master:~$ hadoop dfs -ls hdfs://master:9000/
  Found 4 items
  -rw-r--r--   1 myuser supergroup      19609 2017-10-06 16:17 hdfs://master:9000/input
  -rw-r--r--   1 myuser supergroup    4156181 2017-10-06 16:18 hdfs://master:9000/input1
  drwxrwx---   - myuser supergroup          0 2017-09-17 10:53 hdfs://master:9000/tmp
  drwxr-xr-x   - myuser supergroup          0 2017-09-17 10:56 hdfs://master:9000/user
  ```


- #### Small Dataset

  ##### 1、mapreduce1

  ```sh
  myuser@master:~$ hadoop jar hadoop-2.7.4/share/hadoop/tools/lib/hadoop-streaming-2.7.4.jar -input hdfs://master:9000/input -output hdfs://master:9000/output-s1 -mapper mapper1.py -file mapper1.py -reducer reducer1.py -file reducer1.py
  17/10/06 16:27:05 WARN streaming.StreamJob: -file option is deprecated, please use generic option -files instead.
  17/10/06 16:27:05 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
  packageJobJar: [mapper1.py, reducer1.py, /tmp/hadoop-unjar6225173441586869585/] [] /tmp/streamjob5186779382056902070.jar tmpDir=null
  17/10/06 16:27:06 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 16:27:06 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 16:27:10 INFO mapred.FileInputFormat: Total input paths to process : 1
  17/10/06 16:27:12 INFO mapreduce.JobSubmitter: number of splits:2
  17/10/06 16:27:14 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1507306220197_0001
  17/10/06 16:27:14 INFO impl.YarnClientImpl: Submitted application application_1507306220197_0001
  17/10/06 16:27:14 INFO mapreduce.Job: The url to track the job: http://master:8088/proxy/application_1507306220197_0001/
  17/10/06 16:27:14 INFO mapreduce.Job: Running job: job_1507306220197_0001
  17/10/06 16:27:30 INFO mapreduce.Job: Job job_1507306220197_0001 running in uber mode : false
  17/10/06 16:27:30 INFO mapreduce.Job:  map 0% reduce 0%
  17/10/06 16:27:44 INFO mapreduce.Job:  map 100% reduce 0%
  17/10/06 16:27:51 INFO mapreduce.Job:  map 100% reduce 100%
  17/10/06 16:27:59 INFO mapreduce.Job: Job job_1507306220197_0001 completed successfully
  17/10/06 16:27:59 INFO mapreduce.Job: Counters: 49
  	File System Counters
  		FILE: Number of bytes read=39228
  		FILE: Number of bytes written=450149
  		FILE: Number of read operations=0
  		FILE: Number of large read operations=0
  		FILE: Number of write operations=0
  		HDFS: Number of bytes read=23857
  		HDFS: Number of bytes written=5641
  		HDFS: Number of read operations=9
  		HDFS: Number of large read operations=0
  		HDFS: Number of write operations=2
  	Job Counters 
  		Launched map tasks=2
  		Launched reduce tasks=1
  		Data-local map tasks=2
  		Total time spent by all maps in occupied slots (ms)=17799
  		Total time spent by all reduces in occupied slots (ms)=4591
  		Total time spent by all map tasks (ms)=17799
  		Total time spent by all reduce tasks (ms)=4591
  		Total vcore-milliseconds taken by all map tasks=17799
  		Total vcore-milliseconds taken by all reduce tasks=4591
  		Total megabyte-milliseconds taken by all map tasks=18226176
  		Total megabyte-milliseconds taken by all reduce tasks=4701184
  	Map-Reduce Framework
  		Map input records=1022
  		Map output records=4056
  		Map output bytes=31110
  		Map output materialized bytes=39234
  		Input split bytes=152
  		Combine input records=0
  		Combine output records=0
  		Reduce input groups=946
  		Reduce shuffle bytes=39234
  		Reduce input records=4056
  		Reduce output records=946
  		Spilled Records=8112
  		Shuffled Maps =2
  		Failed Shuffles=0
  		Merged Map outputs=2
  		GC time elapsed (ms)=335
  		CPU time spent (ms)=2300
  		Physical memory (bytes) snapshot=705462272
  		Virtual memory (bytes) snapshot=5710495744
  		Total committed heap usage (bytes)=531628032
  	Shuffle Errors
  		BAD_ID=0
  		CONNECTION=0
  		IO_ERROR=0
  		WRONG_LENGTH=0
  		WRONG_MAP=0
  		WRONG_REDUCE=0
  	File Input Format Counters 
  		Bytes Read=23705
  	File Output Format Counters 
  		Bytes Written=5641
  17/10/06 16:27:59 INFO streaming.StreamJob: Output directory: hdfs://master:9000/output-s1
  myuser@master:~$ hadoop dfs -ls hdfs://master:9000/output-s1
  Found 2 items
  -rw-r--r--   1 myuser supergroup          0 2017-10-06 16:27 hdfs://master:9000/output-s1/_SUCCESS
  -rw-r--r--   1 myuser supergroup       5641 2017-10-06 16:27 hdfs://master:9000/output-s1/part-00000
  ```

  ##### 2、mapreduce2 [6]

  ```Sh
  myuser@master:~$ hadoop jar hadoop-2.7.4/share/hadoop/tools/lib/hadoop-streaming-2.7.4.jar -input hdfs://master:9000/input -output hdfs://master:9000/output-s2 -mapper mapper.py -file mapper.py -reducer 'reducer.py part-00000' -file reducer.py -cacheFile hdfs://master:9000/output-s1/part-00000#part-00000 
  17/10/06 16:37:15 WARN streaming.StreamJob: -file option is deprecated, please use generic option -files instead.
  17/10/06 16:37:15 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
  17/10/06 16:37:16 WARN streaming.StreamJob: -cacheFile option is deprecated, please use -files instead.
  packageJobJar: [mapper.py, reducer.py, /tmp/hadoop-unjar2075644175812971905/] [] /tmp/streamjob1998948041829293104.jar tmpDir=null
  17/10/06 16:37:16 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 16:37:16 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 16:37:20 INFO mapred.FileInputFormat: Total input paths to process : 1
  17/10/06 16:37:21 INFO mapreduce.JobSubmitter: number of splits:2
  17/10/06 16:37:23 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1507306220197_0003
  17/10/06 16:37:23 INFO impl.YarnClientImpl: Submitted application application_1507306220197_0003
  17/10/06 16:37:23 INFO mapreduce.Job: The url to track the job: http://master:8088/proxy/application_1507306220197_0003/
  17/10/06 16:37:23 INFO mapreduce.Job: Running job: job_1507306220197_0003
  17/10/06 16:37:37 INFO mapreduce.Job: Job job_1507306220197_0003 running in uber mode : false
  17/10/06 16:37:37 INFO mapreduce.Job:  map 0% reduce 0%
  17/10/06 16:37:47 INFO mapreduce.Job:  map 100% reduce 0%
  17/10/06 16:37:53 INFO mapreduce.Job:  map 100% reduce 100%
  17/10/06 16:38:02 INFO mapreduce.Job: Job job_1507306220197_0003 completed successfully
  17/10/06 16:38:02 INFO mapreduce.Job: Counters: 49
  	File System Counters
  		FILE: Number of bytes read=991150
  		FILE: Number of bytes written=2354644
  		FILE: Number of read operations=0
  		FILE: Number of large read operations=0
  		FILE: Number of write operations=0
  		HDFS: Number of bytes read=23857
  		HDFS: Number of bytes written=706364
  		HDFS: Number of read operations=9
  		HDFS: Number of large read operations=0
  		HDFS: Number of write operations=2
  	Job Counters 
  		Launched map tasks=2
  		Launched reduce tasks=1
  		Data-local map tasks=2
  		Total time spent by all maps in occupied slots (ms)=13882
  		Total time spent by all reduces in occupied slots (ms)=4270
  		Total time spent by all map tasks (ms)=13882
  		Total time spent by all reduce tasks (ms)=4270
  		Total vcore-milliseconds taken by all map tasks=13882
  		Total vcore-milliseconds taken by all reduce tasks=4270
  		Total megabyte-milliseconds taken by all map tasks=14215168
  		Total megabyte-milliseconds taken by all reduce tasks=4372480
  	Map-Reduce Framework
  		Map input records=1022
  		Map output records=75266
  		Map output bytes=840612
  		Map output materialized bytes=991156
  		Input split bytes=152
  		Combine input records=0
  		Combine output records=0
  		Reduce input groups=57914
  		Reduce shuffle bytes=991156
  		Reduce input records=75266
  		Reduce output records=55949
  		Spilled Records=150532
  		Shuffled Maps =2
  		Failed Shuffles=0
  		Merged Map outputs=2
  		GC time elapsed (ms)=365
  		CPU time spent (ms)=3940
  		Physical memory (bytes) snapshot=710316032
  		Virtual memory (bytes) snapshot=5708587008
  		Total committed heap usage (bytes)=519569408
  	Shuffle Errors
  		BAD_ID=0
  		CONNECTION=0
  		IO_ERROR=0
  		WRONG_LENGTH=0
  		WRONG_MAP=0
  		WRONG_REDUCE=0
  	File Input Format Counters 
  		Bytes Read=23705
  	File Output Format Counters 
  		Bytes Written=706364
  17/10/06 16:38:02 INFO streaming.StreamJob: Output directory: hdfs://master:9000/output-s2

  myuser@master:~$ hadoop dfs -ls hdfs://master:9000/output-s2
  Found 2 items
  -rw-r--r--   1 myuser supergroup          0 2017-10-06 16:37 hdfs://master:9000/output-s2/_SUCCESS
  -rw-r--r--   1 myuser supergroup     706364 2017-10-06 16:37 hdfs://master:9000/output-s2/part-00000
  myuser@master:~$ hadoop dfs -get hdfs://master:9000/output-s2/part-00000 SmallOutput.txt
  myuser@master:~$ ls
  hadoop                      mapper1.py   reducer.py        test.txt
  hadoop-2.7.4                mapper.py    SmallDataset.txt
  hadoop-2.7.4.master.tar.gz  reducer1.py  SmallOutput.txt
  LargeDataset.txt            reducer2.py  testOut.txt
  ```

  ##### 3、results

  - top10.sh [9]

  ```sh
  #!/bin/bash  
  grep "^17," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^117," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^217," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^317," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^417," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^517," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^617," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^717," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^817," SmallOutput.txt | sort -nrk 2 | head -n 10
  grep "^917," SmallOutput.txt | sort -nrk 2 | head -n 10
  ```

  - results of SmallDataset.txt

  ```Sh
  myuser@master:~$ chmod +x top10.sh 
  myuser@master:~$ ./top10.sh 
  17,24	0.25
  17,50	0.22
  17,138	0.22
  17,103	0.22
  17,67	0.20
  17,168	0.20
  17,167	0.20
  17,151	0.20
  17,160	0.18
  17,149	0.18
  117,135	0.43
  117,164	0.22
  117,194	0.20
  117,190	0.20
  117,188	0.20
  117,183	0.20
  117,180	0.20
  117,177	0.20
  117,176	0.20
  117,175	0.20
  217,111	0.33
  317,479	0.12
  317,475	0.12
  317,476	0.11
  317,323	0.10
  317,318	0.09
  317,576	0.04
  417,434	0.88
  417,436	0.75
  417,425	0.56
  417,423	0.50
  417,427	0.44
  417,420	0.43
  417,426	0.42
  417,432	0.40
  417,424	0.38
  417,429	0.33
  517,512	1.00
  517,511	1.00
  517,489	1.00
  517,509	0.67
  517,504	0.67
  517,497	0.67
  517,492	0.67
  517,491	0.67
  517,488	0.67
  517,521	0.50
  617,610	0.50
  617,607	0.50
  617,599	0.50
  617,596	0.50
  617,593	0.50
  617,585	0.50
  617,584	0.50
  617,582	0.50
  617,564	0.50
  617,561	0.50
  717,750	1.00
  717,743	1.00
  717,741	1.00
  717,738	1.00
  717,733	1.00
  717,723	1.00
  717,722	1.00
  717,718	1.00
  717,716	1.00
  717,712	1.00
  917,923	0.67
  917,921	0.50
  917,914	0.50
  917,912	0.50
  917,913	0.40
  917,922	0.33
  917,915	0.33
  917,920	0.25
  917,916	0.25
  917,919	0.20
  ```

- #### Large Dataset

  ##### 1、mapreduce1

  ```Sh
  hadoop jar hadoop-2.7.4/share/hadoop/tools/lib/hadoop-streaming-2.7.4.jar -input hdfs://master:9000/input1 -output hdfs://master:9000/output-l1 -mapper mapper1.py -file mapper1.py -reducer reducer1.py -file reducer1.py
  17/10/06 17:27:53 WARN streaming.StreamJob: -file option is deprecated, please use generic option -files instead.
  17/10/06 17:27:53 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
  packageJobJar: [mapper1.py, reducer1.py, /tmp/hadoop-unjar5423882165772706716/] [] /tmp/streamjob2106858613083692784.jar tmpDir=null
  17/10/06 17:27:53 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 17:27:54 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 17:27:57 INFO mapred.FileInputFormat: Total input paths to process : 1
  17/10/06 17:27:59 INFO mapreduce.JobSubmitter: number of splits:2
  17/10/06 17:28:01 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1507306220197_0004
  17/10/06 17:28:01 INFO impl.YarnClientImpl: Submitted application application_1507306220197_0004
  17/10/06 17:28:01 INFO mapreduce.Job: The url to track the job: http://master:8088/proxy/application_1507306220197_0004/
  17/10/06 17:28:01 INFO mapreduce.Job: Running job: job_1507306220197_0004
  17/10/06 17:28:14 INFO mapreduce.Job: Job job_1507306220197_0004 running in uber mode : false
  17/10/06 17:28:14 INFO mapreduce.Job:  map 0% reduce 0%
  17/10/06 17:28:23 INFO mapreduce.Job:  map 100% reduce 0%
  17/10/06 17:28:32 INFO mapreduce.Job:  map 100% reduce 100%
  17/10/06 17:28:42 INFO mapreduce.Job: Job job_1507306220197_0004 completed successfully
  17/10/06 17:28:43 INFO mapreduce.Job: Counters: 50
  	File System Counters
  		FILE: Number of bytes read=8956108
  		FILE: Number of bytes written=18283912
  		FILE: Number of read operations=0
  		FILE: Number of large read operations=0
  		FILE: Number of write operations=0
  		HDFS: Number of bytes read=4160431
  		HDFS: Number of bytes written=403227
  		HDFS: Number of read operations=9
  		HDFS: Number of large read operations=0
  		HDFS: Number of write operations=2
  	Job Counters 
  		Launched map tasks=2
  		Launched reduce tasks=1
  		Data-local map tasks=1
  		Rack-local map tasks=1
  		Total time spent by all maps in occupied slots (ms)=11281
  		Total time spent by all reduces in occupied slots (ms)=6036
  		Total time spent by all map tasks (ms)=11281
  		Total time spent by all reduce tasks (ms)=6036
  		Total vcore-milliseconds taken by all map tasks=11281
  		Total vcore-milliseconds taken by all reduce tasks=6036
  		Total megabyte-milliseconds taken by all map tasks=11551744
  		Total megabyte-milliseconds taken by all reduce tasks=6180864
  	Map-Reduce Framework
  		Map input records=49995
  		Map output records=661596
  		Map output bytes=7632910
  		Map output materialized bytes=8956114
  		Input split bytes=154
  		Combine input records=0
  		Combine output records=0
  		Reduce input groups=49124
  		Reduce shuffle bytes=8956114
  		Reduce input records=661596
  		Reduce output records=49124
  		Spilled Records=1323192
  		Shuffled Maps =2
  		Failed Shuffles=0
  		Merged Map outputs=2
  		GC time elapsed (ms)=217
  		CPU time spent (ms)=7140
  		Physical memory (bytes) snapshot=715411456
  		Virtual memory (bytes) snapshot=5716193280
  		Total committed heap usage (bytes)=524812288
  	Shuffle Errors
  		BAD_ID=0
  		CONNECTION=0
  		IO_ERROR=0
  		WRONG_LENGTH=0
  		WRONG_MAP=0
  		WRONG_REDUCE=0
  	File Input Format Counters 
  		Bytes Read=4160277
  	File Output Format Counters 
  		Bytes Written=403227
  17/10/06 17:28:43 INFO streaming.StreamJob: Output directory: hdfs://master:9000/output-l1
  myuser@master:~$ hadoop dfs -ls hdfs://master:9000/output-l1
  Found 2 items
  -rw-r--r--   1 myuser supergroup          0 2017-10-06 17:28 hdfs://master:9000/output-l1/_SUCCESS
  -rw-r--r--   1 myuser supergroup     403227 2017-10-06 17:28 hdfs://master:9000/output-l1/part-00000
  ```

  ##### 2、mapreduce2

  ```Sh
  myuser@master:~$ hadoop jar hadoop-2.7.4/share/hadoop/tools/lib/hadoop-streaming-2.7.4.jar -input hdfs://master:9000/input1 -output hdfs://master:9000/output-l2 -mapper mapper.py -file mapper.py -reducer 'reducer.py part-00000' -file reducer.py -cacheFile hdfs://master:9000/output-l1/part-00000#part-00000 
  17/10/06 17:34:21 WARN streaming.StreamJob: -file option is deprecated, please use generic option -files instead.
  17/10/06 17:34:21 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
  17/10/06 17:34:21 WARN streaming.StreamJob: -cacheFile option is deprecated, please use -files instead.
  packageJobJar: [mapper.py, reducer.py, /tmp/hadoop-unjar4690264457949443989/] [] /tmp/streamjob8085182957287102514.jar tmpDir=null
  17/10/06 17:34:22 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 17:34:22 INFO client.RMProxy: Connecting to ResourceManager at master/10.150.0.2:8032
  17/10/06 17:34:25 INFO mapred.FileInputFormat: Total input paths to process : 1
  17/10/06 17:34:27 INFO mapreduce.JobSubmitter: number of splits:2
  17/10/06 17:34:29 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1507306220197_0005
  17/10/06 17:34:29 INFO impl.YarnClientImpl: Submitted application application_1507306220197_0005
  17/10/06 17:34:29 INFO mapreduce.Job: The url to track the job: http://master:8088/proxy/application_1507306220197_0005/
  17/10/06 17:34:29 INFO mapreduce.Job: Running job: job_1507306220197_0005
  17/10/06 17:34:42 INFO mapreduce.Job: Job job_1507306220197_0005 running in uber mode : false
  17/10/06 17:34:42 INFO mapreduce.Job:  map 0% reduce 0%
  17/10/06 17:34:57 INFO mapreduce.Job:  map 21% reduce 0%
  17/10/06 17:35:06 INFO mapreduce.Job:  map 28% reduce 0%
  17/10/06 17:35:09 INFO mapreduce.Job:  map 36% reduce 0%
  17/10/06 17:35:17 INFO mapreduce.Job:  map 47% reduce 0%
  17/10/06 17:35:20 INFO mapreduce.Job:  map 49% reduce 0%
  17/10/06 17:35:26 INFO mapreduce.Job:  map 52% reduce 0%
  17/10/06 17:35:29 INFO mapreduce.Job:  map 57% reduce 0%
  17/10/06 17:35:32 INFO mapreduce.Job:  map 63% reduce 0%
  17/10/06 17:35:35 INFO mapreduce.Job:  map 68% reduce 0%
  17/10/06 17:35:38 INFO mapreduce.Job:  map 75% reduce 0%
  17/10/06 17:35:40 INFO mapreduce.Job:  map 78% reduce 0%
  17/10/06 17:35:41 INFO mapreduce.Job:  map 83% reduce 0%
  17/10/06 17:35:47 INFO mapreduce.Job:  map 85% reduce 0%
  17/10/06 17:35:50 INFO mapreduce.Job:  map 91% reduce 0%
  17/10/06 17:35:53 INFO mapreduce.Job:  map 98% reduce 17%
  17/10/06 17:35:55 INFO mapreduce.Job:  map 100% reduce 17%
  17/10/06 17:36:00 INFO mapreduce.Job:  map 100% reduce 67%
  17/10/06 17:36:03 INFO mapreduce.Job:  map 100% reduce 69%
  17/10/06 17:36:06 INFO mapreduce.Job:  map 100% reduce 71%
  17/10/06 17:36:09 INFO mapreduce.Job:  map 100% reduce 72%
  17/10/06 17:36:12 INFO mapreduce.Job:  map 100% reduce 74%
  17/10/06 17:36:13 INFO mapreduce.Job:  map 100% reduce 76%
  17/10/06 17:36:16 INFO mapreduce.Job:  map 100% reduce 78%
  17/10/06 17:36:19 INFO mapreduce.Job:  map 100% reduce 80%
  17/10/06 17:36:23 INFO mapreduce.Job:  map 100% reduce 81%
  17/10/06 17:36:26 INFO mapreduce.Job:  map 100% reduce 84%
  17/10/06 17:36:29 INFO mapreduce.Job:  map 100% reduce 86%
  17/10/06 17:36:32 INFO mapreduce.Job:  map 100% reduce 88%
  17/10/06 17:36:35 INFO mapreduce.Job:  map 100% reduce 90%
  17/10/06 17:36:38 INFO mapreduce.Job:  map 100% reduce 92%
  17/10/06 17:36:41 INFO mapreduce.Job:  map 100% reduce 93%
  17/10/06 17:36:44 INFO mapreduce.Job:  map 100% reduce 95%
  17/10/06 17:36:47 INFO mapreduce.Job:  map 100% reduce 97%
  17/10/06 17:36:50 INFO mapreduce.Job:  map 100% reduce 98%
  17/10/06 17:36:53 INFO mapreduce.Job:  map 100% reduce 100%
  17/10/06 17:37:05 INFO mapreduce.Job: Job job_1507306220197_0005 completed successfully
  17/10/06 17:37:05 INFO mapreduce.Job: Counters: 49
  	File System Counters
  		FILE: Number of bytes read=907669724
  		FILE: Number of bytes written=1361876909
  		FILE: Number of read operations=0
  		FILE: Number of large read operations=0
  		FILE: Number of write operations=0
  		HDFS: Number of bytes read=4160431
  		HDFS: Number of bytes written=204811937
  		HDFS: Number of read operations=9
  		HDFS: Number of large read operations=0
  		HDFS: Number of write operations=2
  	Job Counters 
  		Launched map tasks=2
  		Launched reduce tasks=1
  		Data-local map tasks=2
  		Total time spent by all maps in occupied slots (ms)=124956
  		Total time spent by all reduces in occupied slots (ms)=72069
  		Total time spent by all map tasks (ms)=124956
  		Total time spent by all reduce tasks (ms)=72069
  		Total vcore-milliseconds taken by all map tasks=124956
  		Total vcore-milliseconds taken by all reduce tasks=72069
  		Total megabyte-milliseconds taken by all map tasks=127954944
  		Total megabyte-milliseconds taken by all reduce tasks=73798656
  	Map-Reduce Framework
  		Map input records=49995
  		Map output records=23570682
  		Map output bytes=406693462
  		Map output materialized bytes=453834838
  		Input split bytes=154
  		Combine input records=0
  		Combine output records=0
  		Reduce input groups=12715792
  		Reduce shuffle bytes=453834838
  		Reduce input records=23570682
  		Reduce output records=12408813
  		Spilled Records=70712046
  		Shuffled Maps =2
  		Failed Shuffles=0
  		Merged Map outputs=2
  		GC time elapsed (ms)=700
  		CPU time spent (ms)=204060
  		Physical memory (bytes) snapshot=729124864
  		Virtual memory (bytes) snapshot=5737504768
  		Total committed heap usage (bytes)=492830720
  	Shuffle Errors
  		BAD_ID=0
  		CONNECTION=0
  		IO_ERROR=0
  		WRONG_LENGTH=0
  		WRONG_MAP=0
  		WRONG_REDUCE=0
  	File Input Format Counters 
  		Bytes Read=4160277
  	File Output Format Counters 
  		Bytes Written=204811937
  17/10/06 17:37:05 INFO streaming.StreamJob: Output directory: hdfs://master:9000/output-l2
  myuser@master:~$ hadoop dfs -ls hdfs://master:9000/output-l2
  Found 2 items
  -rw-r--r--   1 myuser supergroup          0 2017-10-06 17:36 hdfs://master:9000/output-l2/_SUCCESS
  -rw-r--r--   1 myuser supergroup  204811937 2017-10-06 17:36 hdfs://master:9000/output-l2/part-00000
  myuser@master:~$ hadoop dfs -get hdfs://master:9000/output-l2/part-00000 LargeOutput.txt
  myuser@master:~$ ls
  hadoop                      LargeOutput.txt  reducer2.py       testOut.txt
  hadoop-2.7.4                mapper1.py       reducer.py        test.txt
  hadoop-2.7.4.master.tar.gz  mapper.py        SmallDataset.txt
  LargeDataset.txt            reducer1.py      SmallOutput.txt
  ```

  ##### 3、results

  - top10_1.sh(simliar to top10.sh)
  - result of LargeDataset.txt

  ```sh
  myuser@master:~$ vim top10_1.sh
  myuser@master:~$ chmod +x top10_1.sh
  myuser@master:~$ ./top10_1.sh 
  617,16283	0.21
  617,44612	0.18
  617,44608	0.17
  617,44607	0.17
  617,44600	0.15
  617,44609	0.14
  617,585		0.11
  617,539		0.11
  617,530		0.10
  617,47798	0.10
  1617,8671	0.06
  1617,23685	0.05
  1617,1404	0.05
  1617,9891	0.04
  1617,8686	0.04
  1617,4389	0.04
  1617,33637	0.04
  1617,33636	0.04
  1617,33635	0.04
  1617,23698	0.04
  2617,44509	0.25
  2617,42414	0.16
  2617,9786	0.14
  2617,2773	0.13
  2617,2610	0.13
  2617,48780	0.12
  2617,164	0.12
  2617,48791	0.11
  2617,44510	0.11
  2617,28234	0.11
  3617,3639	0.48
  3617,3636	0.48
  3617,3640	0.43
  3617,3637	0.38
  3617,3631	0.38
  3617,3616	0.37
  3617,3645	0.36
  3617,3649	0.30
  3617,3630	0.28
  3617,3628	0.28
  4617,4634	0.30
  4617,42787	0.28
  4617,4640	0.27
  4617,4626	0.19
  4617,42785	0.19
  4617,16820	0.17
  4617,4629	0.16
  4617,4623	0.15
  4617,4621	0.15
  4617,4636	0.11
  5617,5634	0.62
  5617,5622	0.42
  5617,5609	0.42
  5617,5618	0.39
  5617,5616	0.39
  5617,5605	0.39
  5617,5599	0.38
  5617,5597	0.38
  5617,5635	0.35
  5617,5619	0.33
  6617,6560	0.56
  6617,6566	0.50
  6617,6604	0.46
  6617,6619	0.45
  6617,6586	0.33
  6617,6647	0.32
  6617,6503	0.31
  6617,40234	0.21
  6617,6572	0.15
  6617,6573	0.11
  7617,25812	0.10
  7617,8134	0.09
  7617,7478	0.09
  7617,22720	0.09
  7617,7644	0.08
  7617,49062	0.08
  7617,29922	0.08
  7617,29418	0.08
  7617,28630	0.08
  7617,25817	0.08
  8617,17705	0.14
  8617,13533	0.13
  8617,21614	0.12
  8617,8627	0.11
  8617,8622	0.11
  8617,8625	0.10
  8617,8616	0.10
  8617,8600	0.10
  8617,8585	0.10
  8617,3188	0.10
  9617,9655	1.00
  9617,9631	1.00
  9617,9627	1.00
  9617,9622	1.00
  9617,9647	0.50
  9617,9646	0.50
  9617,9623	0.50
  9617,9619	0.50
  9617,9611	0.50
  9617,9614	0.33
  10617,10618	0.60
  10617,10650	0.25
  10617,10649	0.25
  10617,10646	0.25
  10617,10637	0.25
  10617,10635	0.25
  10617,10647	0.20
  10617,10641	0.20
  10617,10639	0.20
  10617,10609	0.20
  11617,22676	0.21
  11617,22657	0.21
  11617,22673	0.18
  11617,22605	0.15
  11617,22693	0.14
  11617,22636	0.14
  11617,33741	0.13
  11617,22669	0.12
  11617,22614	0.12
  11617,22688	0.11
  12617,12606	0.60
  12617,12615	0.33
  12617,12609	0.33
  12617,12604	0.33
  12617,12575	0.33
  12617,12564	0.33
  12617,12610	0.25
  12617,12608	0.25
  12617,12596	0.25
  12617,12595	0.25
  13617,13612	0.50
  13617,13611	0.50
  13617,13610	0.50
  13617,13609	0.50
  13617,13608	0.50
  13617,13605	0.50
  13617,13603	0.50
  13617,13598	0.50
  13617,13597	0.50
  13617,13589	0.50
  14617,14605	0.69
  14617,14620	0.67
  14617,14597	0.58
  14617,14613	0.36
  14617,14604	0.36
  14617,14602	0.31
  14617,10744	0.29
  14617,14594	0.27
  14617,14599	0.21
  14617,20729	0.18
  15617,15616	1.00
  15617,15611	0.50
  15617,15613	0.33
  15617,15615	0.17
  15617,15614	0.12
  15617,15612	0.11
  16617,16618	1.00
  16617,16613	1.00
  16617,16611	1.00
  16617,16610	1.00
  16617,16616	0.50
  16617,16615	0.50
  16617,16607	0.25
  16617,16612	0.17
  16617,16609	0.17
  16617,16608	0.14
  17617,4284	0.09
  17617,42194	0.09
  17617,17613	0.09
  17617,7760	0.08
  17617,7618	0.08
  17617,6661	0.08
  17617,43214	0.08
  17617,42191	0.08
  17617,42190	0.08
  17617,32152	0.08
  18617,18641	0.50
  18617,18635	0.50
  18617,18633	0.50
  18617,18632	0.50
  18617,18607	0.50
  18617,18579	0.50
  18617,42382	0.33
  18617,18622	0.33
  18617,18600	0.33
  18617,9440	0.25
  19617,19694	0.29
  19617,19610	0.27
  19617,19596	0.27
  19617,19712	0.25
  19617,19656	0.21
  19617,19706	0.20
  19617,19645	0.20
  19617,19637	0.20
  19617,19604	0.19
  19617,19606	0.18
  21617,21590	0.75
  21617,21616	0.25
  21617,21611	0.25
  21617,21609	0.25
  21617,21605	0.25
  21617,21592	0.25
  21617,21589	0.25
  21617,21581	0.25
  21617,21593	0.22
  21617,21598	0.21
  22617,13305	0.16
  22617,26100	0.15
  22617,22646	0.15
  22617,18136	0.15
  22617,24178	0.12
  22617,13309	0.12
  22617,13277	0.11
  22617,6180	0.10
  22617,42905	0.10
  22617,18120	0.10
  23617,14078	0.15
  23617,14044	0.15
  23617,13983	0.15
  23617,14207	0.14
  23617,14150	0.14
  23617,14095	0.14
  23617,14015	0.14
  23617,13966	0.14
  23617,13867	0.14
  23617,34356	0.13
  24617,24877	0.50
  24617,24827	0.33
  24617,5043	0.25
  24617,4984	0.25
  24617,26327	0.25
  24617,24828	0.25
  24617,2151	0.25
  24617,21340	0.25
  24617,43922	0.20
  24617,34960	0.20
  25617,25563	0.41
  25617,25581	0.26
  25617,25605	0.22
  25617,25609	0.18
  25617,25608	0.18
  25617,25557	0.18
  25617,25547	0.18
  25617,25624	0.16
  25617,25614	0.14
  25617,25596	0.14
  26617,26646	0.36
  26617,26746	0.34
  26617,26747	0.31
  26617,26745	0.31
  26617,26637	0.28
  26617,26760	0.24
  26617,26750	0.24
  26617,26641	0.24
  26617,15999	0.24
  26617,13323	0.24
  27617,27585	0.29
  27617,27626	0.26
  27617,27583	0.22
  27617,27570	0.22
  27617,27661	0.21
  27617,27620	0.20
  27617,27638	0.19
  27617,27588	0.19
  27617,27650	0.16
  27617,27586	0.16
  28617,28623	0.12
  28617,28618	0.12
  28617,28615	0.12
  28617,30451	0.11
  28617,28627	0.11
  28617,28626	0.11
  28617,28624	0.11
  28617,28616	0.11
  28617,48473	0.10
  28617,28622	0.10
  29617,29640	0.50
  29617,29639	0.50
  29617,29635	0.50
  29617,29631	0.50
  29617,29625	0.50
  29617,29616	0.50
  29617,29595	0.50
  29617,29632	0.33
  29617,29628	0.33
  29617,29627	0.33
  30617,45333	0.20
  30617,33633	0.20
  30617,30685	0.20
  30617,30684	0.20
  30617,30683	0.20
  30617,30679	0.20
  30617,30675	0.20
  30617,30674	0.20
  30617,30671	0.20
  30617,30665	0.20
  31617,31613	0.55
  31617,31607	0.44
  31617,31609	0.41
  31617,31596	0.40
  31617,31556	0.39
  31617,31511	0.38
  31617,31492	0.38
  31617,31606	0.37
  31617,31560	0.37
  31617,31493	0.37
  32617,32625	1.00
  32617,32614	1.00
  32617,32596	1.00
  32617,32624	0.50
  32617,32622	0.50
  32617,32610	0.50
  32617,32609	0.50
  32617,32606	0.50
  32617,32602	0.50
  32617,32594	0.50
  33617,33601	0.50
  33617,33622	0.40
  33617,33572	0.25
  33617,22923	0.25
  33617,23056	0.22
  33617,33620	0.20
  33617,33615	0.20
  33617,33614	0.20
  33617,33611	0.20
  33617,33609	0.20
  34617,34615	0.17
  34617,24909	0.17
  34617,34556	0.15
  34617,34563	0.13
  34617,34569	0.12
  34617,34565	0.12
  34617,34437	0.12
  34617,34678	0.11
  34617,34587	0.11
  34617,34627	0.10
  35617,21841	0.07
  35617,49898	0.06
  35617,38735	0.06
  35617,16864	0.06
  35617,7522	0.05
  35617,42051	0.05
  35617,41412	0.05
  35617,35692	0.05
  35617,35633	0.05
  35617,29546	0.05
  36617,4236	0.43
  36617,36607	0.33
  36617,36604	0.33
  36617,36597	0.33
  36617,36569	0.33
  36617,36566	0.33
  36617,36551	0.33
  36617,36626	0.25
  36617,36603	0.25
  36617,36585	0.25
  37617,37131	0.15
  37617,37122	0.15
  37617,21283	0.12
  37617,37083	0.11
  37617,37169	0.09
  37617,36853	0.09
  37617,44136	0.08
  37617,39330	0.08
  37617,37790	0.08
  37617,37639	0.08
  38617,38619	0.78
  38617,38621	0.60
  38617,38622	0.50
  38617,38614	0.38
  38617,38616	0.31
  38617,38612	0.28
  38617,38785	0.17
  38617,38624	0.12
  38617,38615	0.12
  38617,38623	0.11
  39617,39618	0.67
  39617,39599	0.50
  39617,39610	0.33
  39617,39607	0.33
  39617,39603	0.33
  39617,39615	0.29
  39617,39611	0.29
  39617,39616	0.25
  39617,39614	0.25
  39617,39613	0.25
  40617,40619	0.50
  40617,40618	0.50
  40617,40620	0.33
  40617,40616	0.17
  40617,40614	0.08
  40617,40615	0.07
  40617,38004	0.07
  40617,40612	0.05
  41617,39245	0.25
  41617,41232	0.17
  41617,13860	0.17
  41617,41606	0.14
  41617,18928	0.14
  41617,41365	0.12
  41617,38485	0.11
  41617,41370	0.10
  41617,38483	0.10
  41617,23295	0.09
  42617,22128	0.17
  42617,42620	0.12
  42617,42618	0.12
  42617,42616	0.12
  42617,42612	0.12
  42617,4172	0.11
  42617,20754	0.10
  42617,7553	0.09
  42617,44640	0.09
  42617,44625	0.09
  43617,43636	1.00
  43617,9641	0.25
  43617,43332	0.25
  43617,43616	0.20
  43617,43537	0.20
  43617,22221	0.20
  43617,12588	0.20
  43617,43583	0.17
  43617,243	0.17
  43617,13765	0.17
  44617,44609	0.50
  44617,44598	0.50
  44617,44599	0.42
  44617,28780	0.33
  44617,44611	0.29
  44617,44612	0.27
  44617,44600	0.27
  44617,44616	0.21
  44617,44603	0.21
  44617,44597	0.19
  45617,3554	0.11
  45617,22045	0.11
  45617,12234	0.11
  45617,6404	0.10
  45617,3451	0.10
  45617,20843	0.10
  45617,19835	0.10
  45617,46804	0.09
  45617,3545	0.09
  45617,25985	0.09
  46617,46620	0.50
  46617,46614	0.50
  46617,46613	0.50
  46617,46611	0.50
  46617,46596	0.50
  46617,46619	0.33
  46617,46602	0.33
  46617,46612	0.25
  46617,46610	0.25
  46617,46598	0.25
  47617,47623	0.09
  47617,12227	0.07
  47617,27881	0.06
  47617,21270	0.06
  47617,12775	0.06
  47617,12319	0.06
  47617,9972	0.05
  47617,9848	0.05
  47617,9781	0.05
  47617,9692	0.05
  48617,48665	0.31
  48617,48605	0.31
  48617,48652	0.20
  48617,48664	0.18
  48617,48648	0.18
  48617,48604	0.17
  48617,25271	0.15
  48617,48646	0.14
  48617,48643	0.14
  48617,48653	0.12
  49617,6966	0.13
  49617,23759	0.13
  49617,23762	0.12
  49617,23719	0.12
  49617,23717	0.12
  49617,6977	0.09
  49617,6962	0.09
  49617,34809	0.09
  49617,23814	0.09
  49617,23792	0.09
  ```

- #### Performance Comparsion Report

  | mapper tasks /reducer tasks            | Maximum mapper time | Minimum mapper time | Average mapper time | Maximum reducer time | Minimum reducer time | Average reducer time | Total job time |
  | -------------------------------------- | ------------------- | ------------------- | ------------------- | -------------------- | -------------------- | -------------------- | -------------- |
  | 2 / 1                                  | 1mins, 9sec         | 55sec               | 1mins, 2sec         | 58sec                | 58sec                | 58sec                | 2mins, 16sec   |
  | 4 / 2                                  | 1mins, 18sec        | 59sec               | 1mins, 8sec         | 28sec                | 28sec                | 28sec                | 1mins, 55sec   |
  | 10 / 5                  (3 map killed) | 1mins, 7sec         | 8sec                | 41sec               | 37sec                | 23sec                | 31sec                | 1mins, 58sec   |
  | 20 / 10               (1 map killed)   | 57sec               | 9sec                | 35sec               | 41sec                | 8sec                 | 30sec                | 1mins, 56sec   |
  | 4 / 4                                  | 1mins, 20sec        | 59sec               | 1mins, 10sec        | 29sec                | 16sec                | 22sec                | 2mins, 0sec    |
  | 6 / 3                                  | 1mins, 30sec        | 1mins, 10sec        | 1mins, 18sec        | 21sec                | 20sec                | 20sec                | 2mins, 0sec    |

  From the table, we can observe some identities:

  1. More map or reduce tasks do not always mans little time
  2. The proportion of map and reduce tasks is important
  3. For more statistics, it can be observed that more map and reduce tasks means more shuffle time and merge time

  | mapper tasks /reducer tasks | 2 / 1 | 4 / 2 | 10 / 5 | 20 / 10 | 4 / 4 | 6 / 3 |
  | --------------------------- | ----- | ----- | ------ | ------- | ----- | ----- |
  | Average shuffe time         | 13sec | 17sec | 55sec  | 32sec   | 18sec | 17sec |
  | Average merge time          | 0sec  | 0sec  | 3sec   | 3sec    | 1sec  | 0sec  |

### 3、Promblems and solve

##### 	1、No friend situation<br>

![nofrineds](/Users/marvae/Desktop/homework1/nofrineds.png)

In the begining, I did not consider that there may be situations when some user do not have any friends. So I got this error. Then I added an IF to judge whether a user have friends or not. If a user do not have a friend, according to our question, it also can not be others friends and have no recommend friends. Thus I can just ignore them.

##### 	2、PipeMapRed.waitOutputThreads(): subprocess failed with code 127<br>![error3-solveby4](/Users/marvae/Desktop/homework1/error3-solveby4.png)

In this problem, it threads error code 127 and when I try to run mapper.py in the shell, it occurred that "No such file or directory"[6]. In the reference 6, it can be solved by editting the formula of file in vim.

```sh
:set ff
:set ff=unix
```

##### 	3、PipeMapRed.waitOutputThreads(): subprocess failed with code 1<br>

![error5](/Users/marvae/Desktop/homework1/error5.png)

I was stuck in this problem for almost 3 days, since I could not find useful things in Google. So I try to check the logs in file stderr, it shows like below[8].<br>

![log](/Users/marvae/Desktop/homework1/log.png)

![log的副本](/Users/marvae/Desktop/homework1/log的副本.png)

It can be seen that the error occurs in reducer.py when trying to print and it is an KeyError. Then I printed all the Key but find no problems. So I searched about the information of error, it seems to be encode problem of the file. I checked the given dataset and found that there is an additional comma in the end of line 1.<br>

![屏幕快照 2017-10-08 上午12.34.33](/Users/marvae/Desktop/homework1/屏幕快照 2017-10-08 上午12.34.33.png)

##### 	4、Output top-10 recomendation

​	I am not very good at python, so I used Shell script to solve this problem, which is a little stupid.But by using sublime I can edit the value conveniently. I also spend a bit of time to learn about shell.[10]

### 4、Reference

[1]Introduction to Recommendations with Map-Reduce and mrjob: http://aimotion.blogspot.hk/2012/08/introduction-to-recommendations-with.html

[2]一个计算好友相似度的MapReduce实现(Python版): https://my.oschina.net/u/781842/blog/128979

[3]Recommendations with hadoop streaming and python: https://www.slideshare.net/100k100k/recommendations-with-hadoop-streaming-and-python

[4]Writing an Hadoop MapReduce Program in Python(Chinese): https://python.freelycode.com/contribution/detail/307

[5]python sys.argv[]用法: http://blog.csdn.net/sxingming/article/details/52074311

[6]Hadoop Streaming: https://hadoop.apache.org/docs/r1.0.4/cn/streaming.html

[7]通过 shell 执行 python 报错: No such file or directory: https://www.liaohuqiu.net/cn/posts/python-script-in-bin-directory/

[8]streaming常见计算任务失败原因: http://blog.csdn.net/wuruiaoxue/article/details/52753076

[9]Shell 教程:http://www.runoob.com/linux/linux-shell.html

[10]head与tail——打印文件的前10行和后10行：http://blog.sina.com.cn/s/blog_671127cf01013fvv.html

[11]Intro to Hadoop and MapReduce by udacity: https://cn.udacity.com/course/intro-to-hadoop-and-mapreduce--ud617