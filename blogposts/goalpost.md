---
date: 2020-06-01T12:00
tags:
  - blog/published
---

# Goalpost

## a durable, embeddable, worker queue for golang

##### [Original article](http://tilde.club/~ngp/blob/posts/goalpost/)

One thing that has always stuck out to me about the Go programming language is it's incredibly strong concurrency model. Go is aggressively simple as a language, and it's asynchronous architecture is no different. Channels are a beautiful implementation of a buffered FIFO queue which have an infinite number of applications. There's no `Thread` classes, just a simple `go` in front of a function call. It's great! You can probably tell, but I enjoy working with Go a lot.

With that, I'd like to introduce a small side-project of mine. I call it
[goalpost](https://github.com/chiefnoah/goalpost/blob/master/queue.go). It's a simple, embeddable, worker queue library. It seeks to be a simple way to reliably do *work* in uncertain runtime environments.

<!--more-->

## What's a work queue?

Work or job queues are a way of reliably executing a task in an asyncronous manner. This idea is that when you push a message onto the queue, one or more workers can pick up the message and begin acting on it, typically in a different process or thread. One of the most popular worker queue implementations is [RabbitMQ](https://www.rabbitmq.com/).
RabbitMQ is incredibly mature and robust, and one of the most widely used queue systems in the world. It supports durability, permissions, federation, and many other advanced features. There are many other queues ([redis](https://redis.io/), for example, can be used as a queue), RabbitMQ is just *one* of the many out there.

> If RabbitMQ is so great, why did you make goalpost?

Good question! You see, while RabbitMQ is great, it's a little on the heavy side and requires a fair bit of setup. Further, it requires binding to a TCP port in order to work. While that's *usually* not much of a problem, there are scenarios when that's undesirable or not even an option.

Goalpost started as part of another project (which isn't open source, sorry!), but was designed in a way that it was easy to rip out and turn into it's own external dependency for the project.

The service was intended to be run on a Linux system where binding to a TCP port wasn't an option, and the server could restart at almost any point. The existing system, written as a combination of bash and python scripts, worked but was susceptible to failing ungracefully when the server rebooted, and worse, wouldn't retry when the server came back up. This means that jobs could get "lost" and never finish execution. Ultimately, this resulted in little more than a minor inconvenience for me, but I wanted the whole thing to be completely hands-off as much as possible.

## Exploration

Initially, I looked into other options. Like any good developer, I'm lazy and don't want to rewrite something someone has already done. Ok, part of me really wanted to try writing a queue, but if it already existed I would've taken the lazy path 🙃

There were a few projects that came close to what I needed, but didn't have everything I was looking for.

[goque](https://github.com/beeker1121/goque) seemed promising, but it doesn't actually implement a work queue, and it seems the project is dead (last commit was 3 years ago).

[dque](https://github.com/joncrlsn/dque) was another one I looked at. I actually like dque a lot, but again, not a worker queue. I considered trying to modify it it and add worker queue, but after looking through the code, decided it would probably be easier to just start a new project.

This wonderful post on OpsDash on [Job Queues in Go](https://www.opsdash.com/blog/job-queues-in-go.html) was super helpful, but lacked durability, given that it relies entirely on go channels.

If there were other options, they certainly made themselves hard to find!

## Choosing a storage backend

One of the primary goals of goalpost was to be able to survive un-expected restarts, and also shutdown gracefully when possible. As such, it needs a way of persisting jobs to disk.

Go has several projects that seek to handle everything from in-memory, embedded SQL engines, to persistent, fast, key-value stores. Etcd, a popular distributed key-value store, is backed by a forked version of boltdb. The [original project](https://github.com/boltdb/bolt) was set to read-only awhile ago, and was subsequently forked by the etcd project to continue adding new features as the etcd needed. Both would work, as the interface and practical use-case is the same for both, so I went with etcd's fork: bbolt.

> Key-value stores aren't a great way to do queues

You're probably right, but when combined with a buffered notifications system, they can get the job done well enough. An embedded SQL engine would've been overkill for this, and I didn't want to write my own storage backend. Lazy, remember?

## Writing some code...

I'm not going to go into too much detail on my process for writing goalpost, mostly because it's been months since I originally wrote it. It lived for awhile as part of that other project, until I finally got the motivation to rip it out and properly document everything.

One of the first things I did was define the `Job` struct and the `JobStatus` constant types. I wanted to get the basics of what state the jobs could be in out of the way first. It ended up working out pretty well, as the only fields that were added to `Job` was the `Message` field, and nothing was added to `JobStatus`. I _did_ reference parts of the documentation for RabbitMQ's worker model, which helped with the `JobStatus` definitions, with my implementation being a subset of RabbitMQ's job states with some modifications to the semantics. There may be room for a new state such as `new` to better indicate that a job has not been started by a worker at all yet, but as of right now it's
not strictly needed.

The possible job states are as follows:

`uack` - This is the original state a job is in when it's pushed to the queue.
The meaning is "unacknowledged", meaning the job has not been completed yet.
Workers *do* update the state, but it's to `uack`.

`nack` - This is similar to the AMQP extension, except it simply means "job
processing failed, but should put back on the queue and retried"

`ack` - The basic 'acknowledged' state. This means a job has completed
processing and should not be re-attempted. "Completed"

`failed` - Indicates a hard failure in the job. An unrecoverable error, and the
job should not be reattempted.

It's worth pointing out that the workers `DoWork` function should *never* modify the job status themselves. Doing so could potentially caused undefined behavior.

### Creating a queue registering workers and looping forever

Creating a queue is pretty easy. You call `goalpost.Init(queueID)` with a unique id for the queue, which is used to initialize or open the database. A WaitGroup and a chan are created for handling shutdown and job events. Once a queue is initialized, you need to register a worker in order to start processing jobs.

When a worker is registered, the queue does some setup that allows it to keep track of workers. It then executes the main poll loop with the worker in a separate goroutine, sleeping for 500 milliseconds (now configurable in `Queue`) after checking to either shut down or work a job. Context.context can be used to signal to your `DoWork` function that the queue has received a signal to stop, and it should make an attempt to exit early.

When `PushBytes` is called, a  `Job` is created and written to the boltdb database, then a notification with the resuling JobID is sent to the chan that was created when the queue was initialized. A worker reads from the chan, receiving the JobID, which it then reads from the database for processing.

When a job is marked as `ack`ed or `failed`, the job gets moved to a second bucket, which is purely used for reference. This also should help with performance in certain scenarios, such as when trying to resume uncompleted jobs where the entire bucket is scanned for `uack`ed jobs.

### Using goalpost

Goalpost seeks to be an easy to use. How easy? Well I think it's pretty easy, but you decide how easy it is for yourself.

First, you need to of course import the goalpost package:

```golang
import "git.packetlostandfound.us/chiefnoah/goalpost"
```

Too slow? Ok fine, here's a basic implementation for a Worker. This is basically copy pasted from [the example](https://github.com/chiefnoah/goalpost/blob/master/examples/basic.go), but I'll include it here for redundancy's sake:

```golang
type worker struct {
	id string
}

func (w *worker) ID() string {
	return w.id
}

func (w *worker) DoWork(ctx context.Context, j *goalpost.Job) error {
	//do something cool!
	fmt.Printf("Hello, %s\n", j.Data)
	//Something broke, but we should retry it...
	if j.RetryCount < 9 { //try 10 times
		return goalpost.NewRecoverableWorkerError("Something broke, try again")
	}

	//Something *really* broke, don't retry
	//return errors.New("Something broke, badly")

	//Everything's fine, we're done here
	return nil
}
```

You can do whatever you want in the DoWork function. It's executed in a separate goroutine from the main thread, so keep that in mind if you're interacting with things outside of the worker thread. I recommend storing configuration and shared types (like a database connection) in the type that implements the Worker interface, in this case, the `worker` struct.

Now is probably a good time to talk about error handling. Errors returned from the DoWork function can be of two types:

* `goalpost.RecoverableWorkerError`
* everything else

`goalpost.RecoverableWorkerError` is used to indicate a temporary failure, for example a timeout on a web request. When the queue receives an error of that type from a `DoWork` function, it increments the `Jon.RetryCount`, marks the job as `nack`ed, and pushes the event back onto the queue. If there are workers available to process the job, it will be processed again immediately. It's recommended that you check `Job.RetryCount` when returning `RecoverableWorkerError`s, as failing to do so could result in a job being retried infinitely. Maybe that's what you want? But chances are it's **not**.

Next up is initializing the `Queue` and registering a worker:

```golang
func main() {

	//Init a queue
	q, _ := goalpost.Init(eventQueueID)
	//remember to handle your errors :)

	//Create a worker with id "worker-id"
	w := &worker{
		id: "worker-1",
	}
	//register the worker, so it can do work
	q.RegisterWorker(w)

	//Let's do some work...
	q.PushBytes([]byte("World"))
	//You should see "Hello, World" printed 10 times

	//Make sure your process doesn't exit before your workers can do their work
	time.Sleep(10 * time.Second)

	//Remember to close your queue when you're done using it
	q.Close()
}
```
You can initialize a queue from anywhere, here we do it in `main` for
simplicity. The `Queue` type is thread safe, but you *cannot* have multiple instances of a `Queue` with the same ID/filepath. This is a limitation of boltdb/bbolt. Attempts to initialize a second queue with the same database file will result in errors.

Once you have a queue initialized, you need to create an instance of your `Worker` type and register it with the queue with the `RegisterWorker` method. You can register the same worker instance multiple times, but logging will report the same ID, and you must be mindful of shared resources in `*w`. I recommend creating multiple instances of your worker type and registering each once.

With some workers watching the queue, you can now push data into the queue for processing. In the example, we only push a byte encoded string, but any `[]byte` will work. For my application, I use `encoding/gob` to pass whole structs to my worker.

Another thing to remember is that you need to keep your main process alive. If `main` exits, the goroutines with your workers will receive a signal to shutdown, and may leave unprocessed jobs in the queue. There currently is no way to see if there are jobs that haven't been started, though such a feature could be implemented if the need arises.

I use the following snippet followed by a eternally blocking `http.ListenAndServe` call to gracefully shutdown the queue:

```golang

c := make(chan os.Signal)
signal.Notify(c, os.Interrupt, syscall.SIGTERM)
signal.Notify(c, os.Interrupt, syscall.SIGINT)
go func() {
	<-c
	log.Printf("Closing queue...")
	q.Close()

	os.Exit(0)
}()
```

### Closing things out

I had a lot of fun writing this little library, and I hope someone else can find it useful! It's my first library that I'm opening up to the public, so things may be a little rough. As always, feel free to shoot me an email at noah **at** packetlost.dev with feedback or patches. Pull requests on GitHub also welcome!

Oh, and remember to call `.Close()`

