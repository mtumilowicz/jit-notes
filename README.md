[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)

# jit-notes
* references
    * [JVM JIT for Dummies](https://www.youtube.com/watch?v=0Yud4Q2HEz4)
    * [Douglas Hawkins — Understanding the Tricks Behind the JIT](https://www.youtube.com/watch?v=oH4_unx8eJQ)
    * [JVM Mechanics by Douglas Hawkins](https://www.youtube.com/watch?v=a-U0so9FfqQ)
    * [Devoxx Poland 2016 - Douglas Hawkins - A Peak Inside the JIT](https://www.youtube.com/watch?v=6ICU2xo427M)
    * [Understand the Trade-offs of Using Compilers for Java Applications](https://www.youtube.com/watch?v=C_dV3LQY9TA)
    * [Toruń JUG #29 - "JIT me baby one more time" - Jarek Pałka](https://www.youtube.com/watch?v=GXyM7IOTXOM)
    * [WJUG #257 - Krzysztof Ślusarski - Just-In-Time compiler - ukryty "przyjaciel"](https://www.youtube.com/watch?v=f8zaYDJctTA)
    * [An Introduction to JVM Performance](https://www.youtube.com/watch?v=hjpzLXoUu1Y)
    * [Douglas Hawkins playlist](https://www.youtube.com/playlist?list=PLDnxsaeAegRduDndE6P1MUYRFYoYhGvsV)
    * https://stackoverflow.com/questions/56698783/what-is-the-difference-between-java-intrinsic-and-native-methods
    * https://stackoverflow.com/questions/9105505/differences-between-just-in-time-compilation-and-on-stack-replacement
    * https://www.slideshare.net/dougqh
    * https://blog.takipi.com/java-on-steroids-5-super-useful-jit-optimization-techniques/
    * https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-1-jvm-types-and-compiler-modes/
    * https://blog.joda.org/2011/08/printcompilation-jvm-flag.html
    * https://medium.com/@julio.falbo/understand-jvm-and-jit-compiler-part-2-cc6f26fff721

## general
* JIT - just in time compiler
* "JIT is just complex pattern matchers" - Douglas Hawkins
* a technique of code compilation during runtime
    * first usage: LISP
    * code is compiled to something intermediary (bytecode) and then is compiled to native code 
    (machine code) during runtime
* opposed to Ahead Of Time (AOT) compilation: C, C++, Go, Rust
    * during compilation -> machine code
    * many optimizations during compilation
    * restart - we start from the beginning
        * java 9 - AOTC compiler
* loop of JIT compilation: interpret -> profile -> compile -> deoptimize -> interpret
* optimizes two things: methods and loops
* big part of JVM
    * C2 - 19%
    * C1 - 6,3%
    * gc - 15%
* useful flags
    * `-Xint` - flag forces the JVM to execute all bytecode in interpreted mode, which comes along with a 
    considerable slowdown, usually factor 10 or higher
    * `-XX:+PrintCompilation` - show basic information on when Hotspot compiles methods
        * shows timespan from start-up to compilation
        * shows compilation identifier (each compilation bumps identifier by 1)
            * C1 and C2 have separate identifiers
* hot spots - programming parts that are executed a lot
    * spring configuration classes - only once during bootstrap
    * rest controller methods - many times
* jit compilers consume transient resources (CPU cycles and memory)
    * from under a millisecond to seconds of compile time
    * can allocate 100s MBs
    * takes time to get to "full speed" because there may be 1000s of methods to compile
* collecting profile data is an overhead
    * cost usually paid while code is interpreted: slows start-up and ramp-up

## mechanics
* tiered
    ![alt text](img/tiered_compilation.png)
    * interpreted -> C1: 5x faster -> C2: 2-10x faster
    * compilation speed: 
        * C1:Client: Fast
        * C2:Server: Slow(4x)
    * execution speed
        * C1:Client: Slow
        * C2:Server: Fast(2x)
* counters
    * hot method: invocation counter > invocation threshold
    * hot loops: backedge (loop) counter > backedge threshold
    * invocation + backedge counter > compile threshold
        * medium hot methods with medium hot loops
    * if counter > threshold -> compile loop
    * C1: 2000 invocations
    * C2: 10000 invocations
* Compiler Task Queue
    * JIT compilation is an asynchronous process
    * when the JVM decides that a certain block of code should be compiled, that block of code is put in a queue
        * not strictly first in, first out; methods whose invocation counters are higher have priority
    * the JVM will continue interpreting the method, and the next time 
    the method is called, the JVM will execute the compiled version of the method
    * if method stays in a queue for too long - it will be removed from the queue
        * maybe we don't need optimization for that method anymore
    * on-stack replacement (OSR) context
        * consider a long-running loop
        * JVM has to have the ability to start executing the compiled version of the loop while 
        the loop is still running
        * when the code for the loop has finished compiling, the JVM replaces the code (on the stack), and the 
        next iteration of the loop will execute the much faster-compiled version of the code
* made not entrant, made zombie
    * PrintCompilation output
    * made not entrant
        * there could exist many compilations of a single method simultaneosly
        * if methodA calls methodB, it wants to have the best compilation
            * if methodA calls methodB for the first time - it asks Call Dispatcher for the best compilation
                * methodA caches that information
        * if methodA calls methodB not for the first time and the methodB is made not entrant - methodA calls dispatcher
            * if there is better compilation, we have two compilations in the memory
                * in the meantime dispatcher marks slower compilation as made not entrant and redirect calls
                to the faster one
    * made zombie - method was made not entrant and no thread has it on its stack
        * could be removed
### c1
* fast
* trivial methods, up to 6 bytes (MaxTrivialSize) are inlined by default
    * up to 35 bytes (in bytecode) (MaxInlineSize)
    * inlining depth limit: 9 methods deep

### c2
* slow but generates faster code (compared to C1)
* by default limited by bytecode size of 325 (FreqInlineSize)
    * if > 8000 bytecode - always interpreted

## optimizations
* golden rule of optimization: don't do unnecessary work
* speculative optimizations
* method inlining
    * tuning
        * -XX:+MaxInlineSize=35
        * -XX:+MaxInlineLevel=9
        * -XX:+MaxRecursiveInlineLevel=#
* loop unrolling
* lock coarsening/eliding
* coarsening
    ```
    public void needsLocks() {
        loop {
            process()
        }
    }
    private synchronized String process(String option) { }  
    ```
    compiled into
    ```
    public void needsLocks() {
        synchronized(this) {
            loop {
                // inlined process()
            }
        }
    }
    ```
* eliding
    ```
    public void elide() {
        var l = new ArrayList();
        synchronized(l) { // lock on local variable
            loop {
                 l.add(...) // l never escapes this thread
            }
        }
    }
    ```
    compiled into
    ```
    public void elide() {
        var l = new ArrayList();
        loop {
            l.add(...)
        }
    }
    ```
* dead code elimination
* duplicate code elimination
* escape analysis
    ```
    public void x() {
        var f = new Foo("X", "Y");
        baz(f)
    }
  
    public void baz(Foo f) {
        sout(f.a)
        sout(",")
        quux(f)
    }
    
    public void quux(Foo f) {
        sout(f.b)
        sout("!")
    }
    ```
    compiled into
    ```
    public void x() { // don't bother allocating foo object
        sout("X")
        sout(",")
        sout("Y")
        sout("!")
    }
    ```
* intrinsic
    * JVM knows the implementation of an intrinsic method and can substitute the original java-code with 
    machine-dependent well-optimized instructions (sometimes even with a single processor instruction), 
    whereas the implementation of a JNI method is unknown to a JVM    
    * known to the jit
    * don't inline bytecode
    * do insert "best" native code
        * e.g. kernel-level memory operation
        * e.g. optimized sqrt in machine code
    * common intrinsics
        * String#equals
        * most Math methods
        * System.arraycopy
        * Object#hashCode
        * Object#getClass
* mismatch and no optimization
    ```
    p c X {
        E[] data
        int size
        
        p X (E.. elems) {
            data = elems
            size = elems.length
        }
        
        forEach(consumer action) {
            ...
            for (int i = 0; i < this.size; i++) { // mismatch with this.elementData.length
            ...
            if (i >= this.elementData.length) throw new AIOBE();
            ...
            action.accept(this.elementData[i])
            }
            ...
        }
    }
    ```
    * two checks rather than one

    
* compilation level (tiered compilation)
    ![alt text](img/jit/tiered-compilations.png)
    * 0: interpreter
    * 1: C1 with full optimization (no profiling)
    * 2: C1 with invocation and backedge counters
        * liczy ilosc wywolan metody (invocation counter)
        * wszystkie miejsca w ktore wracamy z petli (backedge counter) - sprawdza
        czy czasami petla nie jest goraca
            * backedge counter - najlatwiej sobie to wyobrazic jako skok do gory
            petli, kolejna iteracja
    * 3: C1 with full profiling (level 2 and MethodData)
        * ile razy weszlismy w ifa a ile razy w else
        * sprawdza jaki typ zwrociliscie w tym miejscu
        * sprawdza na jakim typie obiektu wolaliscie te metode (wolalismy na A czy 
        na obiekcie B ktory dziedziczy po A)
    * 4: C2
* ktory kod jest kompilowany?
    * 2000 invocations for C1
    * 10000 invocations for c2
    * trivial methods - tutaj c2 jest w ogóle nie używana
* escape analysis i registry allocation
    * inlining: expanding optimizations horizon
        * matka wszystkich optymalizacji, czyni inne optymalizacje mozliwymi
        * jakby wywalic wszystko i zostawic inline to i tak byloby szybko Joshua Bloch
* to nigdy nie jest jedna optymalizacja tylko sekwencja optymalizacji
    * zaczyna sie mniej wiecej od escape analysis albo inliningu
    ```
    p s v x(object) {
        if (obj == null) {
            sout('a');
        }
    }
    
    p v y() {
        x(this);
    }
    ```
    inlining:
    ```
    p v y() {
       if (obj == null) {
           sout('a');
       }
    }
    ```
    null-check folding:
    ```
    p v y() {
       if (false) {
           sout('a');
       }
    }
    ```
    death code termination:
    ```
    p v y() {
    }
    ```
          
* constant folding and propagation
    ```
    p s l x() {
        int x = 14;
        int y = 7 - x / 2;
        return y * (28 / x + 2); // to wszystko skompilowane do return 0
    }
    ```
* pointer compare
    ```
    p s int x(Object obj) {
        Object o = new Object(); // escape analysis - ten obiekt jest lokalny dla tej metody
        if (obj == o) {
            return 0;
        }
        return -1;
    }
    ```
    to
    ```
    p s int x(Object obj) {
        return -1;
    }
    ```
* intrinsics - to sa kawalki kodu ktore wiemy jak z gory beda wyglada w assemblerze
    * np. arraycopy - jestesmy w stanie to zrobic 4 instrukcjami assemblera bez petli
    * Unsafe.allocateInstance - stworzenie instancji obiektu nie wywolujac konstruktora
    * Unsafe.storeFence - bezposrednie wywolanie instrukcji CPU
    * `@HotSpotIntrinsticCandidate` - w pakiecie math jest tego bardzo duzo
* lock elision
    ```
    p s i x(int j) {
        Object lock = new Object();
        synchronized (lock) { // to zostanie usuniete
          j++;
        }
        
        return j;
    }
    ```
* lock coarsening
  * potrafi tez dwa locki na tym samym monitorze zaraz po sobie zmerdzowac 
  w jeden duzy lock
* loop unrolling
* redundancy removal - jesli wie ze wynik bedzie taki sam to zaladuje go z cache
* implicit checks eliminations
    ```
    p s int getSize(Collection collection) {
        return collection.size();
    }
    ```
    to tak naprawdę
    ```
    p s int getSize(Collection collection) {
        if (collection == null) { 
            throw new NullPointerException();
        }
        return collection.size(); // w assemblerze jakbys to wywoolal na nullu - segmentation violation
        // crush calej maszyny
    }
    ```
* unused branch removal - jesli w trakcie profilowania nie wchodzisz do brancza to
mozesz go usunac
* optymalizacje
  * dobry programista - signal handler (nie wstawia ifow)
      * gdyby jakims cudem udalo nam sie cos skreszowac, wiec instaluje
      w kernelu signal handler: jak skreszujemy maszyne to ona rzeczywiscie sie
      kreszuje idzie sygnal do kernela ze nastapil kresz procesu a kernel zamiast
      ubijac proces odpala signal hendler a signal hendler rzuca wyjatkiem
      * kosztowny proces, 2 x context switch - z user do jądra i z jądra do usera
  * jak sie to zdarza zbyt czesto - wstawia checki
  * a jak bardzo czesto - to skeszuje wyjatki
  * deref value -SEGV-> signal handler -> throw NPE
1.
	```
	private static void calcLine(int a, int b, int from, int to) {
		Line l = new Line(a, b);
		for (int x = from; x <= to; x++) {
			int y = l.getY(x);
			System.err.println("(" + x + ", " + y + ")");
		}
	}
	
	static class Line {
		public final int a;
		public final int b;
		public Line(int a, int b) {
			this.a = a;
			this.b = b;
		}
	
		// Inlining
		public int getY(int x) {
			return (a * x + b);
		}
	}
	```
	to
	```
	private static void calcLine(int a, int b, int from, int to) {
		Line l = new Line(a, b);
		for (int x = from; x <= to; x++) {
			int y = (l.a * x + l.b);
			System.err.println("(" + x + ", " + y + ")");
		}
	}
	```

## deoptimizations
* deoptimizations
* deoptimization
    * when speculation fails, caught by uncommon trap
    * when CHA (class hierarchy analysis) notices change in class hierarchy
    * when method is no longer "hot", profile traces method frequency invocation
        * bo liczy sie nie tylko ilosc wywolan, ale tez czestotliwosc
* devirtualization, CHA, class hierarchy analysis
    ```
  abstract class AMF {
    abstract double apply(double i);
  }
  
  class CosF / SinF / SqrtF extends AMF {
    double apply(double i) {
        return Math.cos / sin / sqrt (i)
    }
  }
  ```
  oraz kod
  ```
  s d dol(AMF func, double i) {
    return func.apply(i);
  }
  ```
  jeśli classloader zaladuje tylko i wylacznie sinus, to JIT moze spokojnie zrobic
  inlining
  ```
  s d dol(AMF func, double i) {
    return Math.sin(i);
  }
  ```
  classloader laduje cosinus - jit musi zinwalidowac wszystkie funkcje ktore zoptymalizowal
  bazujac na CHA (stop the world); ale to ze classloader zaladowal to nie znaczy ze bedziemy 
  go uzywac, wiec
  ```
  s d dol(AMF func, double i) {
      if (... instanceof SincF)
        return Math.sin(i);
      else uncommon_trap;
  ```
  jesli zaczniemy uzywac cosinusa
  ```
  s d dol(AMF func, double i) {
      if (... instanceof SincF)
        return Math.sin(i);
      else if (... instanceof CosF)
        return Math.cos(i);
      else uncommon_trap;
  ```


## preferences
* jit likes
    * normal code
    * small methods
    * immutability
    * local variables
* jit doesn't like
    * weird code
    * big methods
    * mutability
    * native methods

## tools
* https://github.com/AdoptOpenJDK/jitwatch
    * log analyser and visualiser for the HotSpot JIT compiler.
    