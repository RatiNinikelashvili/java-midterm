## 1.Create a Product entity Create a JPA entity class called Product mapped to the table products.

 It must have: - id (Long, auto-generated primary key) - name (String, not nullable, max 100 chars) - price (BigDecimal, not nullable) - category (String) - createdAt (LocalDateTime, set automatically before persist) . Map a one-to-many relationship

---

# Product Entity



```
import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false)
    private BigDecimal price;

    private String category;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    // Example: One Product -> Many Reviews
    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Review> reviews;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
    }

    // Getters and Setters

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }

    public String getCategory() {
        return category;
    }

    public void setCategory(String category) {
        this.category = category;
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public List<Review> getReviews() {
        return reviews;
    }

    public void setReviews(List<Review> reviews) {
        this.reviews = reviews;
    }
}
```
---

# Example Child Entity (Review)

## To complete the one-to-many relationship, here’s a simple Review entity:

```
import jakarta.persistence.*;

@Entity
@Table(name = "reviews")
public class Review {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String comment;

    @ManyToOne
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    // Getters and Setters

    public Long getId() {
        return id;
    }

    public String getComment() {
        return comment;
    }

    public void setComment(String comment) {
        this.comment = comment;
    }

    public Product getProduct() {
        return product;
    }

    public void setProduct(Product product) {
        this.product = product;
    }
}
```

## 💡 Notes
- @PrePersist ensures createdAt is set automatically before insertion.
- cascade = CascadeType.ALL lets operations on Product propagate to Review.
- orphanRemoval = true ensures removed child entities are deleted.
- You can replace Review with any domain (e.g., OrderItem, Image, etc.)
---
  
## 2. You have two entities: Author and Book. Create both classes so that:
- One Author can have many Books (bidirectional)
- The books collection is lazily loaded
- Book has a ManyToOne back-reference to Author
- The foreign key column in books table is named author_id

## Author Class

```
import jakarta.persistence.*;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "authors")
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(
        mappedBy = "author",
        fetch = FetchType.LAZY,
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    private List<Book> books = new ArrayList<>();

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public List<Book> getBooks() {
        return books;
    }

    public void addBook(Book book) {
        books.add(book);
        book.setAuthor(this);
    }

    public void removeBook(Book book) {
        books.remove(book);
        book.setAuthor(null);
    }
}
```

## Book Class

```
import jakarta.persistence.*;

@Entity
@Table(name = "books")
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private Author author;

    public Long getId() {
        return id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public Author getAuthor() {
        return author;
    }

    public void setAuthor(Author author) {
        this.author = author;
    }
}
```

---

## 3. Write a JPQL search query In an OrderRepository, write a Spring Data method that:
- Accepts a String customerName and an OrderStatus status
- Returns all Orders where the customer name contains the given string (case-insensitive) AND the status matches
- Results must be ordered by createdAt descending
  
```
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("""
        SELECT o
        FROM Order o
        WHERE LOWER(o.customerName) LIKE LOWER(CONCAT('%', :customerName, '%'))
          AND o.status = :status
        ORDER BY o.createdAt DESC
    """)
    List<Order> findByCustomerNameContainingIgnoreCaseAndStatus(
            @Param("customerName") String customerName,
            @Param("status") OrderStatus status
    );
}
```

## ✅ Key Points
- LOWER(...) ensures case-insensitive matching.
- LIKE CONCAT('%', :customerName, '%') enables partial search.
- o.status = :status filters by the given enum.
- ORDER BY o.createdAt DESC ensures latest orders come first.

---
## 4. JOIN FETCH to avoid N+1 Write a repository method that loads all Department entities and eagerly fetches their employees collection in a single SQL query. Without JOIN FETCH this would trigger N+1 queries.

```
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface DepartmentRepository extends JpaRepository<Department, Long> {

    @Query("""
        SELECT DISTINCT d
        FROM Department d
        JOIN FETCH d.employees
    """)
    List<Department> findAllWithEmployees();
}
```

---

## 5. Interface-based projection. The full Employee entity has many fields. Create an interface-based projection called EmployeeSummary that exposes only id, firstName, lastName, and email. Then add a repository method that returns all employees as this projection

### - Create this projection

```
package ge.ibsu.demo.projection;

public interface EmployeeSummary {
    Long getId();
    String getFirstName();
    String getLastName();
    String getEmail();
}
```

### - Then in EmployeeRepository:

```
package ge.ibsu.demo.repository;

import ge.ibsu.demo.entity.Employee;
import ge.ibsu.demo.projection.EmployeeSummary;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    List<EmployeeSummary> findAllProjectedBy();
}
```

### findAllProjectedBy() tells Spring Data JPA: return all employees, but only map the fields exposed by EmployeeSummary.

The method expects your Employee entity to have these fields or getters:

```
private Long id;
private String firstName;
private String lastName;
private String email;
```

---

## 6. DTO (class-based) projection with @Query Create a DTO class called ProductStats that holds category (String) and averagePrice (Double). Then write a JPQL query in ProductRepository that returns average price grouped by category as a list of ProductStats. 

Hint: Given context @Entity @Table(name = "products") public class Product { @Id Long id; String name; BigDecimal price; String category; }

## 6.1 Create the DTO class (ProductStats)

This is a plain Java class (not an entity):

```
package ge.ibsu.demo.dto;

public class ProductStats {

    private String category;
    private Double averagePrice;

    public ProductStats(String category, Double averagePrice) {
        this.category = category;
        this.averagePrice = averagePrice;
    }

    public String getCategory() {
        return category;
    }

    public Double getAveragePrice() {
        return averagePrice;
    }
}
```

🔹 Important rule (very important)

👉 JPQL uses the constructor directly, so:

- The constructor must match exactly
- Order and types must match the query

## 6.2 Repository method with @Query

```
package ge.ibsu.demo.repository;

import ge.ibsu.demo.dto.ProductStats;
import ge.ibsu.demo.entity.Product;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

import java.util.List;

public interface ProductRepository extends JpaRepository<Product, Long> {

    @Query("""
        SELECT new ge.ibsu.demo.dto.ProductStats(
            p.category,
            AVG(p.price)
        )
        FROM Product p
        GROUP BY p.category
    """)
    List<ProductStats> findAveragePriceByCategory();
}
```
