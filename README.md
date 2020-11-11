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

## general
* jit is just complex pattern matchers
    * douglas hawkins
* jit compilers consume transient resources (CPU cycles and memory)
    * from under a milisecond to seonds of compile time, can allocate 100s MBs
    * takes time to get to "full speed" because there may be 1000s of methods to compile
* collecting profile data is an overhead
    * cost usually paid while code is interpreted: slows start-up and ramp-up
* JIT - just in time compiler
* a technique of code compilation during runtime
    * kod kompilujemy do czegos posredniego (bytecode) i potem kod jest kompilowany
    do kodu natywnego w momencie jak aplikacja działa
    * pierwszy raz uzyty w LISP
* opposed to Ahead Of Time compilation: C, C++, Go, Rust
    * kompilujemy kod do kodu maszynowego i w tym czasie mamy mase optymalizacji
* ogromny kawałek kodu w JVM
    * C2 - 19%
    * C1 - 6,3%
    * garbage collector - 15%
* `-Xint` - flaga JVM, tylko kod interpretowany, nie włączają się żadne kompilatory
* tak naprawdę java jest porównywalnie szybka co cpp, tam gdzie jest naprawdę wolna
 i odstaje to jest IO
* czemu mamy dwa kompilatory C1 i C2? bo byly dwa zespoly i nie mogli sie dogadac jak
to trzeba napisac
* JIT optymalizuje 2 rzeczy: metody i pętle
   
* `-XX:+PrintCompilation` - pokazuje moment, w którym JVM zaczyna kompilować kod
    * "uuuuuu czas najwyższy skompilować ten kod"
    * pokazuje czas od start upu do czasu skompilowania metody
    * pokazuje indentyfikator kompilacji (każda kolejna kompilacja zwiększa o 1)
        * c2 ma swój własny indentyfikator kompilacji
        * c1 ma swój własny indentyfikator kompilacji
        * dlatego cyferki moga sie przeplatac
* the loop: interpret -> profile -> compile -> deoptimize -> interpret
* AOT
    * mamy kod i w trakcie compilacji do kodu maszynowego jest on
    optymalizowany
    * restart JVM -> zaczynamy wszystko od nowa, String, LinkedList, HashMap...
        * java 9 ma cos takiego jak AOTC compiler
* hot spots - programming parts that are executed a lot
    * spring configuration classes - only once during bootstrap
    * rest controller methods - many times

## mechanics
* counters
    * hot method: invocation counter > invocation threshold
    * hot loops: backedge (loop) counter > backedge threshold
    * invocation + backedge counter > compile threshold
        * medium hot methods with medium hot loops
    * if counter > threshold -> compile loop
* tiered
    * level 0 = interpreter
    * level 1-3 = c1
    * level 4 = c2
* tiered compilation
    * compilation speed: 
        * C1:Client: Fast
        * C2:Server: Slow(4x)
    * execution speed
        * C1:Client: Slow
        * C2:Server: Fast(2x)

### c1
* fast
* trivial methods, up to 6 bytes (MaxTrivialSize) are inlined by default
    * up to 35 bytes (in bytecode) (MaxInlineSize)
    * limit inliningu: 9 metod wgłąb
* linear registry allocation
* szybki, ale jego kod wynikowy nie jest fantastycznie szybki, 1000x szybszy niz 
interpretowany

### c2
* c2 - profile until 10k calls
* slow but generates faster code (compared to C1)
* by default limited by bytecode size of 325 (FreqInlineSize)
* jesli przekroczymy 8000 bytow bytecodu - JIT nawet sie nia nie zainteresuje i 
bedzie zawsze interpetowana
* uses graph coloring algorithm for registry allocation
* ADL, Architecture Description Language, language in JVM
    * template based
    * przygotowane, gotowe, napisane templaty w kodzie ktore jesli wystapi
    odpowiednia sekwencja w java AST i java AST bedzie mialo odpowiedni ksztalt
    ktory pasuje do kodu to wypluje taki kawalek assemblera
    * jesli twoj kod wyglada w taki sposob to wygeneruj taki kod w assemblerze
* tiered compilation
    * by default since java 8 - uses both compilers for better JVM startup time
    * because JVM suffered from so called warmups, it didn't solve the problem
    just made it less annoying
* w momencie w ktorym JVM stwierdzi, ze ta metoda nadaje sie do kompilacji to wrzuca
ja na kolejke - Compiler Task Queue - a z drugiej strony sa watki ktore zbieraja z kolejki 
i kompiluja (tyle watkow co polowa korow)
    * jak dlugo jakas metoda nie schodzi z kolejki to ja wywali (moze ta metoda juz nie jest wazna)
* compilation attributes
    * %: the compilation is OSR
        * zalozenie jest takie, ze musicie wyjsc z metody zeby ja skompilowac
        bo nie mozemy podmienic tego kodu w trakcie dzialania metody
            * mozemy dzieki OSR
        * petle w watkach - while(true)
        * OSR is useful in situations where you identify a function as "hot" while it is running. 
        This might not necessarily be because the function gets called frequently; it might be 
        called only once, but it spends a lot of time in a big loop which could benefit from optimization. 
        When OSR occurs, the VM is paused, and the stack frame for the target function is replaced by an 
        equivalent frame which may have variables in different locations.
    * s: the method is synchronized
    * !: the method has an exception handler
    * n: byla juz natywna    
* JIT not only compiles hot methods but also optimizes hot paths so it speculates
which part of your code is actually executed
    * jesli mamy jakiegos elsa, do ktorego nie wchodzimy to go nie skompiluje, ale zostanie
    tam wlozony uncommon trap
        * i wtedy made not entrant - JIT mowi nie wchodze juz do skompilowanego 
        kodu, zaczelo sie zachowywac inaczej
        * uncommon trap - jak watek tam wdepnie to jest wywlaszczany do interpretera
        a JIT dostaje informacje, ze sie pomylil
    * chodzi o to, zeby czas kompilacji nie dominowal czasu aplikacji
* made not entrant, made zombie
    * made zombie - metoda była wcześniej made not entrant, ale już żaden
    wątek nie ma jej na stacku więc można ją wyrzucić do kosza
    * made not entrant
        * może istniec wiele kompilacji metody jednoczesnie, jak metodaA chce
        zawolac metodeB to musi sie dowiedziec ktora kompilacja jest najlepsza
        * jesli metodaA chce uzyc po raz pierwszy metodyB - pyta sie Call dispatcher - daj mi kompilacje najlepsza
            * metodaA cacheuje te informacje
        * i przy nastepnym wywolaniu mamy dwie opcje
            * skeszowana kompilacja jest juz made no entrant, wiec metodaA widzi ze
            nie moze jej uzyc - pyta dispatchera
            * jest juz lepsza kompilacja - wtedy przez chwile mamy w pamieci 2 kompilacje
            nic sie zlego nie dzieje, metoda bedzie sie uruchamiac po prostu wolniej; w miedzyczasie
            dispatcher oznacza wolniejsza kompilacje jako made not entrant i przekierowuje na 
            nowa, szybsza 
  

## optimizations
* speculative optimizations
* golden rule of optimization: don't do unnecessary work
* optimization
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
                for ... {
                    process(option)
                }
            }
            private synchronized String process(String option) { }  
            ```
            compiled into
            ```
            public void needsLocks() {
                synchronized(this) {
                    for ... {
                        // inlined
                    }
                }
            }
            ```
        * eliding
            ```
            p v xxx() {
                var l = new ArrayList();
                synchronized(l) { // lock on local variable
                    for ... l.add(...) // l never escapes this thread
                }
            }
            ```
            compiled into
            ```
            public void xxx() {
                var l = new ArrayList();
                for ... l.add(...)
            }
            ```
    * dead code elimination
    * duplicate code elimination
    * escape analysis
        ```
        p v x() {
            var f = new Foo("X", "Y");
            baz(f)
        }
      
        p v baz(Foo f) {
            sout(f.a)
            sout(",")
            quux(f)
        }
        
        p v quux(Foo f) {
            sout(f.b)
            sout("!")
        }
        ```
        compiled into
        ```
        p v inlined() { // don't bother allocating foo object
        sout("X")
        sout(",")
        sout("Y")
        sout("!")
        }
        ```
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
    