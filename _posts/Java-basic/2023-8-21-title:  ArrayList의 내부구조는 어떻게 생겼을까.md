---
title:  "ArrayList의 내부구조는 어떻게 생겼을까?"

categories:
  - Java
tags:
  - Java

toc: true
toc_sticky: true

date: 2023-08-21
last_modified_at: 2023-08-21
---

# ArrayList

내부적으로 특정한 크기의 배열을 가지고 있다.

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
    private static final int DEFAULT_CAPACITY = 10; // 기본 사이즈는 10
    transient Object[] elementData; // 데이터가 저장되는 배열
    private int size; // 배열의 사이즈

    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
}
```

## 배열이 꽉 찼을 때 값을 추가하면 어떻게 될까?

배열은 크기는 고정되어 있다.
따라서 새로운 배열을 만들고, 그 배열에 기존의 배열의 값을 복사한다.

```java
    public boolean add(E e) {
        modCount++;
        add(e, elementData, size); // 값 추가
        return true;
    }
```

```java
private void add(E e, Object[] elementData, int s) {
        // 기존의 배열이 크기가 차있다면
        if (s == elementData.length)
            elementData = grow(); // 배열의 크기 증가
        elementData[s] = e;
        size = s + 1;
    }
```

```java
    private Object[] grow() {
        return grow(size + 1);
    }
```

```java
    private Object[] grow(int minCapacity) {
        int oldCapacity = elementData.length;
        if (oldCapacity > 0 || elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            // 새로운 배열의 크기 = 기존의 배열의 크기 + Math.max(기존의 배열의 크기 / 2, 1)
            int newCapacity = ArraysSupport.newLength(oldCapacity, // 기존의 배열의 크기
                    minCapacity - oldCapacity, // 크기 1
                    oldCapacity >> 1); // 기존 배열의 크기의 절반

            // 새로운 배열에 기존의 배열을 복사
            return elementData = Arrays.copyOf(elementData, newCapacity);
        } else {
            return elementData = new Object[Math.max(DEFAULT_CAPACITY, minCapacity)];
        }
    }
```

결과적으로 새로운 배열의 크기는 `(3/2) * 기존의 배열의 크기`가 됨을 알 수 있습니다. (새로운 배열의 크기가 오버플로우가 발생하지 않을 경우)

## StringBuilder도 배열이 꽉 찰 때도, ArrayList처럼 동작할까?

StringBuilder도 ArrayList와 유사하게 내부적으로 배열을 가지고 있다.  
배열이 가득 차 있는 상태에서 .append() 메서드를 호출할 때, ArrayList와 동일하게 동작할까?

> 아래의 코드에 나올 용어 정리
>
> length : 배열의 크기
> capacity : 배열에 저장될 수 있는 문자의 개수 
> -> 하나의 문자열을 저장할 때 사용하는 byte의 크기는 다를 수 있기 때문에 capacity 용어 존재

```java
// StringBuilder가 상속 받고 있는 클래스
abstract sealed class AbstractStringBuilder implements Appendable, CharSequence
    permits StringBuilder, StringBuffer {

    byte[] value; // 문자열이 담기는 배열
    int count; // 배열의 사이즈

    // 생략
```

```java 
public AbstractStringBuilder append(String str) {
        if (str == null) {
            return appendNull();
        }
        int len = str.length();
        // 배열의 사이즈가 부족하다면 늘려준다.
        ensureCapacityInternal(count + len);
        putStringAt(count, str);
        count += len;
        return this;
    }
```

```java
private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        int oldCapacity = value.length >> coder;
        if (minimumCapacity - oldCapacity > 0) {
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity) << coder);
        }
    }
```

```java
private int newCapacity(int minCapacity) {
        int oldLength = value.length;
        int newLength = minCapacity << coder;
        int growth = newLength - oldLength;

        // overflow를 고려하여 새로운 배열의 길이 계산 
        int length = ArraysSupport.newLength(oldLength, growth, oldLength + (2 << coder));
        if (length == Integer.MAX_VALUE) {
            throw new OutOfMemoryError("Required length exceeds implementation limit");
        }

        // 새로운 문자열의 길이 반환 (배열의 길이가 아님)
        return length >> coder;
    }
```

```java
public static int newLength(int oldLength, int minGrowth, int prefGrowth) {

        // overflow가 발생하지 않는다면
        int prefLength = oldLength + Math.max(minGrowth, prefGrowth);
        if (0 < prefLength && prefLength <= SOFT_MAX_ARRAY_LENGTH) {
            return prefLength;
        } else {
            // put code cold in a separate method
            return hugeLength(oldLength, minGrowth);
        }
    }
```

위의 코드를 봤을 때, 기존의 배열의 사이즈보다 `약 2배` 더 커지는 것을 알 수 있다.

> 왜 ArrayList 랑 StringBuilder 에서 배열의 사이즈를 증가시키는 방법이 다를까?
