# 16. CompletableFuture: composable asynchronous programming

1. Simple use of Futures
2. Implementing an asynchronous API
3. Making your code nonblocking
4. Pipelining asynchronous tasks
5. Reacting to a CompletableFuture completion
6. Read map
7. Summary

> ### This chapter covers
>
> - 비동기 연산을 만드는 방법과 결과 반환
> - nonblocking 연산을 만들어 생산량 향상
> - 비동기 API 설계, 구현
> - 동기 API를 비동기적으로 사용
> - 2개 이상의 비동기 연산 pipelining, merging
> - 비동기 연산 결과에 react

---

## 1. Simple use of Futures

- `java.util.concurrent.Future` : 미래 특정 시점에 result를 반환하는 연산
- request 즉시 result를 반환하지 않음
- 비동기 연산 설계, 연산 수행 result에 대한 참조 제공
- 시간이 걸리는 연산을 `Future`에게 위임하고, 다른 작업 수행
    - e.g. 세탁기를 돌리는 동안 다른 일을 함

````
// Java 8 이전
ExecutorService executor = Executors.newCachedThreadPool();

// 비동기 연산 시작
Future<Double> future = executor.submit(new Callable<Double>() {
    @Override
    public Double call() throws Exception {
        return doSomeLongComputation(); // 시간이 걸리는 연산
    }
});

doSomethingElse(); // 다른 작업 수행

try {
  Double result = future.get(1, TimeUnit.SECONDS); // 연산 결과를 가져옴 (blocking)
} catch (ExecutionException e) {
    // 계산 중 예외 발생
} catch (InterruptedException e) {
    // 현재 스레드에서 대기 중 인터럽트 발생
} catch (TimeoutException e) {
    // TimeUnit.SECONDS 동안 결과가 없음
}
````

<img src="img.png"  width="70%"/>

### 1.1 Understanding Futures and their limitations

#### 복잡한 비동기 연산 요구사항 (`Future`로 해결 불가)

- 2가지 비동기 연산이 서로 독립적이거나 아닐 수 있음
- `Future`s 연산들이 모두 완료되기를 기다림
- `Future`s 연산들 중 가장 빨리 완료되는 연산만을 기다림
- 비동기 연산 결과를 수동으로 반환하여 `Future`를 완료시킴
- `Future` 연산 결과에 react
    - 완료되면 알림을 받고 추가 action
- `Future`의 구현체 `CompletableFuture`는 이러한 요구사항을 declarative하게 표현, 지원

### 1.2 Using CompletableFuture to build an asynchronous application

#### 구현 예시 : 최저가 찾기 Application

- 사용자에게 비동기 API를 제공하는 방법
- 동기 API 사용자일때 nonblocking code를 만드는 방법
    - 2개의 동기 연산을 pipelining 하고 결과를 하나의 동기 연산으로 merge
- 비동기 연산이 완료되었을 때 reactive하게 동작
    - e.g. 상품별 최저가를 지속적으로 update

#### Synchronous vs asynchronous API

- synchronous API (_blocking_ call)
    - caller는 result를 기다림
    - caller 와 callee가 다른 threa에 있어도, caller는 기다림
- asynchronous API (_nonblocking_ call)
    - 계산이 완료되기 전에 return (남은 계산은 thread에게 위임)
    - callback method를 통해 caller와 소통
    - I/O system programmion에 적합

## 2. Implementing an asynchronous API

````java
public class Shop {

    public double getPrice(String product) {
        return calculatePrice(product);
    }

    private double calculatePrice(String product) {
        delay(); // 1초간 delay
        return new Random().nextDouble() * product.charAt(0) + product.charAt(1);
    }
}

````

- 사용자가 `getPrice()`를 호출하면, blocking되어 1초 이상 소요 후 결과 반환
- 100개의 상품을 조회하면 100초 이상 소요

### 2.1 Converting a synchronous method into an asynchronous one

```
// public double getPrice(String product) { .. .}
public Future<Double> getPriceAsync(String product) { ... }
```

````
public Future<Double> getPriceAsync(String product) {
    CompletableFuture<Double> futurePrice = new CompletableFuture<>();
    new Thread(() -> {
        double price = calculatePrice(product);
        futurePrice.complete(price);
    }).start();
    return futurePrice;
}
````

- `getPrice()`를 비동기로 만듦

```

@Test
public void tst1() throws ExecutionException, InterruptedException {
    Shopp shop = new Shopp("Best Shop");

    long start = System.nanoTime();
    Future<Double> futurePrice = shop.getPriceAsync("IPhone 12"); // 즉시 return & 비동기 연산 시작
    long invocationTime = ((System.nanoTime() - start) / 1_000_000);
    System.out.println("Invocation returned after " + invocationTime + " msecs");

    double price = futurePrice.get(); // blocking (연산이 완료될 때까지 기다림)
    System.out.printf("Price is %.2f%n", price);

    long retrievalTime = ((System.nanoTime() - start) / 1_000_000);
    System.out.println("Price returned after " + retrievalTime + " msecs");
}

```

<details>
<summary>실행 결과</summary>

```bash
Invocation returned after 0 msecs
Price is 112.66
Price returned after 1020 msecs
```

</details>

### 2.2 Dealing with errors

```
public Future<Double> getPriceAsync(String product) {
    CompletableFuture<Double> futurePrice = new CompletableFuture<>();
        new Thread( () -> {
            try {
                double price = calculatePrice(product);
                futurePrice.complete(price);
            } catch (Exception ex) {
                futurePrice.completeExceptionally(ex); // 에러 발생시 exception을 포함하여 complete
            }
        }).start();
    return futurePrice;
}
```

- `completeExceptionally()` : `CompletableFuture`에 에러를 포함하여 complete
- client는 `get()`을 호출할 때 `ExecutionException`을 받음

<details>
<summary>예외 발생시 console</summary>

```bash
java.util.concurrent.ExecutionException: java.lang.RuntimeException: [에러 메시지 예시]
  at java.base/java.util.concurrent.CompletableFuture.reportGet(CompletableFuture.java:396)
  at java.base/java.util.concurrent.CompletableFuture.get(CompletableFuture.java:2073)
...
```

</details>

#### CREATING A COMPLETABLEFUTURE WITH THE SUPPLYASYNC FACTORY METHOD

````
public Future<Double> getPriceAsync(String product) {
    return CompletableFuture.supplyAsync(() -> calculatePrice(product));
}
````

- `supplyAsync()` : `Supplier`를 인수로 받아 `CompletableFuture`를 rturn
- 비동기로 Supplier를 실행
- `Supplier`는 ForkJoinPool의 `Executor`를 사용하여 실행
- 예외처리도 동일하게 구현되어있음

## 3. Making your code nonblocking

````
private List<Shop> shops = List.of(new Shop("11번가"), new Shop("G마켓"), new Shop("옥션"), new Shop("쿠팡"), new Shop("위메프"));
public List<String> findPrices(String product) {
    return shops.stream()
            .map(shop -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product)))
            .collect(toList());
}

@Test
public void tst2() {
    long start = System.nanoTime();
    System.out.println(findPrices("Aespa - MY WORLD (정규 3집)"));
    long duration = (System.nanoTime() - start) / 1_000_000;
    System.out.println("Done in " + duration + " msecs");
}
````

<details>
<summary>실행 결과</summary>

```bash
[11번가 price is 156.03 
, G마켓 price is 131.27 
, 옥션 price is 142.56 
, 쿠팡 price is 133.57 
, 위메프 price is 101.21 
]
Done in 5044 msecs
```

</details>
- `calculatePrice()`를 동기적으로 실행
- 5초 이상 (shop 수 만큼) 소요

### 3.1 Parallelizing requests using a parallel Stream

````
public List<String> findPrices(String product) {
    return shops.parallelStream()
            .map(shop -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product)))
            .collect(toList());
}

````

- `Done in 1038 msecs`

### 3.2 Making asynchronous requests with CompletableFuture

````
public List<String> findPrices(String product) {

    List<CompletableFuture<String>> priceFutures = shops.stream()
            .map(shop -> CompletableFuture.supplyAsync(() -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product))))
            .collect(toList())
            .stream()
            // .map(CompletableFuture::join) // join을 사용하면 supplyAsync() 완료될 때까지 블록킹
            .collect(toList());

    return priceFutures.stream()
            .map(CompletableFuture::join)
            .collect(toList());
}
````

<img src="img_1.png"  width="70%"/>

### 3.3 Looking for the solution that scales better

#### thread 수에 따른 성능 변화 (thread 5개 가정)

| shop | synchrnous | parallelStream | CompletableFuture |
|------|------------|----------------|-------------------|
| 2    | 2 sec      | 1 sec ~        | 1 sec ~           |
| 5    | 5 sec      | 1 sec ~        | 1 sec ~           |
| 6    | 6sec       | 2 sec ~        | 2 sec ~           |

- parallelStream, `CompletableFuture` 모두 `Runtime.getRuntime().availableProcessors()` 수만큼 thread를 사용
- `CompletableFuture`은 직접 executor를 지정 할 수 있음

### 3.4 Using a custom Executor

> #### thread pool 사이즈 정하는 공식 (book _Java Concurrency in Practice_ Addison-Wesley, 2006)
>
> N<sup>threads</sup> = N<sup>CPU</sup> * U<sub>CPU</sub> * (1 + W/C)
>
> - N<sup>CPU</sup> : CPU 수 (코어 수, `Runtime.getRuntime().availableProcessors()`)
> - U<sub>CPU</sub> : CPU 사용률 (0 ~ 1)
> - W/C : wait time / compute time
> - e.g. 4 core | 100% CPU 사용 | 대기 50ms, 계산 5ms
> - N<sup>threads</sup> = 4 * 1 * (1 + 50/5) = **44**

````
private final Executor customExecutor 
  = Executors.newFixedThreadPool(Math.min(shops.size(), 44) // shop size < thread pool size < 44
    , (Runnable r) -> {
        Thread t = new Thread(r);
        t.setDaemon(true); // daemon thread : main thread가 종료되면 함께 종료되는 thread
        return t;
    });
    
...

CompletableFuture.supplyAsync(() -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product))
                    , customExecutor); // custom executor 지정

````

##### Parallelism: via Streams or CompletableFutures?

|     | Stream                              | CompletableFuture                          |
|-----|-------------------------------------|--------------------------------------------|
| 사용처 | I/O 없는 무거운 계산                       | I/O waiting이 포함된 연산                        |
| 특징  | Stream API 사용, 간단한 구현               | thread pool 사이즈 지정 가능                      |
| 단점  | I/O 가 있으면 Stream의 lazy로 인해 디버깅이 어려움 | 코드가 길어짐, hardware 의존적 (thread pool 사이즈 지정) |

## 4. Pipelining asynchronous tasks

<details>
<summary>Enum : Discount.Code</summary>

````
// return [shopName]:[price]:[DiscountCode]
public String getPrice(String product) {
    double price = calculatePrice(product);
    Discount.Code code = Discount.Code.values()[new Random().nextInt(Discount.Code.values().length)];
    return String.format("%s:%.2f:%s", name, price, code);
}

````

```java
public class Discount {
    public enum Code {
        NONE(0), SILVER(5), GOLD(10), PLATINUM(15), DIAMOND(20);
        private final int percentage;

        Code(int percentage) {
            this.percentage = percentage;
        }
    }
}
```

</details>

### 4.1 Implementing a discount service

```java
// 가격 정보 문자열을 파싱해서 캡슐화
public class Quote {
    private final String shopName;
    private final double price;
    private final Discount.Code discountCode;

    public Quote(String shopName, double price, Discount.Code code) {
        this.shopName = shopName;
        this.price = price;
        this.discountCode = code;
    }

    // 문자열을 파싱해서 Quote 객체를 만드는 factory 메서드
    public static Quote parse(String s) {
        String[] split = s.split(":");
        String shopName = split[0]; // 상점명
        double price = Double.parseDouble(split[1]); // 가격
        Discount.Code discountCode = Discount.Code.valueOf(split[2]); // 할인 코드
        return new Quote(shopName, price, discountCode); // Quote 객체 생성
    }

    public String getShopName() {
        return shopName;
    }

    public double getPrice() {
        return price;
    }

    public Discount.Code getDiscountCode() {
        return discountCode;
    }
}

public class Discount {
    public enum Code {
        // ...
    }

    // 할인 적용
    public static String applyDiscount(Quote quote) {
        return quote.getShopName() + " price is " +
                Discount.apply(quote.getPrice(),
                        quote.getDiscountCode());
    }

    // 가격 계산
    private static double apply(double price, Code code) {
        delay();
        return Math.round(price * (100 - code.percentage) / 100);
    }
}
```

### 4.2 Using the Discount service

````
public List<String> findPrices(String product) {
    return shops.stream()
            .map(shop -> shop.getPrice(product)) // return Stream<String> [shopName]:[price]:[DiscountCode]
            .map(Quote::parse) // return Stream<Quote>
            .map(Discount::applyDiscount)// return Stream<String> [shopName] price is [price]
            .collect(toList());
}

@Test
public void tst2() {
    long start = System.nanoTime();
    System.out.println(findPrices("Aespa - MY WORLD (정규 3집)"));
    long duration = (System.nanoTime() - start) / 1_000_000;
    System.out.println("Done in " + duration + " msecs");
}

````

<details>
<summary>실행 결과</summary>

```bash
[11번가 price is 131.0, G마켓 price is 98.0, 옥션 price is 151.0, 쿠팡 price is 104.0, 위메프 price is 145.0]
Done in 10051 msecs
```

</details>

- 10 sec 이상 소요
- 5초 (`getPrice()`) + 5초 (`applyDiscount()`) + @

### 4.3 Composing synchronous and asynchronous operations

````
public List<String> findPricesAsync(String product) {

  List<CompletableFuture<String>> priceFutures = shops.stream() // return Stream<Shop>
    .map(shop -> CompletableFuture.supplyAsync(() -> 
        shop.getPrice(product), customExecutor))// return Stream<CompletableFuture<String>>, async
    .map(future -> future.thenApply(Quote::parse))// return Stream<CompletableFuture<Quote>>, sync
    .map(future -> future.thenCompose(quote ->
            CompletableFuture.supplyAsync(() ->
                    Discount.applyDiscount(quote), customExecutor))) // return Stream<CompletableFuture<String>>, async
    .collect(toList());

  return priceFutures.stream()
    .map(CompletableFuture::join) // return Stream<String>
    .collect(toList());
    
}
````

<img src="img_2.png"  width="70%"/>

#### GETTING THE PRICES `getPrice()`

- _asynchronous_ operation
- `supplyAsync()` factory method 이용
- return `Stream<CompletableFuture<String>>`

#### PARSING THE QUOTES `Quote::parse`

- _synchronous_ operation
- `delay()`가 없기 때문에 동기로 실행
- `thenApply()` 이용
- **주의 : `thenApply()`는 block되지 않음. 앞선 `map()` 과정이 끝나야 실행 시작**

#### COMPOSING THE FUTURES FOR CALCULATING THE DISCOUNTED PRICE `Discount::applyDiscount`

- _asynchronous_ operation
- `thenCompose()` : 2개의 async 연산을 pipeline
    - 첫번째 async 연산이 완료되면 두번째 async 연산 실행
- 첫번째 async 연산 : 가격 parasing + `Quote` 캡슐화 (`map 1 ~ 2`)
- 두번째 async 연산 : 할인 적용 (`map 3`)
- `thenComposeAsync()` : 첫번째 연산과 다른 thread에서 두번째 연산 실행
    - **_Spring_ 처럼 직접 thread pool을 관리하는 경우 같은 thread에서 실행될 수 있음 주의**

#### 실행 결과

```bash
[11번가 price is 136.0, G마켓 price is 97.0, 옥션 price is 136.0, 쿠팡 price is 136.0, 위메프 price is 152.0]
Done in 2035 msecs
```

### 4.4 Combining two CompletableFuture dependent and independent

- 2개의 `CompletableFutures`가 서로 독립적일 때 (비동기로 실행 하고 싶을 때)
- `thenCombine()` : 2번째 인자로 `java.util.function.BiFunction`을 받음
    - `BiFunciton`이 어떻게 두가지를 merge할 지 정의
- `thenCombineAsync()` : `thenCombine()`과 동일하게 동작하나, `BiFunction`을 다른 thread에서 실행

````
Future<Double> futurePriceInUSD =
    CompletableFuture.supplyAsync(() -> shop.getPrice(product)) // 첫번쨰 비동기 연산, return CompletableFuture<String>
        .thenCombine(CompletableFuture.supplyAsync(() -> 
            exchangeService.getRate(Money.EUR, Money.USD)) // 두번째 비동기 연산, return CompletableFuture<Double>
                , (price, rate) -> price * rate)); // 두 연산의 결과를 merge하는 BiFunction
````

<img src="img_3.png"  width="70%"/>

### 4.5 Reflecting on Future vs CompletableFuture

````
ExecutorService executor = Executors.newCachedThreadPool();

// Java 7 이전 : Future
final Future<Double> futureRate = executor.submit(new Callable<Double>() {
    public Double call() {
       return exchangeService.getRate(Money.EUR, Money.USD);
    }
});

Future<Double> futurePriceInUSD = executor.submit(new Callable<Double>() {
    public Double call() {
        double priceInEUR = shop.getPrice(product);
        return priceInEUR * futureRate.get();
    }
});

// Java 8 : CompletableFuture
CompletableFuture<Double> futurePriceInUSD = CompletableFuture.supplyAsync(() -> shop.getPrice(product))
    .thenCombine(CompletableFuture.supplyAsync(() -> 
        exchangeService.getRate(Money.EUR, Money.USD))
            , (price, rate) -> price * rate);
````

### 4.6 Using timeouts effectively

- 무한 blocking을 방지하기 위해 **항상** timout 설정
- _Java 9_ `orTimeout()` : `ScheduledThreadExecutor`을 사용해서 timeout 설정
    - `CompletableFuture`가 완료되지 않으면 `TimeoutException` 발생하고, 새로운 `CompletableFuture`를 반환
- `completeOnTimeout()` : `CompletableFuture`가 완료되지 않으면 기본 값을 사용하고, `CompletableFuture`를 반환

````
CompletableFuture<Double> futurePriceInUSD = CompletableFuture.supplyAsync(() -> shop.getPrice(product))
    .thenCombine(CompletableFuture.supplyAsync(() -> 
        exchangeService.getRate(Money.EUR, Money.USD))
            , (price, rate) -> price * rate)
    .orTimeout(3, TimeUnit.SECONDS); // 3초 이상 걸리면 TimeoutException 발생

// Execption 없이 기본 값을 반환하고 싶을 때
CompletableFuture<Double> futurePriceInUSD = CompletableFuture.supplyAsync(() -> shop.getPrice(product))
    .thenCombine(CompletableFuture.supplyAsync(() -> 
        exchangeService.getRate(Money.EUR, Money.USD))
            .completeOnTimeout(DEFAULT_RATE, 1, TimeUnit.SECONDS) // 1초 이상 걸리면 DEFAULT_RATE 반환
        , (price, rate) -> price * rate)
````

## 5. Reacting to a CompletableFuture completion

````
// 실제 상황에선 딜레이 시간이 램덤
public static void delay() {
    int delay = 500 + new Random().nextInt(2000);
    try {
        Thread.sleep(delay);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
}
````

- 모든 가격이 다 구해면 응답하지 않고, 가장 빨리 구해진 가격을 응답하고 싶음

### 5.1 Refactoring the best-price-finder application

````
@Test
@DisplayName("비동기 출력")
public void tst3() {
    CompletableFuture[] futures =
            findPricesStream("Aespa - MY WORLD (정규 3집)")
                    .map(future -> future.thenAccept(System.out::println)) //return Stream<CompletableFuture<Void>>
                    .toArray(size -> new CompletableFuture[size]);

    CompletableFuture.allOf(futures).join();
    // CompletableFuture.anyOf(futures).join(); // 가장 먼저 응답한 가격 하나 출력
}

public Stream<CompletableFuture<String>> findPricesStream(String product) {
    return shops.stream()
        .map(shop ->
                CompletableFuture.supplyAsync(() -> 
                    shop.getPrice(product), customExecutor)) // return Stream<CompletableFuture<String>>, async
        .map(future ->
                future.thenApply(Quote::parse)) // return Stream<CompletableFuture<Quote>>, sync
        .map(future ->
                future.thenCompose(quote -> 
                    CompletableFuture.supplyAsync(() -> 
                        Discount.applyDiscount(quote), customExecutor))); // return Stream<CompletableFuture<String>>, async
}

````

- `thenAccept()`
    - return `CompletableFuture<Void>`
- `thenAcceptAsync()` : 인자 `Consumer`를 다른 thread에서 실행
- `allOf()` : 인자 `CompletableFuture[]`를 받아 모든 `CompletableFuture`가 완료되면 `CompletableFuture<Void>`를 반환
    - `anyOf()` : 인자 `CompletableFuture[]`를 받아 하나의 `CompletableFuture`가 완료되면 `CompletableFuture<Object>`를 반환
- `join()` : 실행, `CompletableFuture`가 완료될 때까지 기다림

### 5.2 putting it all together

````
@Test
@DisplayName("최종")
public void tst4() {
    long start = System.nanoTime();
    CompletableFuture[] futures = findPricesStream("Aespa - MY WORLD (정규 3집)")
            .map(f -> f.thenAccept(s ->
                    System.out.println(s + " (done in " + ((System.nanoTime() - start) / 1_000_000) + " msecs)")))
            .toArray(size -> new CompletableFuture[size]);

    CompletableFuture.allOf(futures).join();
    System.out.println("All shops have now responded in " + ((System.nanoTime() - start) / 1_000_000) + " msecs");
}
````

<details>
<summary>실행 결과</summary>

```bash
G마켓 price is 116.0 (done in 2315 msecs)
위메프 price is 145.0 (done in 2470 msecs)
옥션 price is 132.0 (done in 3104 msecs)
11번가 price is 129.0 (done in 3273 msecs)
쿠팡 price is 127.0 (done in 3803 msecs)
All shops have now responded in 3803 msecs
```

</details>

## 6. Read map

- Chapter 17. Java 9 Flow API : `CompletableFuture`를 일반화
    - 연산 후 선택적으로 종료할 수 있음

## 7. Summary

- 긴시간이 걸리는 연산에 비동기 작업을 적용하면 성능, 응답성이 향상됨
    - 특히 remote service 간의 통신인 경우
- client에게 비동기 API를 제공하는 방법 : `java.util.concurrent.CompletableFuture`
- `CompletableFuture`는 비동기 작업에서 발생한 에러를 처리하는 방법을 제공 (전파 or 처리)
- 동기 API를 `CompletableFuture`로 wrapping하여 비동기로 만들 수 있음
- 2개 이상의 비동기 연산을 pipelining 하고 결과를 merge할 수 있음
- `CompletableFuture`에 callback을 등록할 수 있음
- `CompletableFuture` 들이 모두 완료되야 응답하거나, 가장 빨리 완료된 연산의 결과를 사용할 수 있음
- `CompletableFuture`의 `orTimeout()`, `completeOnTimeout()`을 사용하여 timeout 설정 가능 (Java 9)