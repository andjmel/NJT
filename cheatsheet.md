# Cheatsheet – NjtIoC1 + DiTime (Spring IoC/DI, JPA, Spring JDBC)

## 0. Priprema
- Kreirati folder `C:\njt_11_05_2026`.
- U NetBeans/IntelliJ: New Project -> Maven -> Java Application, **Project Location = C:\njt_11_05_2026**.
- Dva modula: `NjtIoC1` i `DiTime`.
- Baza: MySQL šema npr. `kolokvijum_njt` sa tabelama `pisac`, `izdavac`, `knjiga`, `korisnik`.

---

## 1. Entitet model (5 poena)

Tabele (SQL skica):

```sql
CREATE TABLE pisac (
  idpisac BIGINT AUTO_INCREMENT PRIMARY KEY,
  ime VARCHAR(50),
  prezime VARCHAR(50),
  godiste INT
);

CREATE TABLE korisnik (
  idkorisnik BIGINT AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(100),
  sifra VARCHAR(100),
  idpisac BIGINT,
  FOREIGN KEY (idpisac) REFERENCES pisac(idpisac)
);

CREATE TABLE izdavac (
  idizdavac BIGINT AUTO_INCREMENT PRIMARY KEY,
  naziv VARCHAR(100),
  pib VARCHAR(20),
  maticni_broj VARCHAR(20),
  sediste VARCHAR(100)
);

CREATE TABLE knjiga (
  idknjiga BIGINT AUTO_INCREMENT PRIMARY KEY,
  naziv VARCHAR(100),
  datum_izdavanja DATE,
  tiraz INT,
  idizdavac BIGINT,
  idpisac BIGINT,
  FOREIGN KEY (idizdavac) REFERENCES izdavac(idizdavac),
  FOREIGN KEY (idpisac) REFERENCES pisac(idpisac)
);
```

### Entitet Knjiga (JPA)

```java
@Entity
@Table(name = "knjiga")
public class Knjiga implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "idknjiga")
    private Long idknjiga;

    @Column(name = "naziv")
    private String naziv;

    @Column(name = "datum_izdavanja")
    private LocalDate datumIzdavanja;

    @Column(name = "tiraz")
    private Integer tiraz;

    @ManyToOne
    @JoinColumn(name = "idizdavac", referencedColumnName = "idizdavac")
    private Izdavac izdavac;

    @ManyToOne
    @JoinColumn(name = "idpisac", referencedColumnName = "idpisac")
    private Pisac pisac;

    // konstruktori, getteri, setteri
}
```

### Entitet Pisac

```java
@Entity
@Table(name = "pisac")
public class Pisac implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "idpisac")
    private Long idpisac;

    @Column(name = "ime")
    private String ime;

    @Column(name = "prezime")
    private String prezime;

    @Column(name = "godiste")
    private Integer godiste;

    @OneToMany(mappedBy = "pisac")
    private Collection<Knjiga> knjige;
}
```

### Entitet Izdavac

```java
@Entity
@Table(name = "izdavac")
public class Izdavac implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "idizdavac")
    private Long idizdavac;

    @Column(name = "naziv")
    private String naziv;

    @Column(name = "pib")
    private String pib;

    @Column(name = "maticni_broj")
    private String maticniBroj;

    @Column(name = "sediste")
    private String sediste;

    @OneToMany(mappedBy = "izdavac")
    private Collection<Knjiga> knjige;
}
```

### Entitet Korisnik (nalog pisca)

```java
@Entity
@Table(name = "korisnik")
public class Korisnik implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "idkorisnik")
    private Long idkorisnik;

    @Column(name = "email")
    private String email;

    @Column(name = "sifra")
    private String sifra;

    @OneToOne
    @JoinColumn(name = "idpisac", referencedColumnName = "idpisac")
    private Pisac pisac;
}
```

---

## 2. Spring IoC / DI projekat (NjtIoC1)

### pom.xml – ključne zavisnosti

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>7.0.7</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>7.0.7</version>
    </dependency>
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>9.7.0</version>
    </dependency>
    <dependency>
        <groupId>org.eclipse.persistence</groupId>
        <artifactId>org.eclipse.persistence.jpa</artifactId>
        <version>2.7.10</version>
    </dependency>
    <dependency>
        <groupId>jakarta.persistence</groupId>
        <artifactId>jakarta.persistence-api</artifactId>
        <version>3.1.0</version>
    </dependency>
</dependencies>
```

### AppConfig.java – IoC kontejner

```java
@ComponentScan(basePackages = {"fon.bg.ac.rs.njtioc1"})
public class AppConfig {

    @Bean("emf")
    public EntityManagerFactory getEntityManagerFactory() {
        return Persistence.createEntityManagerFactory("NjtPU");
    }

    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setUrl("jdbc:mysql://localhost:3306/kolokvijum_njt");
        ds.setUsername("root");
        ds.setPassword("");
        return ds;
    }

    @Bean("jdbc")
    public JdbcTemplate getJdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    @Bean("jpa-service")
    public BookService getJpaBookService(@Qualifier("jpa-repo") BookRepository repo,
                                          BookConverter conv) {
        return new BookServiceImpl(repo, conv);
    }

    @Bean("spring-service")
    public BookService getSpringBookService(@Qualifier("spring-repo") BookRepository repo,
                                             BookConverter conv) {
        return new BookServiceImpl(repo, conv);
    }
}
```

### persistence.xml (src/main/resources/META-INF/)

```xml
<persistence version="2.1" xmlns="http://xmlns.jcp.org/xml/ns/persistence">
  <persistence-unit name="NjtPU" transaction-type="RESOURCE_LOCAL">
    <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
    <class>fon.bg.ac.rs.njtioc1.domain.Pisac</class>
    <class>fon.bg.ac.rs.njtioc1.domain.Knjiga</class>
    <class>fon.bg.ac.rs.njtioc1.domain.Korisnik</class>
    <class>fon.bg.ac.rs.njtioc1.domain.Izdavac</class>
    <properties>
      <property name="javax.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/kolokvijum_njt"/>
      <property name="javax.persistence.jdbc.user" value="root"/>
      <property name="javax.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
      <property name="javax.persistence.jdbc.password" value=""/>
    </properties>
  </persistence-unit>
</persistence>
```

---

## 3. Repozitorijum (5 poena) – JPA + Spring JDBC implementacije

### Interfejs

```java
public interface BookRepository {
    Book save(Book b);
    void delete(Book b);
}
```

### JPA implementacija

```java
@Repository("jpa-repo")
public class BookJpaRepository implements BookRepository {

    private final EntityManagerFactory emf;

    @Autowired
    public BookJpaRepository(@Qualifier("emf") EntityManagerFactory emf) {
        this.emf = emf;
    }

    @Override
    public Book save(Book b) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        Book merged = em.merge(b);
        em.getTransaction().commit();
        em.close();
        return merged;
    }

    @Override
    public void delete(Book b) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        Book toRemove = em.find(Book.class, b.getIdknjiga());
        if (toRemove != null) em.remove(toRemove);
        em.getTransaction().commit();
        em.close();
    }
}
```

### Spring JDBC implementacija

```java
@Repository("spring-repo")
public class BookSpringJdbcRepository implements BookRepository {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public BookSpringJdbcRepository(@Qualifier("jdbc") JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Book save(Book b) {
        jdbcTemplate.update(
            "INSERT INTO knjiga (naziv, datum_izdavanja, tiraz, idizdavac, idpisac) VALUES (?,?,?,?,?)",
            b.getNaziv(), b.getDatumIzdavanja(), b.getTiraz(),
            b.getIzdavac() != null ? b.getIzdavac().getIdizdavac() : null,
            b.getPisac() != null ? b.getPisac().getIdpisac() : null
        );
        return b;
    }

    @Override
    public void delete(Book b) {
        jdbcTemplate.update("DELETE FROM knjiga WHERE idknjiga = ?", b.getIdknjiga());
    }
}
```

---

## 4. Converter (5 poena)

```java
public interface Converter<Dto, Entity> {
    Entity toEntity(Dto dto);
    Dto toDto(Entity entity);
}
```

```java
@Component("conv")
public class BookConverter implements Converter<BookDto, Book> {

    @Override
    public Book toEntity(BookDto dto) {
        Book b = new Book();
        b.setIdknjiga(dto.getIdknjiga());
        b.setNaziv(dto.getNaziv());
        b.setDatumIzdavanja(dto.getDatumIzdavanja());
        b.setTiraz(dto.getTiraz());

        if (dto.getIzdavacId() != null) {
            Izdavac i = new Izdavac();
            i.setIdizdavac(dto.getIzdavacId());
            b.setIzdavac(i);
        }
        if (dto.getPisacId() != null) {
            Pisac p = new Pisac();
            p.setIdpisac(dto.getPisacId());
            b.setPisac(p);
        }
        return b;
    }

    @Override
    public BookDto toDto(Book entity) {
        BookDto dto = new BookDto();
        dto.setIdknjiga(entity.getIdknjiga());
        dto.setNaziv(entity.getNaziv());
        dto.setDatumIzdavanja(entity.getDatumIzdavanja());
        dto.setTiraz(entity.getTiraz());
        dto.setIzdavacId(entity.getIzdavac() != null ? entity.getIzdavac().getIdizdavac() : null);
        dto.setPisacId(entity.getPisac() != null ? entity.getPisac().getIdpisac() : null);
        return dto;
    }
}
```

---

## 5. Servis (5 poena)

```java
public interface BookService {
    BookDto save(BookDto b) throws BookExistException;
    void delete(BookDto b) throws BookDoesNotExistException;
}
```

### Custom izuzeci

```java
public class BookExistException extends Exception {
    public BookExistException(String msg) { super(msg); }
}

public class BookDoesNotExistException extends Exception {
    public BookDoesNotExistException(String msg) { super(msg); }
}
```

### Implementacija

```java
public class BookServiceImpl implements BookService {

    private final BookRepository repo;
    private final BookConverter conv;

    public BookServiceImpl(BookRepository repo, BookConverter conv) {
        this.repo = repo;
        this.conv = conv;
    }

    @Override
    public BookDto save(BookDto dto) throws BookExistException {
        // provera da li knjiga sa istim nazivom već postoji
        if (repo.findByNaziv(dto.getNaziv()) != null) {
            throw new BookExistException("Knjiga sa nazivom '" + dto.getNaziv() + "' već postoji!");
        }
        Book entity = conv.toEntity(dto);
        Book saved = repo.save(entity);
        return conv.toDto(saved);
    }

    @Override
    public void delete(BookDto dto) throws BookDoesNotExistException {
        if (dto.getIdknjiga() == null || repo.findById(dto.getIdknjiga()) == null) {
            throw new BookDoesNotExistException("Knjiga ne postoji!");
        }
        repo.delete(conv.toEntity(dto));
    }
}
```

> Napomena: `findByNaziv` / `findById` dodati u `BookRepository` interfejs i implementirati u obe repo klase (JPA preko `EntityManager.find` / JPQL upita, Spring JDBC preko `queryForObject`).

---

## 6. Application klasa (5 poena)

```java
@Component
public class Application {

    private final BookService jpaService;
    private final BookService springService;

    @Autowired
    public Application(@Qualifier("jpa-service") BookService jpaService,
                        @Qualifier("spring-service") BookService springService) {
        this.jpaService = jpaService;
        this.springService = springService;
    }

    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        Application app = ctx.getBean(Application.class);

        Scanner sc = new Scanner(System.in);
        System.out.println("Izaberite repozitorijum: 1-JPA  2-Spring JDBC");
        int repoChoice = sc.nextInt();

        BookService service = (repoChoice == 1) ? app.jpaService : app.springService;

        System.out.println("1-save  2-delete");
        int action = sc.nextInt();

        try {
            if (action == 1) {
                BookDto dto = new BookDto();
                dto.setNaziv("Na Drini cuprija");
                dto.setTiraz(1000);
                dto.setPisacId(1L);
                dto.setIzdavacId(1L);
                service.save(dto);
                System.out.println("Sacuvano.");
            } else {
                BookDto dto = new BookDto();
                dto.setIdknjiga(1L);
                service.delete(dto);
                System.out.println("Obrisano.");
            }
        } catch (BookExistException | BookDoesNotExistException ex) {
            System.out.println("Greska: " + ex.getMessage());
        }
    }
}
```

---

## 7. DiTime aplikacija (5 poena)

### pom.xml – samo spring-context

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>7.0.7</version>
</dependency>
```

### SpringIoc.java – konfiguracija/kontejner

```java
@ComponentScan(basePackages = {"fon.bg.ac.rs.ditime"})
public class SpringIoc {

    @Bean("time")
    public LocalTime getTime() {
        return LocalTime.now();
    }
}
```

### DiTimeApp.java – bin koji koristi DiTime bin (vreme)

```java
@Component
public class DiTimeApp {

    private final LocalTime time;

    @Autowired
    public DiTimeApp(@Qualifier("time") LocalTime time) {
        this.time = time;
    }

    public LocalTime getCurrentTime() {
        return time;
    }
}
```

### Main klasa

```java
public class DiTime {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringIoc.class);
        DiTimeApp app = ctx.getBean(DiTimeApp.class);
        System.out.println("Trenutno vreme: " + app.getCurrentTime());
    }
}
```

---

## Checklist pred predaju
- [ ] Projekat fizički u `C:\njt_11_05_2026`
- [ ] `Pisac` ima `ime, prezime, godiste`
- [ ] `Izdavac` ima `naziv, pib, maticniBroj, sediste (String)`
- [ ] `Knjiga` ima `naziv, datumIzdavanja, tiraz, izdavac, pisac`
- [ ] `Korisnik` povezan 1:1 sa `Pisac` (email, sifra)
- [ ] 2 implementacije repozitorijuma (JPA + Spring JDBC) sa `save`/`delete`
- [ ] `Converter<Dto,Entity>` implementiran, bez NPE na null FK
- [ ] `BookService` baca `BookExistException` (po nazivu) i `BookDoesNotExistException`
- [ ] `Application` bira repo preko korisničkog unosa i poziva servis
- [ ] `DiTimeApp` bin koristi `LocalTime` preko Spring IoC kontejnera
