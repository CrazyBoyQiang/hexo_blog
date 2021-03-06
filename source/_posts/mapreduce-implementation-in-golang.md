---
title: golang实现单机版MapReduce
date: 2016-08-27 17:09:00
categories: 分布式
tags:
  - golang
  - MapReduce
---

# 介绍

本篇文章主要描述了如何使用golang实现一个单机版的MapReduce程序，想法来自于MIT-6.824课程的一个lab。本文分为以下几个模块：

- MapReduce基本原理
- MapReduce简单使用
- MapReduce单机版实现

# MapReduce基本原理

MapReduce的计算以一组Key/Value对为输入，然后输出一组Key/Value对，用户通过编写Map和Reduce函数来控制处理逻辑。

Map函数把输入转换成一组中间的Key/Value对，MapReduce library会把所有Key的中间结果传递给Reduce函数处理。

Reduce函数接收Key和其对应的一组Value，它的作用就是聚合这些Value，产生最终的结果。Reduce的输入是以迭代器的方式输入，使得MapReduce可以处理数据量比内存大的情况。

一次MapReduce的处理过程如下图：

![](http://o8m1nd933.bkt.clouddn.com/blog/mapreduce/mapreduce_overview.png)

1. MapReduce library会把输入文件划分成多个16到64MB大小的分片（大小可以通过参数调节），然后在一组机器上启动程序。
2. 其中比较特殊的程序是master，剩下的由master分配任务的程序叫worker。总共有M个map任务和R个reduce任务需要分配，master会选取空闲的worker，然后分配一个map任务或者reduce任务。
3. 处理map任务的worker会从输入分片读入数据，解析出输入数据的K/V对，然后传递给Map函数，生成的K/V中间结果会缓存在内存中。
4. map任务的中间结果会被周期性地写入到磁盘中，以partition函数来分成R个部分。R个部分的磁盘地址会推送到master，然后由它转发给响应的reduce worker。
5. 当reduce worker接收到master发送的地址信息时，它会通过RPC来向map worker读取对应的数据。当reduce worker读取到了所有的数据，它先按照key来排序，方便聚合操作。
6. reduce worker遍历排序好的中间结果，对于相同的key，把其所有数据传入到Reduce函数进行处理，生成最终的结果会被追加到结果文件中。
7. 当所有的map和reduce任务都完成时，master会唤醒用户程序，然后返回到用户程序空间执行用户代码。

成功执行后，输出结果在R个文件中，通常，用户不需要合并这R个文件，因为，可以把它们作为新的MapReduce处理逻辑的输入数据，或者其它分布式应用的输入数据。

更详细的介绍可以参考我之前写的博客[MapReduce原理](http://oserror.com/distributed/mapreduce/)

## MapReduce核心组件

MapReduce核心组件包括Master和Worker，它们的职责分别如下。

### Master

MapReduce负责一次执行过程中Map和Reduce任务的调度，其需要维护的信息包括如下：

- worker的状态
- 任务的状态
- Map生成的文件的位置

### Worker

Worker分为两种，分别是Map和Reduce：

Map Worker的职责：

- 对分片数据调用用户指定的Map函数
- 根据Reduce的个数，把数据分成R份

Reduce Worker的职责：

- 对收集到的数据进行排序
- 对于相同的Key调用Reduce函数进行处理

# MapReduce简单使用

了解MapReduce基本原理后，再来通过一个简单的word count例子，来描述MapReduce的使用方法，代码如下：

```go
// The mapping function is called once for each piece of the input.
// In this framework, the key is the name of the file that is being processed,
// and the value is the file's contents. The return value should be a slice of
// key/value pairs, each represented by a mapreduce.KeyValue.
func mapF(document string, value string) (res []mapreduce.KeyValue) {
  // TODO: you have to write this function
  f := func(r rune) bool {
    return !unicode.IsLetter(r)
  }
  words := strings.FieldsFunc(value, f)
  for _, word := range words {
    kv := mapreduce.KeyValue{word, " "}
    res = append(res, kv) 
  }
  return
}
 
// The reduce function is called once for each key generated by Map, with a
// list of that key's string value (merged across all inputs). The return value
// should be a single output value for that key.
func reduceF(key string, values []string) string {
  // TODO: you also have to write this function
  s := strconv.Itoa(len(values))
  return s
}
 
// Can be run in 3 ways:
// 1) Sequential (e.g., go run wc.go master sequential x1.txt .. xN.txt)
// 2) Master (e.g., go run wc.go master localhost:7777 x1.txt .. xN.txt)
// 3) Worker (e.g., go run wc.go worker localhost:7777 localhost:7778 &)
func main() {
  if len(os.Args) < 4 { 
    fmt.Printf("%s: see usage comments in file\n", os.Args[0])
  } else if os.Args[1] == "master" {
    var mr *mapreduce.Master
    if os.Args[2] == "sequential" {
      mr = mapreduce.Sequential("wcseq", os.Args[3:], 3, mapF, reduceF)
    } else {
      mr = mapreduce.Distributed("wcseq", os.Args[3:], 3, os.Args[2])
    }   
    mr.Wait()
  } else {
    mapreduce.RunWorker(os.Args[2], os.Args[3], mapF, reduceF, 100)
  }
}                   
```

一个MapReduce程序由三个部分组成：

- Map函数
- Reduce函数
- 调用MapReduce执行的函数

## Map函数

Map函数主要的功能为吐出K/V对

## Reduce函数

Reduce函数则是对相同的Key做操作，一般是统计之类的功能，具体地看应用的需求。

## 调用MapReduce库函数

分为Sequential和Distributed，其中Sequential为串行地执行Map和Reduce任务，主要用于用户程序调试的场景，Distributed则用于真正的用户程序执行的场景。

# MapReduce单机版实现

本节实现的MapReduce单机版与Google论文中的MapReduce主要的不同如下：

- 输入和输出数据都采用本机的文件系统，没有使用到类似于GFS的分布式文件存储
- Google的MapReduce通过GFS的文件名字的原子操作来保证Reduce Worker宕机时，最终只会生成一份结果文件；在单机文件系统中，如果Worker和Master之间网络通信断掉，但是Worker本身可能还在工作，这时候如果重新启动另一个Worker可能会造成两个Worker写入同一份文件，这种场景，在单机版MapReduce的Worker容灾中不考虑。

本节分为两个部分来讨论：

- MapReduce的Sequential实现
- MapReduce的Distributed实现(带Worker容灾)

## MapReduce的Sequential实现

Sequential部分的调度程序实现如下：

```go
// Sequential runs map and reduce tasks sequentially, waiting for each task to            // complete before scheduling the next.
func Sequential(jobName string, files []string, nreduce int,
  mapF func(string, string) []KeyValue,
  reduceF func(string, []string) string,
) (mr *Master) {
  mr = newMaster("master")
  go mr.run(jobName, files, nreduce, func(phase jobPhase) {
    switch phase {
    case mapPhase:
      for i, f := range mr.files {
        doMap(mr.jobName, i, f, mr.nReduce, mapF)
      }
    case reducePhase:
      for i := 0; i < mr.nReduce; i++ {
        doReduce(mr.jobName, i, len(mr.files), reduceF)
      }
    } 
  }, func() {
    mr.stats = []int{len(files) + nreduce}
  })  
  return
}     
```

其逻辑非常简单，就是按照顺序先一个个的处理Map任务，处理完成之后，再一个个的处理Reduce任务。

接下来，看doMap和doReduce是如何实现的。

doMap的实现如下：

```go

// doMap does the job of a map worker: it reads one of the input files
// (inFile), calls the user-defined map function (mapF) for that file's
// contents, and partitions the output into nReduce intermediate files.
func doMap(
	jobName string, // the name of the MapReduce job
	mapTaskNumber int, // which map task this is
	inFile string,
	nReduce int, // the number of reduce task that will be run ("R" in the paper)
	mapF func(file string, contents string) []KeyValue,
) {
	// TODO:
	// You will need to write this function.
	// You can find the filename for this map task's input to reduce task number
	// r using reduceName(jobName, mapTaskNumber, r). The ihash function (given
	// below doMap) should be used to decide which file a given key belongs into.
	//
	// The intermediate output of a map task is stored in the file
	// system as multiple files whose name indicates which map task produced
	// them, as well as which reduce task they are for. Coming up with a
	// scheme for how to store the key/value pairs on disk can be tricky,
	// especially when taking into account that both keys and values could
	// contain newlines, quotes, and any other character you can think of.
	//
	// One format often used for serializing data to a byte stream that the
	// other end can correctly reconstruct is JSON. You are not required to
	// use JSON, but as the output of the reduce tasks *must* be JSON,
	// familiarizing yourself with it here may prove useful. You can write
	// out a data structure as a JSON string to a file using the commented
	// code below. The corresponding decoding functions can be found in
	// common_reduce.go.
	//
	//   enc := json.NewEncoder(file)
	//   for _, kv := ... {
	//     err := enc.Encode(&kv)
	//
	// Remember to close the file after you have written all the values!``
	file, err := os.Open(inFile)
	if err != nil {
		log.Fatal(err)
	}
	defer file.Close()
	scanner := bufio.NewScanner(file)
	contents := ""
	for scanner.Scan() {
		s := scanner.Text()
		s += "\n"
		contents = contents + s
	}
	kvs := mapF(inFile, contents)
	reduceFileMap := make(map[string]*os.File)
	jsonFileMap := make(map[string]*json.Encoder)
	for i := 0; i < nReduce; i++ {
		reduceFileName := reduceName(jobName, mapTaskNumber, i)
		file, err := os.Create(reduceFileName)
		if err != nil {
			log.Fatal(err)
		}
		enc := json.NewEncoder(file)
		reduceFileMap[reduceFileName] = file
		jsonFileMap[reduceFileName] = enc
		defer reduceFileMap[reduceFileName].Close()
	}
	for _, kv := range kvs {
		hashValue := int(ihash(kv.Key))
		fileNum := hashValue % nReduce
		reduceFileName := reduceName(jobName, mapTaskNumber, fileNum)
		enc, ok := jsonFileMap[reduceFileName]
		if !ok {
			log.Fatal(err)
		}
		err := enc.Encode(&kv)
		if err != nil {
			log.Fatal(err)
		}
	}
}

```

处理过程如下：

- 读入输出文件
- 调用用户指定的Map函数，吐出所有的K/V对
- 创建跟Reduce Worker相同数量的文件，然后，对每个K/V对，根据Key来做hash，输出到对应的文件

doReduce实现如下：

```go
// doReduce does the job of a reduce worker: it reads the intermediate
// key/value pairs (produced by the map phase) for this task, sorts the
// intermediate key/value pairs by key, calls the user-defined reduce function
// (reduceF) for each key, and writes the output to disk.
func doReduce(
	jobName string, // the name of the whole MapReduce job
	reduceTaskNumber int, // which reduce task this is
	nMap int, // the number of map tasks that were run ("M" in the paper)
	reduceF func(key string, values []string) string,
) {
	// TODO:
	// You will need to write this function.
	// You can find the intermediate file for this reduce task from map task number
	// m using reduceName(jobName, m, reduceTaskNumber).
	// Remember that you've encoded the values in the intermediate files, so you
	// will need to decode them. If you chose to use JSON, you can read out
	// multiple decoded values by creating a decoder, and then repeatedly calling
	// .Decode() on it until Decode() returns an error.
	//
	// You should write the reduced output in as JSON encoded KeyValue
	// objects to a file named mergeName(jobName, reduceTaskNumber). We require
	// you to use JSON here because that is what the merger than combines the
	// output from all the reduce tasks expects. There is nothing "special" about
	// JSON -- it is just the marshalling format we chose to use. It will look
	// something like this:
	//
	// enc := json.NewEncoder(mergeFile)
	// for key in ... {
	// 	enc.Encode(KeyValue{key, reduceF(...)})
	// }
	// file.Close()
	kvs := make(map[string][]string)
	for i := 0; i < nMap; i++ {
		reduceFileName := reduceName(jobName, i, reduceTaskNumber)
		file, err := os.Open(reduceFileName)
		if err != nil {
			log.Fatal(err)
		}
		defer file.Close()
		enc := json.NewDecoder(file)
		for {
			var kv KeyValue
			if err := enc.Decode(&kv); err == io.EOF {
				break
			} else if err != nil {
				log.Fatal(err)
			} else {
				log.Println(kv.Key + kv.Value)
				kvs[kv.Key] = append(kvs[kv.Key], kv.Value)
			}
		}
	}

	var keys []string
	for k, _ := range kvs {
		keys = append(keys, k)
	}
	sort.Sort(sort.StringSlice(keys))
	mergeFileName := mergeName(jobName, reduceTaskNumber)
	mergeFile, err := os.Create(mergeFileName)
	if err != nil {
		log.Fatal(err)
	}
	defer mergeFile.Close()
	enc := json.NewEncoder(mergeFile)
	for _, key := range keys {
		enc.Encode(KeyValue{key, reduceF(key, kvs[key])})
	}
}

```
Reduce任务的处理逻辑如下：

- 根据之前约定好的命名格式，找到该Reduce Worker需要处理的文件，然后，按照约定的方式进行解码
- 得到所有的K/V对之后，根据Key对K/V对排序
- 调用用户指定的ReduceF函数，对相同的Key的所有Value进行处理
- 把处理后的结果以一定的编码方式写入文件

## MapReduce的Distributed实现(带Worker容灾)

Distributed和Sequencial的主要区别在于调度函数的实现，如下

```go
// schedule starts and waits for all tasks in the given phase (Map or Reduce).
func (mr *Master) schedule(phase jobPhase) {
	var ntasks int
	var nios int // number of inputs (for reduce) or outputs (for map)
	switch phase {
	case mapPhase:
		ntasks = len(mr.files)
		nios = mr.nReduce
	case reducePhase:
		ntasks = mr.nReduce
		nios = len(mr.files)
	}

	fmt.Printf("Schedule: %v %v tasks (%d I/Os)\n", ntasks, phase, nios)

	// All ntasks tasks have to be scheduled on workers, and only once all of
	// them have been completed successfully should the function return.
	// Remember that workers may fail, and that any given worker may finish
	// multiple tasks.
	//
	// TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO TODO
	//
	switch phase {
	case mapPhase:
		mr.scheduleMap()
	case reducePhase:
		mr.scheduleReduce()
	}
	fmt.Printf("Schedule: %v phase done\n", phase)
}
```

分为Map和Reduce两阶段的调度，先来看ScheduleMap部分：

### ScheduleMap

```go
type taskStat struct {
	taskNumber int
	ok         bool
}

func (mr *Master) checkWorkerExist(w string) bool {
	mr.Lock()
	defer mr.Unlock()
	for _, v := range mr.workers {
		if w == v {
			return true
		}
	}
	return false
}

func (mr *Master) chooseTask(failedTasks []int, nTaskIndex int) ([]int, int) {
	if 0 == len(failedTasks) {
		return failedTasks, nTaskIndex
	} else {
		fmt.Println("choose failed tasks")
		task := failedTasks[0]
		failedTasks = failedTasks[1:len(failedTasks)]
		fmt.Println("failedTasks", failedTasks)
		return failedTasks, task
	}
}

func (mr *Master) runMapTask(nTaskNumber int, w string, doneTask chan taskStat) {
	if nTaskNumber < len(mr.files) {
		var args DoTaskArgs
		args.JobName = mr.jobName
		args.File = mr.files[nTaskNumber]
		args.Phase = mapPhase
		args.TaskNumber = nTaskNumber
		args.NumOtherPhase = mr.nReduce
		go func() {
			ok := call(w, "Worker.DoTask", args, new(struct{}))
			var taskstat taskStat
			taskstat.taskNumber = args.TaskNumber
			taskstat.ok = ok
			if ok {
				doneTask <- taskstat
				fmt.Println("done task %d", args.TaskNumber)
			} else {
				doneTask <- taskstat
				fmt.Println("get failed task %d", args.TaskNumber)
			}
		}()
	} else {
		fmt.Printf("all tasks sent out")
	}
}

func (mr *Master) scheduleMap() {
	fmt.Println("begin scheduling map tasks")
	taskWorkerMap := make(map[int]string)
	doneTask := make(chan taskStat, 1)
	var nTaskIndex = 0
	var failedTasks []int
	mr.Lock()
	var nInitTask = min(len(mr.files), len(mr.workers))
	mr.Unlock()
	for ; nTaskIndex < nInitTask; nTaskIndex++ {
		mr.Lock()
		w := mr.workers[nTaskIndex]
		mr.Unlock()
		mr.runMapTask(nTaskIndex, w, doneTask)
	}

	for {
		select {
		case newWorker := <-mr.registerChannel:
			fmt.Println("New Worker register %s", newWorker)
			var nextTask int
			failedTasks, nextTask = mr.chooseTask(failedTasks, nTaskIndex)
			if nextTask < len(mr.files) {
				fmt.Println("nextTask %d, total %d", nextTask, len(mr.files))
				mr.runMapTask(nextTask, newWorker, doneTask)
				taskWorkerMap[nextTask] = newWorker
				if nTaskIndex == nextTask {
					nTaskIndex++
				}
			}
		case taskStat := <-doneTask:
			var w string
			taskNumber, ok := taskStat.taskNumber, taskStat.ok
			if !ok {
				failedTasks = append(failedTasks, taskNumber)
			} else {
				w = taskWorkerMap[taskNumber]
				delete(taskWorkerMap, taskNumber)
			}
			if mr.checkWorkerExist(w) {
				fmt.Println("failed task count, failed tasks", len(failedTasks), failedTasks)
				var nextTask int
				failedTasks, nextTask = mr.chooseTask(failedTasks, nTaskIndex)
				if nextTask < len(mr.files) {
					fmt.Println("failed task count, failed tasks", len(failedTasks), failedTasks)
					fmt.Println("nextTask %d, total %d", nextTask, len(mr.files))
					mr.runMapTask(nextTask, w, doneTask)
					taskWorkerMap[nextTask] = w
					if nTaskIndex == nextTask {
						nTaskIndex++
					}
				}
			}
			fmt.Println("task index %d, task number %d, map count", nTaskIndex, taskNumber, len(taskWorkerMap))
			for k, v := range taskWorkerMap {
				fmt.Println("%s:%v", k, v)
			}
			if (nTaskIndex == len(mr.files)) && (0 == len(taskWorkerMap)) {
				fmt.Println("all tasks in mapPhase is done")
				return
			}
		}

	}
}

```

- 先根据已注册的Worker数量，生成相应数量的Map任务，然后发送给Worker执行
- 接下来在，在select中处理两种事件：一是有新Worker注册的事件；二是之前调度的任务执行完成的事件

**不带Worker容灾的处理：**

处理新Worker注册的事件的方式为选择下一个要执行的任务，发送给新注册的Worker去执行。

处理调度完成的事件的方式为选择下个需要执行的任务，调度给刚刚完成执行任务的Worker执行。当所有的Map任务都处理完成后，表示Map阶段完成，退出调度。

**带Worker容灾的处理：**

Worker容灾的处理逻辑为，当任务执行失败时，加入到执行失败的任务队列中，当发生上述两种事件时，先从失败的任务队列中拿下一个任务执行，只有当失败的任务队列为空时，才调度新的任务执行。

### ScheduleReduce

ScheduleReduce的实现如下

```go
func (mr *Master) scheduleReduce() {
	fmt.Println("start scheduling reduce tasks")
	taskWorkerMap := make(map[int]string)
	doneTask := make(chan taskStat, 1)
	var failedTasks []int
	var nTaskIndex = 0
	mr.Lock()
	var nInitTask = min(mr.nReduce, len(mr.workers))
	mr.Unlock()
	for ; nTaskIndex < nInitTask; nTaskIndex++ {
		mr.Lock()
		w := mr.workers[nTaskIndex]
		mr.Unlock()
		taskWorkerMap[nTaskIndex] = w
		mr.runReduceTask(nTaskIndex, w, doneTask)
	}

	for {
		select {
		case newWorker := <-mr.registerChannel:
			fmt.Println("New Worker register %s", newWorker)
			var nextTask int
			failedTasks, nextTask = mr.chooseTask(failedTasks, nTaskIndex)
			if nextTask < mr.nReduce {
				fmt.Println("nextTask %d, total %d", nextTask, len(mr.files))
				mr.runReduceTask(nextTask, newWorker, doneTask)
				taskWorkerMap[nextTask] = newWorker
				if nTaskIndex == nextTask {
					nTaskIndex++
				}
			}
		case taskStat := <-doneTask:
			var w string
			taskNumber, ok := taskStat.taskNumber, taskStat.ok
			if !ok {
				failedTasks = append(failedTasks, taskNumber)
			} else {
				w = taskWorkerMap[taskNumber]
				delete(taskWorkerMap, taskNumber)
			}
			if mr.checkWorkerExist(w) {
				var nextTask int
				failedTasks, nextTask = mr.chooseTask(failedTasks, nTaskIndex)
				if nextTask < mr.nReduce {
					fmt.Println("nextTask %d, total %d", nextTask, len(mr.files))
					mr.runReduceTask(nextTask, w, doneTask)
					taskWorkerMap[nextTask] = w
					if nTaskIndex == nextTask {
						nTaskIndex++
					}
				}
			}
			fmt.Println("task index %d, task number %d, map count", nTaskIndex, taskNumber, len(taskWorkerMap))
			for k, v := range taskWorkerMap {
				fmt.Println("%s:%v", k, v)
			}
			if (nTaskIndex == mr.nReduce) && (0 == len(taskWorkerMap)) {
				fmt.Println("all tasks in mapPhase is done")
				return
			}
		}
	}
}
```

整体的处理逻辑和ScheduleMap类似，如下

- 先根据已注册的Worker数量，生成相应数量的Reduce任务，然后发送给Worker执行
- 接下来在，在select中处理两种事件：一是有新Worker注册的事件；二是之前调度的任务执行完成的事件

容灾过程也是类似的，不再赘述。

博客中用到的程序放在[链接](https://github.com/Charles0429/toys/tree/master/6.824/src/mapreduce)中，有需要的同学自取。

PS:
本博客更新会在第一时间推送到微信公众号，欢迎大家关注。

![qocde_wechat](http://o8m1nd933.bkt.clouddn.com/blog/qcode_wechat.jpg)

# 参考文献

- [MapReduce: Simplified Data Processing on Large Clusters](http://research.google.com/archive/mapreduce-osdi04.pdf)
- [MapReduce原理](http://oserror.com/distributed/mapreduce/)
- [MIT 6.824 MapReduce lab](https://pdos.csail.mit.edu/6.824/labs/lab-1.html)
