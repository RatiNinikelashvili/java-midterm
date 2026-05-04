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

---

## Add the following enhancements to the Spring project based on the sample HR database:

- Create new entities: Location, Country, and Region.
- Update the Department entity:
- Add a relationship to Location.
- Add a manager property (linked to the appropriate employee entity).
- Implement an API endpoint to retrieve a list of all departments with the following fields: Department name, Manager full name (concatenation of firstName and lastName), Country, City, Street address.
- The API should support filtering by:
Country
City

## Location Entity

```
package ge.ibsu.demo.entities;

import jakarta.persistence.*;

import java.util.List;

@Entity
public class Location {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String streetAddress;
    private String city;

    @ManyToOne
    @JoinColumn(name = "country_id")
    private Country country;

    @OneToMany(mappedBy = "location")
    private List<Department> departments;

    public String getStreetAddress() {
        return streetAddress;
    }

    public void setStreetAddress(String streetAddress) {
        this.streetAddress = streetAddress;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

    public Country getCountry() {
        return country;
    }

    public void setCountry(Country country) {
        this.country = country;
    }

    public List<Department> getDepartments() {
        return departments;
    }

    public void setDepartments(List<Department> departments) {
        this.departments = departments;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getId() {
        return id;
    }
}
```

## Country Entity

```
package ge.ibsu.demo.entities;
import jakarta.persistence.*;

import java.util.List;


@Entity
public class Country {
    @Id
    private String id;
    private String name;

    @ManyToOne
    @JoinColumn(name= "region_id")
    private Region region;

    @OneToMany(mappedBy = "country")
    private List<Location> locations;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getId() {
        return id;
    }
}
```

## Region Entity

```
package ge.ibsu.demo.entities;

import jakarta.persistence.*;

import java.util.List;

@Entity
public class Region {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "region")
    private List<Country> countries;
}
```

## Department Entity

```
package ge.ibsu.demo.entities;

import jakarta.persistence.*;
import org.apache.catalina.Manager;

@Entity
@Table(name = "departments")
public class Department {

    @Id
    @Column(name = "department_id")
    private Long id;

    @Column(name = "department_name")
    private String name;

    @ManyToOne
    @JoinColumn(name = "location_id")
    private Location location;

    @ManyToOne
    @JoinColumn(name = "manager_id")
    private Employee manager;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

## DepartmentRepository.java

```
package ge.ibsu.demo.repositories;

import ge.ibsu.demo.dto.DepartmentDetails;
import ge.ibsu.demo.entities.Department;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface DepartmentRepository extends JpaRepository<Department, Long> {
    @Query("""
        SELECT new ge.ibsu.demo.dto.DepartmentDetails(
            d.name,
            CONCAT(m.firstName, ' ', m.lastName),
            c.name,
            l.city,
            l.streetAddress
        )
        FROM Department d
        LEFT JOIN d.manager m
        LEFT JOIN d.location l
        LEFT JOIN l.country c
        WHERE (:country IS NULL OR c.name = :country)
        AND (:city IS NULL OR l.city = :city)
    """)
    List<DepartmentDetails> findDepartments(
            @Param("country") String country,
            @Param("city") String city
    );
}
```

## DepartmentController.java

```
package ge.ibsu.demo.controllers;

import ge.ibsu.demo.dto.DepartmentDetails;
import ge.ibsu.demo.entities.Department;
import ge.ibsu.demo.entities.Employee;
import ge.ibsu.demo.services.DepartmentService;
import ge.ibsu.demo.services.EmployeeService;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/departments")
public class DepartmentController {

    private final DepartmentService departmentService;

    private final EmployeeService employeeService;

    public DepartmentController(DepartmentService departmentService, EmployeeService employeeService) {
        this.departmentService = departmentService;
        this.employeeService = employeeService;
    }

    @GetMapping("/all")
    public List<Department> getAll() {
        return departmentService.getAll();
    }

    @GetMapping("/{id}")
    public Department getById(@PathVariable Long id) throws Exception {
        return departmentService.getById(id);
    }

    @GetMapping("/{id}/employees")
    public List<Employee> getEmployees(@PathVariable Long id) {
        return employeeService.getByDepartment(id);
    }

    @GetMapping("/details")
    public List<DepartmentDetails> getDepartments(
            @RequestParam(required = false) String country,
            @RequestParam(required = false) String city
    ) {
        return departmentService.getDepartments(country, city);
    }
}
```