# 8. Collection API enhancements (Java 9)

1. Collection factories
2. Working with List and Set
3. Working with Map
4. Improved ConcurrentHashMap
5. Summary

> ### This chapter covers
>
> - collection factory 사용
> - List, Set을 사용한 새로운 idomatic pattern
> - Map을 사용한 idomatic pattern

---

- Java 9에 추가된 Collection factory
    - 작은 list, set, map을 생성하는데 사용
- idomatic removal / replacement pattern
- map을 편리하게 사용하는 방법

##                     

````
List<String> aespa = new ArrayList<>();
aespa.add("Karina");
aespa.add("Winter");
aespa.add("Giselle");
aespa.add("Ningning");

// Arrays.asList()를 사용하여 더 간단하게 작성
List<String> aespa = Arrays.asList("Karina", "Winter", "Giselle", "Ningning");

aespa.set(0, "KARINA"); 
aeps.add("newMember"); // thorws UnsupportedOperationException

Set<String> ive = new HashSet<>(Arrays.asList("Yujin", "Wonyoung"));
ive.add("newMember"); // mutalbe
````

- `Arrays.asList()` : 고정된 사이즈의 list를 생성
    - `add()`, `remove()` 등 사용할 수 없음
    - `set()` 사용 가능
- Set은 mutable

### 1.1 List factory

````
List<String> aespa = List.of("Karina", "Winter", "Giselle", "Ningning");
aespa.add("newMember"); // throws UnsupportedOperationException
````

- `List.of()` : immutable list 생성
    - `add()`, `remove()`, `set()` 시 예외 발생

#### Overloading vs varargs

```java

package java.util;

public interface List<E> extends Collection<E> {
    // ...
    static <E> List<E> of(E e1) {
        // ...
    }

    static <E> List<E> of(E e1, E e2) {
        // ...
    }

    //...

    static <E> List<E> of(E... elements) {
        return new ImmutableCollections.ListN<>(elements);
    }

}

````

- `List.of()`, `Set.of()`, `Map.of()`는 varargs, overloading 둘 다 사용
- 10개의 element까지는 각각 overloading된 method를 사용
    - array 할당, 초기화, GC 비용을 줄이기 위함

### 1.2 Set factory

````
Set<String> aespa = Set.of("Karina", "Winter", "Giselle", "Ningning");
aespaSet.add("newMember"); // UnsupportedOperationException

Set<String> aespaSet = Set.of("Karina", "Winter", "Giselle", "Ningning", "Karina");// IllegalArgumentException
````

### 1.3 Map factory

````
Map<String, Integer> aespaAge1 = Map.of("Karina", 21, "Winter", 21, "Giselle", 20, "Ningning", 18);

aespaAge1.put("새로운 멤버", 22); // UnsupportedOperationException
aespaAge1.put("Karina", 22); // UnsupportedOperationException

Map<String, Integer> aespaAge2 = Map.ofEntries(
        Map.entry("Karina", 21),
        Map.entry("Winter", 21),
        Map.entry("Giselle", 20),
        Map.entry("Ningning", 18)
);
````

- `java.util.Map.entry()` : `Map.ofEntries()`를 사용하기 위한 helper method
    - Map element가 많을 때 유용

## 2. Working with List and Set

Java 8에서 Colelction 스스로 수정하는 method를 추가

- `rmoveIf()` : predicate를 사용하여 element를 제거
    - `List`, `Set`을 구현한 모든 class에서 사용 가능
- `replaceAll()` : `UnaryOperator`를 사용하여 element를 대체
- `sort()` : list를 정렬
    - `List`를 구현한 모든 class에서 사용 가능

### 2.1 removeIf

````
List<String> memberList = new ArrayList<>();
memberList.add("Karina");
memberList.add("Giselle");
memberList.add("Winter");
memberList.add("Ningning");

// ConcurrentModificationException
for (String memberName : memberList) {
    if (memberName.equals("Karina")) {
        memberList.remove(memberName);
    }
}

// ConcurrentModificationException : 내부적으로 아래와 같이 동작
for (Iterator<String> iterator = memberList.iterator();
     iterator.hasNext(); ) {
    String memberName = iterator.next();
    if (memberName.equals("Karina")) {
        memberList.remove("Karina");
    }
}

// Iterator로 제거
for (Iterator<String> iterator = memberList.iterator();
     iterator.hasNext(); ) {
    String memberName = iterator.next();
    if (memberName.equals("Karina")) {
        iterator.remove();
    }
}

// Java 8
memberList.removeIf(memberName -> memberName.equals("Karina"));
````

- `for ~ loop` 은 내부적으로 `Iterator`를 사용
- `ConcurrentModificationException` 발생
    - Iterator를 사용하면서, `List` object를 통해 수정하려하기 때문
- `Iterator`를 사용하여 제거하면 해결
- Java 8 부터는 `removeIf()`를 사용하여 간단하게 제거 가능

### 2.2 replaceAll

````
// 새로운 collection을 생성
memberList.stream().map(memberName -> "original : " + memberName + " | upper : " + memberName.toUpperCase())
        .collect(Collectors.toList())
        .forEach(System.out::println);
````

```bash
original : Karina | upper : KARINA
original : Giselle | upper : GISELLE
original : Winter | upper : WINTER
original : Ningning | upper : NINGNING
```

````
// 기존 collection을 수정
for (ListIterator<String> iterator = memberList.listIterator();
 iterator.hasNext(); ) {
    String memberName = iterator.next();
    iterator.set("original : " + memberName + " | upper : " + memberName.toUpperCase());
}

// Java 8
memberList.replaceAll(memberName -> "original : " + memberName + " | upper : " + memberName.toUpperCase());
````

## 3. Working with Map

## 4. Improved ConcurrentHashMap

## 5. Summary