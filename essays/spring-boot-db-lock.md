---
layout: essay
type: essay
title: 'Handling Database Locks in Spring Boot: Solving Concurrency (AI-GEN-BLOG 🤖)'
# All dates must be YYYY-MM-DD format!
date: 2025-04-16
labels:
  - AI-GEN-BLOG
  - Java
  - Spring
  - RDBMS
  - Concurrency
---

<figure class="ui image centered">
	<img src="../images/spring-boot-db-lock-essay-cover.png">
  <figcaption class="ui centered label">Gemini generated image</figcaption>
</figure>

Concurrency issues are common in modern applications where multiple users interact with the same data. One such issue is when two users try to purchase the last item of a product at the same time. In this blog post, we'll explore how to solve this using **database locks** in a **Spring Boot** application with **Spring Data JPA** and **PostgreSQL**.

## ✨ Objectives

- Understand why database locks are important
- Learn how to use **pessimistic** and **optimistic** locking in Spring Boot
- Demonstrate the problem using a real-world e-commerce example
- Solve it using **pessimistic locking**
- Write a test that simulates the issue and proves the fix

---

## 📝 The Problem: Race Condition During Purchase

Imagine a scenario where only one unit of a product is left in stock. Two users attempt to purchase it at the same time. If your code looks like this:

```java
@Transactional
public void purchaseProduct(Long productId) {
    // Step 1: Load product from DB
    Product product = productRepository.findById(productId)
            .orElseThrow(() -> new RuntimeException("Product not found"));

    // Step 2: Check quantity
    if (product.getQuantity() > 0) {
        // Step 3: Reduce and save
        product.setQuantity(product.getQuantity() - 1);
        productRepository.save(product);
    } else {
        throw new RuntimeException("Out of stock");
    }
}
```

This method is annotated with `@Transactional`, which ensures that the operations within it are executed in a single transaction. However, **this does not prevent concurrent transactions from reading the same value at the same time**.

### 🐞 What Goes Wrong?

Let's say:

- The database has one unit of Product A.
- Thread A and Thread B both invoke `purchaseProduct(1)` at nearly the same moment.

Here’s what can happen:

1. **Thread A** reads the product and sees quantity = 1.
2. **Thread B** does the same before A commits.
3. Both threads pass the `if (quantity > 0)` check.
4. Both reduce quantity by 1 and save.

Now you've sold the same item twice! The DB allows this because no lock was enforced during the `findById` call.

---

## 🔒 The Solution: Pessimistic Locking

With pessimistic locking, a row in the database is locked as soon as it is read, preventing other transactions from modifying it until the lock is released.

### ✅ Update the Repository

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

The `@Lock(LockModeType.PESSIMISTIC_WRITE)` annotation ensures that the database acquires a lock on the row, preventing other transactions from reading or writing to it until the current transaction finishes.

### ✅ Update the Service Method

```java
@Service
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @Transactional
    public void purchaseProduct(Long productId) {
        // Acquires a DB lock on the row, blocking concurrent access
        Product product = productRepository.findByIdForUpdate(productId)
                .orElseThrow(() -> new RuntimeException("Product not found"));

        if (product.getQuantity() > 0) {
            product.setQuantity(product.getQuantity() - 1);
            productRepository.save(product);
        } else {
            throw new ProductOutOfStockException("Product is out of stock");
        }
    }
}
```

This prevents multiple threads from simultaneously reading and modifying the same row, eliminating the overselling issue.

---

## 🔍 Test: Simulate Concurrency

We simulate two concurrent purchases for the last item in stock. Only one should succeed.

```java
@SpringBootTest
@DirtiesContext(classMode = DirtiesContext.ClassMode.BEFORE_EACH_TEST_METHOD)
public class ProductServiceConcurrencyTest {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private ProductService productService;

    @BeforeEach
    public void setup() {
        Product product = new Product();
        product.setName("Test Product");
        product.setQuantity(1);
        productRepository.save(product);
    }

    @Test
    public void testConcurrentPurchaseThrowsOutOfStock() throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(2);
        Long productId = productRepository.findAll().get(0).getId();

        List<Future<Exception>> futures = new ArrayList<>();

        Callable<Exception> task = () -> {
            try {
                productService.purchaseProduct(productId);
                return null;
            } catch (Exception e) {
                return e;
            }
        };

        futures.add(executor.submit(task));
        futures.add(executor.submit(task));

        executor.shutdown();
        executor.awaitTermination(5, TimeUnit.SECONDS);

        int outOfStockCount = 0;
        for (Future<Exception> future : futures) {
            Exception ex = future.get();
            if (ex instanceof ProductOutOfStockException) {
                outOfStockCount++;
            }
        }

        assertEquals(1, outOfStockCount);
        Product updatedProduct = productRepository.findById(productId).orElseThrow();
        assertEquals(0, updatedProduct.getQuantity());
    }
}
```

---

## 🧠 Alternative: Optimistic Locking

Optimistic locking is another approach where a version field is used to detect concurrent modifications. It's great when conflicts are rare and performance is critical. If a conflict is detected, you can retry the operation using `@Retryable` from Spring Retry.

We won't dive into the code here, but optimistic locking is ideal when you want fast reads and are okay with occasional retries.

---

## 🧠 Summary
<table>
<tr>
    <th>Lock Type</th>
    <th>Use When</th>
    <th>Notes</th>
</tr>
<tr>
    <td>Pessimistic Lock</td>
    <td>Conflicts are frequent</td>
    <td>Uses actual DB row locks</td>
</tr>
<tr>
    <td>Optimistic Lock  </td>
    <td>Conflicts are rare, high read volume </td>
    <td>Uses versioning, faster, needs retry mechanism</td>
</tr>
</table>

Using **Spring Data JPA + Pessimistic Locking**, you can reliably handle race conditions. For read-heavy systems with rare conflicts, **Optimistic Locking** with retry support can be an efficient alternative.

## 🔗 Resources
 This blog is [generated by ChatGPT](https://chatgpt.com/canvas/shared/67ff697092788191a301a103e5530a71) as a summary of a conversation about [Redhwan Nacef](https://www.youtube.com/@RedhwanNacef) video [Locks Explained | With Real-World E-Commerce Example](https://www.youtube.com/watch?v=Juq20LYXpSU) and how it's implemented in the context of Java and Spring.

