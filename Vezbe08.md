# Vezbe08 — Kompletan Tutorijal: JPA, Spring JDBC i Hibernate ORM

> **Projekat:** Čuvanje podataka o predavačima koristeći tri različite tehnologije pristupa bazi podataka u okviru Spring IoC kontejnera.

---

## Sadržaj

1. [Struktura projekta](#1-struktura-projekta)
2. [Model podataka (domenski sloj)](#2-model-podataka-domenski-sloj)
3. [DTO — Data Transfer Object](#3-dto--data-transfer-object)
4. [Konfiguracija (AppConfig)](#4-konfiguracija-appconfig)
5. [Repository sloj — interfejs](#5-repository-sloj--interfejs)
6. [Repository implementacije](#6-repository-implementacije)
   - [6.1 JPA Repository](#61-jpa-repository)
   - [6.2 Spring JDBC Repository](#62-spring-jdbc-repository)
   - [6.3 Hibernate ORM Repository](#63-hibernate-orm-repository)
7. [Service sloj](#7-service-sloj)
8. [Konfiguracioni fajlovi](#8-konfiguracioni-fajlovi)
   - [8.1 persistence.xml (JPA)](#81-persistencexml-jpa)
   - [8.2 hibernate.cfg.xml](#82-hibernatecfgxml)
   - [8.3 pom.xml (Maven zavisnosti)](#83-pomxml-maven-zavisnosti)
9. [Glavna klasa — Vezbe08.java](#9-glavna-klasa--vezbe08java)
10. [Tok izvršavanja programa](#10-tok-izvršavanja-programa)
11. [Šema baze podataka](#11-šema-baze-podataka)
12. [Poređenje tehnologija](#12-poređenje-tehnologija)

---

## 1. Struktura projekta

```
Vezbe08/
├── pom.xml                          ← Maven konfiguracija (zavisnosti)
└── src/main/
    ├── java/rs/ac/bg/fon/njt/vezbe08/
    │   ├── Vezbe08.java             ← Ulazna tačka programa (main metoda)
    │   ├── config/
    │   │   └── AppConfig.java       ← Spring konfiguracija (IoC bean-ovi)
    │   ├── domain/
    │   │   ├── Katedra.java         ← Entitet: tabela katedra
    │   │   ├── Predavac.java        ← Entitet: tabela predavac
    │   │   └── Zvanje.java          ← Entitet: tabela zvanje
    │   ├── dto/
    │   │   └── PredavacDto.java     ← DTO za prenos podataka
    │   ├── repository/
    │   │   ├── PredavacRepository.java           ← Interfejs repozitorijuma
    │   │   └── impl/
    │   │       ├── PredavacJpaRepository.java     ← JPA implementacija
    │   │       ├── PredavacSpringJdbcRepository.java ← Spring JDBC impl.
    │   │       └── PredavacHibernateRepository.java  ← Hibernate impl.
    │   └── service/
    │       ├── PredavacService.java              ← Interfejs servisa
    │       └── impl/
    │           └── PredavacServiceImpl.java      ← Implementacija servisa
    └── resources/
        ├── hibernate.cfg.xml        ← Hibernate konfiguracija
        └── META-INF/
            └── persistence.xml      ← JPA konfiguracija
```

**Zašto ovakva struktura?**  
Projekat prati tzv. **slojevitu arhitekturu** (layered architecture):
- `domain` → Java objekti koji odgovaraju tabelama u bazi
- `dto` → objekti za prenos podataka između slojeva (zaštita domenskog modela)
- `repository` → direktna komunikacija sa bazom
- `service` → poslovna logika, koordinira repozitorijume
- `config` → Spring IoC konfiguracija

---

## 2. Model podataka (domenski sloj)

### Zvanje.java

```java
@Entity                          // Govori JPA/Hibernate-u: ovo je tabela u bazi
@Table(name = "zvanje")          // Mapira klasu na tabelu "zvanje"
public class Zvanje {

    @Id                          // Primarni ključ
    @Column(name = "idzvanje")   // Mapira polje na kolonu "idzvanje"
    private Long idZvanje;

    @Column(name = "srpskiprevod")
    private String srpskiPrevod;

    @Column(name = "engleskiprevod")
    private String engleskiPrevod;

    // konstruktori, getteri, setteri, toString...
}
```

**Objašnjenje anotacija:**
| Anotacija | Šta radi |
|-----------|----------|
| `@Entity` | Označava da je ova klasa JPA entitet — ima svoju tabelu u bazi |
| `@Table(name="...")` | Eksplicitno navodi ime tabele (ako se razlikuje od naziva klase) |
| `@Id` | Označava primarni ključ |
| `@Column(name="...")` | Mapira Java polje na kolonu u tabeli (kad se nazivi razlikuju) |

> `Zvanje` nema `@GeneratedValue` jer se ID unosi ručno (šifarnik sa fiksnim vrednostima).

---

### Katedra.java

```java
@Entity
@Table(name = "katedra")
public class Katedra {

    @Id
    @Column(name = "idkatedra")
    private Long idKatedra;

    private String naziv;          // kolona se zove "naziv" — ime polja = ime kolone

    @Column(name = "skraceninaziv")
    private String skraceniNaziv;

    // konstruktori, getteri, setteri, toString...
}
```

Ista logika kao `Zvanje` — šifarnik sa poznatim ID-jevima, bez auto-generisanja.

---

### Predavac.java

```java
@Entity
@Table(name = "predavac")
public class Predavac {

    @Id
    @Column(name = "idpredavac")
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    // ↑ ID se automatski generiše u bazi (AUTO_INCREMENT)
    private Long idPredavac;

    @Column(name = "imeprezime")
    private String imePrezime;

    @Column(name = "datumrodjenja")
    private LocalDate datumRodjenja;   // Java 8 tip — Hibernate i JPA to podržavaju

    @ManyToOne                         // Mnogo predavača → jedna katedra
    @JoinColumn(name = "katedraid")    // Strani ključ u tabeli predavac
    private Katedra katedra;

    @ManyToOne                         // Mnogo predavača → jedno zvanje
    @JoinColumn(name = "zvanjeid")     // Strani ključ u tabeli predavac
    private Zvanje zvanje;

    // konstruktori, getteri, setteri, toString...
}
```

**Ključne anotacije:**
| Anotacija | Šta radi |
|-----------|----------|
| `@GeneratedValue(strategy = GenerationType.IDENTITY)` | Baza generiše ID (AUTO_INCREMENT u MySQL-u) |
| `@ManyToOne` | Veza više-prema-jedan: mnogo predavača može imati isti zvanje/katedru |
| `@JoinColumn(name="katedraid")` | U tabeli `predavac` postoji kolona `katedraid` koja je FK ka tabeli `katedra` |

**Šema veza:**
```
predavac (idpredavac, imeprezime, datumrodjenja, katedraid FK, zvanjeid FK)
    ↓ katedraid                          ↓ zvanjeid
katedra (idkatedra, naziv, skraceninaziv)   zvanje (idzvanje, srpskiprevod, engleskiprevod)
```

---

## 3. DTO — Data Transfer Object

```java
public class PredavacDto {
    private Long idPredavac;
    private String imePrezime;
    private LocalDate datumRodjenja;
    private Katedra katedra;
    private Zvanje zvanje;

    // konstruktori, getteri, setteri...
}
```

**Zašto DTO?**  
DTO (Data Transfer Object) je objekat koji prenosi podatke između slojeva aplikacije. U ovom projektu:

- **Spolja** (main metoda, UI) radi sa `PredavacDto` — ne zna ništa o bazi
- **Unutra** (repository) radi sa `Predavac` entitetom — koji ima JPA anotacije

Ovo razdvajanje štiti domenski model od direktne manipulacije i ostavlja fleksibilnost da se interna implementacija promeni bez uticaja na ostale delove koda.

Konverzija DTO → entitet dešava se u **service sloju**:
```java
new Predavac(dto.getIdPredavac(), dto.getImePrezime(), dto.getDatumRodjenja(),
             dto.getKatedra(), dto.getZvanje())
```

---

## 4. Konfiguracija (AppConfig)

```java
@ComponentScan(basePackages = {"rs.ac.bg.fon.njt.vezbe08"})
public class AppConfig {
```

`@ComponentScan` govori Springu: "skeniraj sve klase u ovom paketu i podpaketu i registruj one koje imaju `@Component`, `@Service`, `@Repository` anotacije kao bean-ove".

### Bean: EntityManagerFactory (JPA)

```java
@Bean(value = "emf")
public EntityManagerFactory createEntityManagerFactory() {
    return Persistence.createEntityManagerFactory("Vezbe09PU");
    // "Vezbe09PU" je ime persistence-unit iz persistence.xml
}
```

`EntityManagerFactory` je fabrički objekat koji kreira `EntityManager`-e. Skupo se kreira (jednom pri startu), zato je Spring bean (singleton po defaultu).

### Bean: DataSource (konekcija ka bazi)

```java
@Bean
public DataSource dataSource() {
    DriverManagerDataSource datasource = new DriverManagerDataSource();
    datasource.setUrl("jdbc:mysql://localhost:3306/njt_hibernate");
    datasource.setUsername("root");
    datasource.setPassword("root");
    return datasource;
}
```

`DataSource` je standardni Java interfejs koji predstavlja izvor konekcija ka bazi. `DriverManagerDataSource` je Spring-ova jednostavna implementacija (svaki put otvori novu konekciju — za produkciju bi se koristio connection pool).

### Bean: JdbcTemplate

```java
@Bean
public JdbcTemplate jdbcTemplate(DataSource dataSource) {
    return new JdbcTemplate(dataSource);
}
```

`JdbcTemplate` je Spring-ov wrapper oko standardnog JDBC-a. Spring automatski injektuje `DataSource` bean (koji smo upravo definisali) kao parametar.

### Bean: SessionFactory (Hibernate)

```java
@Bean
public SessionFactory getSessionFactory() {
    return new Configuration().configure("hibernate.cfg.xml").buildSessionFactory();
}
```

`SessionFactory` je Hibernate-ov ekvivalent `EntityManagerFactory`-ju. Čita `hibernate.cfg.xml` i kreira fabriku sesija.

### Tri service bean-a sa Qualifier

```java
@Bean(value = "JpaPredavacService")
public PredavacService getPredavacJpaService(
        @Qualifier(value = "predavac-jpa-repository") PredavacRepository predavacRepository) {
    return new PredavacServiceImpl(predavacRepository);
}

@Bean(value = "SpringJdbcPredavacService")
public PredavacService getPredavacSpringJdbcService(
        @Qualifier(value = "predavac-spring-jdbc-repository") PredavacRepository predavacRepository) {
    return new PredavacServiceImpl(predavacRepository);
}

@Bean(value = "HibernatePredavacService")
public PredavacService getPredavacHibernateService(
        @Qualifier(value = "hibernate-repository") PredavacRepository predavacRepository) {
    return new PredavacServiceImpl(predavacRepository);
}
```

Sve tri metode vraćaju isti tip (`PredavacServiceImpl`), ali svaka dobija **drugu implementaciju repozitorijuma**. `@Qualifier` precizira koji bean da se injektuje kada postoji više kandidata istog tipa.

---

## 5. Repository sloj — interfejs

```java
public interface PredavacRepository {
    void save(Predavac predavac);
}
```

Ovo je **ugovor** — svaka implementacija mora imati `save` metodu. Ovaj pattern se zove **programiranje prema interfejsu** i omogućava:
- Laku zamenu implementacije (JPA ↔ JDBC ↔ Hibernate) bez promene service sloja
- Testabilnost (možeš zameniti pravu implementaciju sa mock-om)

---

## 6. Repository implementacije

### 6.1 JPA Repository

```java
@Repository(value = "predavac-jpa-repository")
// ↑ Registruje se kao Spring bean sa ovim imenom
public class PredavacJpaRepository implements PredavacRepository {

    private EntityManagerFactory emf;

    @Autowired
    public PredavacJpaRepository(@Qualifier(value = "emf") EntityManagerFactory emf) {
        this.emf = emf;
        // Spring injektuje "emf" bean koji je definisan u AppConfig-u
    }

    @Override
    public void save(Predavac predavac) {
        EntityManager em = emf.createEntityManager();
        // EntityManager je glavna JPA klasa za rad sa bazom
        
        em.getTransaction().begin();
        // Počni transakciju — sve operacije su atomične
        
        em.merge(predavac);
        // merge() — upiši ili ažuriraj entitet.
        // Ako predavac ima ID koji već postoji → UPDATE
        // Ako nema ili je null → INSERT
        
        em.getTransaction().commit();
        // Potvrdi transakciju — promene se fizički upisuju u bazu
        
        em.close();
        // Oslobodi resurse EntityManager-a
    }
}
```

**Zašto `merge()` a ne `persist()`?**  
- `persist()` — radi samo za **nove** entitete (bez ID-ja). Ako entitet ima ID, baca grešku.
- `merge()` — radi i za nove i za postojeće (upsert logika).

---

### 6.2 Spring JDBC Repository

```java
@Repository(value = "predavac-spring-jdbc-repository")
public class PredavacSpringJdbcRepository implements PredavacRepository {

    private JdbcTemplate jdbcTemplate;

    @Autowired
    public PredavacSpringJdbcRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
        // Spring injektuje JdbcTemplate bean iz AppConfig-a
    }

    @Override
    public void save(Predavac predavac) {
        jdbcTemplate.update(
            "INSERT INTO predavac(imeprezime, datumrodjenja, katedraid, zvanjeid) VALUES (?,?,?,?)",
            predavac.getImePrezime(),
            predavac.getDatumRodjenja(),
            predavac.getKatedra().getIdKatedra(),   // uzimamo samo ID, ne ceo objekat
            predavac.getZvanje().getIdZvanje()
        );
    }
}
```

**Kako radi `jdbcTemplate.update()`?**  
- Prvi argument je SQL upit sa `?` kao placeholder-ima
- Ostali argumenti se redosljedno vezuju za placeholder-e (1. `?` → `imePrezime`, 2. `?` → `datumRodjenja`, itd.)
- Spring automatski upravlja otvaranjem/zatvaranjem konekcije i hendluje `SQLException`
- Za ID katedre/zvanja upisujemo samo numeričku vrednost (`getIdKatedra()`) jer baza čuva strani ključ, ne ceo objekat

---

### 6.3 Hibernate ORM Repository

```java
@Repository(value = "hibernate-repository")
public class PredavacHibernateRepository implements PredavacRepository {

    private SessionFactory sf;

    @Autowired
    public PredavacHibernateRepository(SessionFactory sf) {
        this.sf = sf;
        // Spring injektuje SessionFactory bean iz AppConfig-a
    }

    @Override
    public void save(Predavac predavac) {
        Session session = sf.openSession();
        // Session je Hibernate-ov ekvivalent JPA EntityManager-u
        // Otvori novu sesiju (konekciju ka bazi)
        
        Transaction tr = session.beginTransaction();
        // Počni transakciju
        
        session.persist(predavac);
        // persist() u Hibernate-u → INSERT u bazu
        
        tr.commit();
        // Potvrdi transakciju
        
        // Napomena: session se ne zatvara eksplicitno — mogući memory leak
        // Bolje bi bilo: session.close() u finally bloku
    }
}
```

**Hibernate vs JPA:**  
Hibernate je implementacija JPA specifikacije. Razlika:
- `EntityManager` (JPA API) ↔ `Session` (Hibernate API)
- `EntityManagerFactory` (JPA) ↔ `SessionFactory` (Hibernate)
- `em.merge()` ↔ `session.persist()`

U ovom projektu, JPA repository koristi **EclipseLink** kao JPA provider (definisano u `persistence.xml`), dok Hibernate repository direktno koristi Hibernate API.

---

## 7. Service sloj

### PredavacService.java (interfejs)

```java
public interface PredavacService {
    void save(PredavacDto predavacDto);
    // Prima DTO — ne zna ništa o entitetima ili bazi
}
```

### PredavacServiceImpl.java

```java
public class PredavacServiceImpl implements PredavacService {

    private PredavacRepository predavacRepository;

    @Autowired
    public PredavacServiceImpl(PredavacRepository predavacRepository) {
        this.predavacRepository = predavacRepository;
        // Koja implementacija repozitorijuma će biti injektovana
        // zavisi od toga kako je service kreiran u AppConfig-u
    }

    @Override
    public void save(PredavacDto predavacDto) {
        // Konverzija DTO → domenski entitet
        predavacRepository.save(
            new Predavac(
                predavacDto.getIdPredavac(),
                predavacDto.getImePrezime(),
                predavacDto.getDatumRodjenja(),
                predavacDto.getKatedra(),
                predavacDto.getZvanje()
            )
        );
    }
}
```

Service sloj je "most" između spoljnog sveta (DTO) i baze (entiteti). Ovde bi se mogla dodati poslovna logika (validacija, kalkulacije, itd.).

---

## 8. Konfiguracioni fajlovi

### 8.1 persistence.xml (JPA)

```xml
<persistence-unit name="Vezbe09PU" transaction-type="RESOURCE_LOCAL">
    <!-- Ime mora da odgovara onome u AppConfig.java:
         Persistence.createEntityManagerFactory("Vezbe09PU") -->

    <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
    <!-- Koji JPA provider se koristi — ovde EclipseLink -->

    <!-- Sve klase koje su entiteti moraju biti navedene -->
    <class>rs.ac.bg.fon.njt.vezbe08.domain.Katedra</class>
    <class>rs.ac.bg.fon.njt.vezbe08.domain.Zvanje</class>
    <class>rs.ac.bg.fon.njt.vezbe08.domain.Predavac</class>

    <properties>
        <property name="jakarta.persistence.jdbc.url"
                  value="jdbc:mysql://localhost:3306/njt_hibernate?zeroDateTimeBehavior=CONVERT_TO_NULL"/>
        <!-- zeroDateTimeBehavior=CONVERT_TO_NULL: MySQL "nulti datumi" (0000-00-00)
             se konvertuju u null umesto da bace izuzetak -->
        <property name="jakarta.persistence.jdbc.user" value="root"/>
        <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="jakarta.persistence.jdbc.password" value="root"/>
        <property name="jakarta.persistence.schema-generation.database.action" value="create"/>
        <!-- "create" → JPA automatski kreira tabele pri prvom pokretanju
             Ostale opcije: "drop-and-create", "drop", "none" -->
    </properties>
</persistence-unit>
```

### 8.2 hibernate.cfg.xml

```xml
<session-factory>
    <property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>
    <!-- Dialect govori Hibernate-u "sintaksa kojeg SQL-a da generiše" -->

    <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
    <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/njt_hibernate</property>
    <property name="hibernate.connection.username">root</property>
    <property name="hibernate.connection.password">root</property>

    <property name="hibernate.show_sql">true</property>
    <!-- Ispisuje generisani SQL u konzolu — korisno za debagovanje -->
    <property name="hibernate.format_sql">true</property>
    <!-- Lepo formatira SQL -->

    <!-- Sve klase entiteta moraju biti registrovane -->
    <mapping class="rs.ac.bg.fon.njt.vezbe08.domain.Katedra"/>
    <mapping class="rs.ac.bg.fon.njt.vezbe08.domain.Predavac"/>
    <mapping class="rs.ac.bg.fon.njt.vezbe08.domain.Zvanje"/>
</session-factory>
```

### 8.3 pom.xml (Maven zavisnosti)

```xml
<!-- Spring IoC kontejner -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>7.0.7</version>
</dependency>

<!-- MySQL JDBC drajver -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>9.7.0</version>
</dependency>

<!-- Spring JDBC (JdbcTemplate) -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>7.0.7</version>
</dependency>

<!-- Hibernate ORM (i implementacija JPA specifikacije) -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>6.5.2.Final</version>
</dependency>

<!-- JPA API (interfejsi/anotacije) -->
<dependency>
    <groupId>jakarta.persistence</groupId>
    <artifactId>jakarta.persistence-api</artifactId>
    <version>3.1.0</version>
</dependency>

<!-- EclipseLink JPA provider (alternativa Hibernate-u za JPA deo) -->
<dependency>
    <groupId>org.eclipse.persistence</groupId>
    <artifactId>org.eclipse.persistence.jpa</artifactId>
    <version>4.0.2</version>
</dependency>
```

---

## 9. Glavna klasa — Vezbe08.java

```java
@Component
// ↑ Spring će registrovati ovu klasu kao bean (zbog @ComponentScan u AppConfig-u)
public class Vezbe08 {

    private final PredavacService predavacJpaService;
    private PredavacService predavacSpringJdbcService;
    private PredavacService predavacHibernateOrmService;

    @Autowired
    public Vezbe08(
            @Qualifier("JpaPredavacService") PredavacService predavacJpaService,
            @Qualifier("SpringJdbcPredavacService") PredavacService predavacSpringJdbcService,
            @Qualifier("HibernatePredavacService") PredavacService predavacHibernateOrmService) {
        // Spring injektuje sva tri servisa korišćenjem @Qualifier koji tačno navodi ime bean-a
        this.predavacJpaService = predavacJpaService;
        this.predavacSpringJdbcService = predavacSpringJdbcService;
        this.predavacHibernateOrmService = predavacHibernateOrmService;
    }

    public static void main(String[] args) {
        // Pokretanje Spring IoC kontejnera — on čita AppConfig klasu
        // i kreira sve bean-ove (DataSource, JdbcTemplate, SessionFactory, itd.)
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

        // Uzimamo bean tipa Vezbe08 iz kontejnera
        // (Spring ga je automatski kreirao zbog @Component)
        Vezbe08 app = context.getBean(Vezbe08.class);

        // Kreiramo test podatak
        PredavacDto dto = new PredavacDto();
        dto.setImePrezime("Tijana Balotic");
        dto.setDatumRodjenja(LocalDate.of(2004, Month.AUGUST, 3));
        dto.setKatedra(new Katedra(1L, "softversko inzenjerstvo", "si"));
        dto.setZvanje(new Zvanje(1L, "profesor", "professor"));
        // Napomena: Katedra i Zvanje sa ID=1 moraju već postojati u bazi!

        // Čitanje sa standardnog ulaza
        Scanner sc = new Scanner(System.in);
        System.out.println("Unesite broj: 1-jpa, 2-spring jdbc, 3-hibernate: ");
        String input = sc.next().trim();

        if (input.equals("1")) {
            app.savePredavacJpa(dto);
        } else if (input.equals("2")) {
            app.savePredavacSpringJdbc(dto);
        } else if (input.equals("3")) {
            app.savePredavacHibernateOrm(dto);
        }
    }

    private void savePredavacJpa(PredavacDto dto) {
        predavacJpaService.save(dto);
    }

    private void savePredavacSpringJdbc(PredavacDto dto) {
        predavacSpringJdbcService.save(dto);
    }

    private void savePredavacHibernateOrm(PredavacDto dto) {
        predavacHibernateOrmService.save(dto);
    }
}
```

---

## 10. Tok izvršavanja programa

Kada korisnik unese npr. `1` (JPA), tok je sledeći:

```
main()
  │
  ├─ new AnnotationConfigApplicationContext(AppConfig.class)
  │     Kreira: DataSource, JdbcTemplate, SessionFactory,
  │             EntityManagerFactory, sva tri Service bean-a,
  │             sva tri Repository bean-a, Vezbe08 bean
  │
  ├─ context.getBean(Vezbe08.class)
  │     Spring vraća već kreiran Vezbe08 objekat sa injektovanim servisima
  │
  ├─ Kreiranje PredavacDto (test podaci)
  │
  ├─ Korisnik unosi "1"
  │
  └─ app.savePredavacJpa(dto)
        │
        └─ predavacJpaService.save(dto)          [PredavacService interfejs]
              │
              └─ PredavacServiceImpl.save(dto)
                    │
                    ├─ Konverzija: dto → new Predavac(...)
                    │
                    └─ predavacRepository.save(predavac)   [PredavacRepository interfejs]
                          │
                          └─ PredavacJpaRepository.save(predavac)
                                │
                                ├─ emf.createEntityManager()
                                ├─ em.getTransaction().begin()
                                ├─ em.merge(predavac)  → SQL: INSERT INTO predavac(...)
                                ├─ em.getTransaction().commit()
                                └─ em.close()
```

---

## 11. Šema baze podataka

SQL za kreiranje tabela (ako ne koristiš JPA auto-create):

```sql
CREATE DATABASE njt_hibernate;
USE njt_hibernate;

CREATE TABLE zvanje (
    idzvanje       BIGINT PRIMARY KEY,
    srpskiprevod   VARCHAR(100),
    engleskiprevod VARCHAR(100)
);

CREATE TABLE katedra (
    idkatedra     BIGINT PRIMARY KEY,
    naziv         VARCHAR(200),
    skraceninaziv VARCHAR(50)
);

CREATE TABLE predavac (
    idpredavac    BIGINT PRIMARY KEY AUTO_INCREMENT,
    imeprezime    VARCHAR(200),
    datumrodjenja DATE,
    katedraid     BIGINT,
    zvanjeid      BIGINT,
    FOREIGN KEY (katedraid) REFERENCES katedra(idkatedra),
    FOREIGN KEY (zvanjeid)  REFERENCES zvanje(idzvanje)
);

-- Test podaci (moraju postojati pre pokretanja programa)
INSERT INTO zvanje VALUES (1, 'profesor', 'professor');
INSERT INTO katedra VALUES (1, 'softversko inzenjerstvo', 'si');
```

---

## 12. Poređenje tehnologija

| Aspekt | JPA (EclipseLink) | Spring JDBC | Hibernate ORM |
|--------|-------------------|-------------|---------------|
| **Nivo apstrakcije** | Visok — ne pišeš SQL | Nizak — pišeš SQL ručno | Visok — ne pišeš SQL |
| **SQL kontrola** | Automatski generiše | Potpuna kontrola | Automatski generiše |
| **Konfiguracija** | `persistence.xml` | `DataSource` bean | `hibernate.cfg.xml` |
| **Ključna klasa** | `EntityManager` | `JdbcTemplate` | `Session` |
| **Transakcije** | `em.getTransaction()` | Automatski u `update()` | `session.beginTransaction()` |
| **Pogodno za** | Složene objektne modele | Jednostavne upite, performanse | Slično JPA, Hibernate-specifične funkcije |
| **Odnos prema JPA** | JPA API + EclipseLink impl. | Nezavisan od JPA | Implementira JPA + dodaje sopstveni API |

### Ista stvar, tri načina

```java
// JPA
em.merge(predavac);

// Spring JDBC
jdbcTemplate.update("INSERT INTO predavac(...) VALUES (?,?,?,?)", ...);

// Hibernate
session.persist(predavac);
```

Sve tri linije koda rade isto — čuvaju predavača u bazu. Razlika je u tome ko generiše SQL i koliko kontrole imaš nad tim procesom.

---

> **Napomena za pokretanje:**  
> Pre pokretanja programa, u MySQL bazi `njt_hibernate` moraju postojati tabele i redovi za `katedra` (ID=1) i `zvanje` (ID=1). JPA sa opcijom `schema-generation.database.action=create` će automatski kreirati tabele, ali ne i testne podatke.
