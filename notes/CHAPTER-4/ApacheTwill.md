# Twill
**Slider** runs existing applications while **Twill** helps create new ones.

**Problem:**

Raw YARN code is painful. Normally you would write:

```Text
request containers
launch processes
monitor containers
collect logs
retry failures
```

Twill lets you write something simpler.

**Example idea:**

```Java
class Worker implements Runnable
```

You define:

```Java
class ImageProcessor implements Runnable {
    public void run() {
        process();
    }
}
```

Twill turns that into:

```
Runnable
 ↓
YARN Container
 ↓
Distributed execution
```

> Suppose you want: 20 workers

Twill:

```
Creates 20 containers
Runs 20 Runnable instances
Collects logs
```

**You don't manage YARN directly.**

### Runnable

**Runnable** is Java’s Runnable interface, not a **YARN** concept.

```Java
public interface Runnable {
    void run();
}
```
If a class implements Runnable, it must provide:

```Java
run()
```

Example:
```Java
class Printer implements Runnable {

    @Override
    public void run() {
        processImages();
    }

}
```
> When someone executes this Runnable, call run().
```Java
Normally in Java, Runnable is often used with threads.

Runnable task = new Printer();

Thread t = new Thread(task);

t.start();
```
Equals to:
```Java
Thread t = new Thread(() -> {
    System.out.println("Hello");
});
```

**Twill does something conceptually like:**
```
Create Container
 ↓
Start JVM
 ↓
Instantiate ImageProcessor
 ↓
Call run()
```

If you request **10 instances**, so **Twill** turns **Runnable** into ***Distributed process running on YARN***.

```
Container1 → ImageProcessor.run()
Container2 → ImageProcessor.run()
Container3 → ImageProcessor.run()
...
Container10 → ImageProcessor.run()
```
