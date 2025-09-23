# <u> Important Kotlin Concepts </u>

## <u> Collections & Sequences </u>:

### Mapping Collections: `associate` vs `associateWith` vs `associateBy`

Kotlin provides powerful functions to transform collections into maps. Commonly used functions are `associate`, `associateWith` and `associateBy`. Understanding the differences between these functions is crucial for effective data manipulation:

- **`associate`**: Transforms a collection into a map by applying a function to each element to produce a key-value pair (`Pair<K, V>`).
  
  ```kotlin
  val words = listOf("apple", "banana", "cherry")
  val map = words.associate { it to it.length }
  // Result: {apple=5, banana=6, cherry=6}
  
  val orderEntries = checkoutResponse.orderLines
    ?.associate { it.lineNumber to fromOrderLine(it) }
    .orEmpty()
   // Result: {1=OrderLine(...), 2=OrderLine(...), ...}
  
  // Manually build a Pair(key, value) using to.
  // Flexible, but a little noisier.
  ```
- **`associateBy`**: is a collection function that builds a map from a list (or any iterable).

  - You tell it how to choose a key from each element (the keySelector). 
  - Optionally, you tell it how to transform the element into a value (the valueTransform). 
  - The result is a map where each key comes from the keySelector and each value is either:
    - the original element (if you don’t provide a value transform), or 
    - the transformed result (if you do provide a value transform). 
  - If two elements produce the same key, the last one wins and overwrites earlier ones.

```kotlin
    public inline fun <T, K, V> Iterable<T>.associateBy(
        keySelector: (T) -> K,
        valueTransform: (T) -> V
    ): Map<K, V>
    // Iterable<T> → You call it on a collection of type T (e.g., List<OrderLine>).
    // keySelector → A lambda that picks the key (K) from each element.
    // valueTransform → A lambda that transforms the element (T) into a value (V).
    // Returns a Map<K, V> where each key is from keySelector and each value

    val people = listOf(
    Person(1, "Alice"),
    Person(2, "Bob")
    )

    // Key = person.id, Value = person.name
    val nameById = people.associateBy(
        keySelector = { it.id },
        valueTransform = { it.name }
    )
    
    // Result: {1=Alice, 2=Bob}

    val orderEntries = checkoutResponse.orderLines
    ?.associateBy(
        keySelector = { it.lineNumber },
        valueTransform = { fromOrderLine(it) }
    )
    .orEmpty()

    // Explicitly states how to extract keys and compute values.
    // Best when your key is a property of the element.
    // ✅ This is the most idiomatic for the above case.
```
- **`associateWith`**: Transforms a collection into a map where each element becomes a key, and the value is produced by applying a function to the element.
  
  ```kotlin
  val words = listOf("apple", "banana", "cherry")
  val map = words.associateWith { it.length }
  // Result: {apple=5, banana=6, cherry=6}
  
  val orderEntries = checkoutResponse.orderLines
    ?.associateWith { fromOrderLine(it) }
    .orEmpty()
    // Result: {OrderLine(lineNumber=101, ...)=OrderEntry, OrderLine(lineNumber=102, ...)=OrderEntry, ...}
  
   // Here, the entire OrderLine object is the key.
   // That gives you a Map<OrderLine, OrderEntry>, not keyed by lineNumber.
  ```

**Key Difference:**
- Use `associate` when you want to control both the key and value.
- Use `associateWith` when the key is the element itself and you only want to compute the value.
- Use `associateBy` when you want to derive the key from a property of the element and optionally transform the value.

---
