# Parallelism simplified by Akka in Java /Scala

This is a simple sample project showing AKKA in Java. It is a simple code that shows how easy it is to setup AKKA and how big the performance gain is.

*May 10, 2016 *
I have bean writing applications in Java EE environment for some time now and I didn’t need to worry about creating own threads. When I needed to create and manage my own threads it always felt wrong because of to much low level programming. Then I came across akka and wanted to try it out.

##What is Akka?
Quote from akka.io “We believe that writing correct concurrent, fault-tolerant and scalable applications is too hard. Most of the time it’s because we are using the wrong tools and the wrong level of abstraction. Akka is here to change that. Using the Actor Model we raise the abstraction level and provide a better platform to build correct, concurrent, and scalable applications. For fault-tolerance we adopt the “Let it crash”; model which the telecom industry has used with great success to build applications that self-heal and systems that never stop. Actors also provide the abstraction for transparent distribution and the basis for truly scalable and fault-tolerant applications.”

### Context
For the purpose of the sample we have a trivial working unit that simulates CPU intensive task that can find a factorial for a large number. We assume that we need to find factorials for many different numbers, for the purpose of consistency the number is now fixed.
Factorial calculator

```java
public class CalculateFactorial {
    public BigInteger calculate() {
        BigInteger fact = BigInteger.valueOf(1);
        for (int i = 1; i <= 739; i++)
            fact = fact.multiply(BigInteger.valueOf(i));
        return fact;}}
```
## Java way
The most simple way of implementing this is to loop through the numbers and calculate the factorial for them, like this.  

```java
public class JavaWay {
    private final long messages = 100;
    
    public static void main(String[] array) {
        new JavaWay().run();}

    private void run() {
        time.start();
        calculateFactorial();
        time.end();
        printElapsedTime(time);}

    private void calculateFactorial() {
        for (int i = 0; i < messages; i++) {
            list.add(new CalculateFactorial().calculate());}}

    private void printElapsedTime(Time time) {
        System.out.println("Done: " + time.elapsedTimeMilliseconds());}}
```

This is a simple solution that works well, but the problem with it is that the processing speed entirely depends on one thread on the local machine even though the messages could be processed simultaneously. It is possible to solve this problem in many different ways, for example by using a container managed environment. Although it can solve the problem it brings complexity and overhead to the project. Another way is to use threading, but that means working on a low level coding and it does not allow distributed scaling. But there is also Akka way.  

### Akka way
Akka can be used as a library like in this sample, as a microkernel or part of the Typesafe Platform. This sample consists of four parts: bootstrapping Akka (starting ActorSystem), master actor that handles the messages, worker actor that handles the calculating process and at last there are triggers (messages).  

Bootstrapping Akka consists of starting the ActorSystem, creating the master actor and telling it to start processing.  

```java
public class AkkaWay {

    public static void main(String[] args) {
        new AkkaWay().run();}

    private void run() {
        ActorSystem system = ActorSystem.create("CalcSystem");
        ActorRef master = system.actorOf(Master.createMaster(), "master");
        master.tell(new Calculate(), ActorRef.noSender());}}
```

Master actor is responsible for processing the messages, meaning telling the Worker actor to calculate the factorial. Akka has different ways to scale, in this sample RoundRobinRouter is being used so that multiple Worker actors can do their work at once. Master worker is also responsible for receiving the results from the Worker actors and knowing when all messages are processed.  

```java
public class Master extends UntypedActor {

    private long messages = 100;
    private long processed = 0;
    private ActorRef workerRouter;
    private final Time time = new Time();
    private ArrayList list = new ArrayList();

    public Master() {
        workerRouter = this.getContext().actorOf(Worker.createWorker().withRouter(new RoundRobinRouter(4)), "workerRouter");
    }

    @Override
    public void onReceive(Object message) {
        if (message instanceof Calculate) {
            time.start();
            processMessages();
        } else if (message instanceof Result) {
            list.add(((Result) message).getFactorial());
            if (list.size() == messages)
                end();
        } else {
            unhandled(message);
        }
    }

    private void processMessages() {
        for (int i = 0; i < messages; i++) {
            workerRouter.tell(new Work(), getSelf());}}

    private void end() {
        time.end();
        System.out.println("Done: " + time.elapsedTimeMilliseconds());
        getContext().system().shutdown();}

    public static Props createMaster() {
        return Props.create(Master.class, new ArraySeq<Object>(0));}}
```

Worker actor is responsible for the actual factorial calculation and returning a response to the parent (master actor) when done.  

```java
public class Worker extends UntypedActor {

    @Override
    public void onReceive(Object message) {
        if (message instanceof Work) {
            BigInteger bigInt = new CalculateFactorial().calculate();
            getSender().tell(new Result(bigInt), getSelf());
        } else
            unhandled(message);
    }

    public static Props createWorker() {
        return Props.create(Worker.class, new ArraySeq<Object>(0));}}
```
For communication between actors Akka uses triggers (messages) that are immutable objects.

```java
public class Calculate {}

public class Result {

    private BigInteger bigInt;

    public Result(BigInteger bigInt) {
        this.bigInt = bigInt;}

    public BigInteger getFactorial() {
        return this.bigInt;}}
```

This is all the code that is needed to take advantage of parallelism. If local scaling is not enough Akka also has a simple distributed solution build in that allows the Worker actor to be run remotely without changing the code.  

```code

Speed: Java way vs Akka way

Messages	Java (milliseconds)	Akka quad (milliseconds)
100	53	38
1.000	302	139
10.000	2661	1072
1.000.000	247579	102315
```

## Conclusion
This sample shows that it is quite easy to take advantage of parallelism by using Akka to speed up the application. Because Akka is on a higher abstraction level than threads themselves it is possible to speed up the application even further by changing the actors number or by running it distributed.

##Sources
The working project and the complete code can be found on this repo.  

For more information there are books like Akka concurrency and Akka in action or watch a talk from Jonas Bonér  
