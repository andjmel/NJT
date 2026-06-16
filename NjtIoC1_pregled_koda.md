# NjtIoC1 – Pregled koda (redosled izgradnje projekta)

---

## 1. Domain entiteti

### `Lekar.java`
```java
@Entity
@Table(name = "lekar")
public class Lekar {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idlekar;

    @Column(name = "ime")
    private String ime;

    @Column
    private String prezime;

    @Column
    private String specijalizacija;

    @Column
    private String brojlicence;

    public Lekar() {}

    public Lekar(Long idlekar) {
        this.idlekar = idlekar;
    }
}
```

### `Pacijent.java`
```java
@Entity
@Table(name = "pacijent")
public class Pacijent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idpacijent;

    @Column
    private String ime;

    @Column
    private String prezime;

    @Column
    private int godinarodjenja;

    public Pacijent() {}

    public Pacijent(Long idpacijent) {
        this.idpacijent = idpacijent;
    }
}
```

### `MedicalExamination.java`
```java
@Entity
@Table(name = "medical_examination")
public class MedicalExamination {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idpregled;

    @Column
    private LocalDate datumizvrsenja;

    @Column
    private String opis;

    @Column
    private double cena;

    @ManyToOne
    @JoinColumn(name = "idlekar", nullable = false)
    private Lekar lekar;

    @ManyToOne
    @JoinColumn(name = "idpacijent", nullable = true)
    private Pacijent pacijent;

    public MedicalExamination() {}
}
```

---

## 2. Izuzetak

### `MedicalExaminationDoesNotExistException.java`
```java
public class MedicalExaminationDoesNotExistException extends Exception {

    public MedicalExaminationDoesNotExistException(String message) {
        super(message);
    }
}
```

---

## 3. DTO

### `MedicalExaminationDto.java`
```java
public class MedicalExaminationDto {

    private Long idpregled;
    private LocalDate datumizvrsenja;
    private String opis;
    private double cena;
    private Long idlekar;
    private Long idpacijent;

    public MedicalExaminationDto() {}
}
```

### `Converter.java` (interfejs)
```java
public interface Converter<Dto, Entity> {
    Entity toEntity(Dto dto);
    Dto toDto(Entity entity);
}
```

### `MedicalExaminationConverter.java`
```java
@Component
public class MedicalExaminationConverter implements Converter<MedicalExaminationDto, MedicalExamination> {

    @Override
    public MedicalExamination toEntity(MedicalExaminationDto dto) {
        MedicalExamination m = new MedicalExamination();
        m.setCena(dto.getCena());
        m.setDatumizvrsenja(dto.getDatumizvrsenja());
        m.setIdpregled(dto.getIdpregled());
        if (dto.getIdlekar() != null) m.setLekar(new Lekar(dto.getIdlekar()));
        if (dto.getIdpacijent() != null) m.setPacijent(new Pacijent(dto.getIdpacijent()));
        return m;
    }

    @Override
    public MedicalExaminationDto toDto(MedicalExamination entity) {
        MedicalExaminationDto m = new MedicalExaminationDto();
        m.setCena(entity.getCena());
        m.setDatumizvrsenja(entity.getDatumizvrsenja());
        m.setIdpregled(entity.getIdpregled());
        if (entity.getLekar() != null) m.setIdlekar(entity.getLekar().getIdlekar());
        if (entity.getPacijent() != null) m.setIdpacijent(entity.getPacijent().getIdpacijent());
        return m;
    }
}
```

---

## 4. Repozitorijum

### `MedicalExaminationRepo.java` (interfejs)
```java
public interface MedicalExaminationRepo {
    void save(MedicalExamination e);
    void delete(MedicalExamination e);
}
```

### `JpaRepo.java`
```java
@Repository(value = "jpa_repo")
public class JpaRepo implements MedicalExaminationRepo {

    private final EntityManagerFactory emf;

    @Autowired
    public JpaRepo(@Qualifier("emf") EntityManagerFactory emf) {
        this.emf = emf;
    }

    @Override
    public void save(MedicalExamination e) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        em.merge(e);
        em.getTransaction().commit();
        em.close();
    }

    @Override
    public void delete(MedicalExamination e) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        MedicalExamination toRemove = em.find(MedicalExamination.class, e.getIdpregled());
        if (toRemove != null) em.remove(toRemove);
        em.getTransaction().commit();
        em.close();
    }
}
```

### `SpringJdbcRepo.java`
```java
@Repository(value = "spring_repo")
public class SpringJdbcRepo implements MedicalExaminationRepo {

    private final JdbcTemplate jdbc;

    @Autowired
    public SpringJdbcRepo(@Qualifier("jdbc") JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    @Override
    public void save(MedicalExamination e) {
        jdbc.update(
            "INSERT INTO medical_examination (datumizvrsenja, opis, cena, idlekar, idpacijent) VALUES (?, ?, ?, ?, ?)",
            e.getDatumizvrsenja(),
            e.getOpis(),
            e.getCena(),
            e.getLekar().getIdlekar(),
            e.getPacijent() != null ? e.getPacijent().getIdpacijent() : null
        );
    }

    @Override
    public void delete(MedicalExamination e) {
        jdbc.update("DELETE FROM medical_examination WHERE idpregled = ?", e.getIdpregled());
    }
}
```

---

## 5. Servis

### `MedicalExaminationService.java` (interfejs)
```java
public interface MedicalExaminationService {
    MedicalExaminationDto save(MedicalExamination e);
    void delete(MedicalExaminationDto e) throws MedicalExaminationDoesNotExistException;
}
```

### `MedicalExaminationServiceImpl.java`
```java
public class MedicalExaminationServiceImpl implements MedicalExaminationService {

    private final MedicalExaminationConverter conv;
    private final MedicalExaminationRepo repo;

    public MedicalExaminationServiceImpl(MedicalExaminationRepo repo, MedicalExaminationConverter conv) {
        this.conv = conv;
        this.repo = repo;
    }

    @Override
    public MedicalExaminationDto save(MedicalExamination e) {
        repo.save(e);
        return conv.toDto(e);
    }

    @Override
    public void delete(MedicalExaminationDto e) throws MedicalExaminationDoesNotExistException {
        MedicalExamination entity = conv.toEntity(e);
        if ((entity == null) || (entity.getOpis() == null) || entity.getDatumizvrsenja() == null)
            throw new MedicalExaminationDoesNotExistException("Ne postoji pregled!");
        repo.delete(entity);
    }
}
```

---

## 6. Konfiguracija Spring IoC kontejnera

### `persistence.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="3.1" xmlns="https://jakarta.ee/xml/ns/persistence" ...>
  <persistence-unit name="MedicalPU" transaction-type="RESOURCE_LOCAL">
    <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
    <class>com.andjmel.njtioc1.domain.Pacijent</class>
    <class>com.andjmel.njtioc1.domain.Lekar</class>
    <class>com.andjmel.njtioc1.domain.MedicalExamination</class>
    <properties>
      <property name="jakarta.persistence.jdbc.url" value="jdbc:mysql://localhost:3306/njt?zeroDateTimeBehavior=CONVERT_TO_NULL"/>
      <property name="jakarta.persistence.jdbc.user" value="root"/>
      <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
      <property name="jakarta.persistence.jdbc.password" value=""/>
      <property name="jakarta.persistence.schema-generation.database.action" value="create"/>
    </properties>
  </persistence-unit>
</persistence>
```

### `AppConfig.java`
```java
@Configuration
@ComponentScan(basePackages = {"com.andjmel.njtioc1"})
public class AppConfig {

    @Bean("emf")
    public EntityManagerFactory getEntityManagerFactory() {
        return Persistence.createEntityManagerFactory("MedicalPU");
    }

    @Bean("jpa_service")
    public MedicalExaminationService getJpaService(
            @Qualifier("jpa_repo") MedicalExaminationRepo repo,
            MedicalExaminationConverter conv) {
        return new MedicalExaminationServiceImpl(repo, conv);
    }

    @Bean("spring_service")
    public MedicalExaminationService getSpringService(
            @Qualifier("spring_repo") MedicalExaminationRepo repo,
            MedicalExaminationConverter conv) {
        return new MedicalExaminationServiceImpl(repo, conv);
    }

    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource source = new DriverManagerDataSource();
        source.setUrl("jdbc:mysql://localhost:3306/njt");
        source.setUsername("root");
        source.setPassword("");
        return source;
    }

    @Bean("jdbc")
    public JdbcTemplate getJdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

---

## 7. Glavna klasa (Application)

### `NjtIoC1.java`
```java
@Component
public class NjtIoC1 {

    private MedicalExaminationService jpa;
    private MedicalExaminationService spring;

    @Autowired
    public NjtIoC1(
            @Qualifier("jpa_service") MedicalExaminationService jpa,
            @Qualifier("spring_service") MedicalExaminationService spring) {
        this.jpa = jpa;
        this.spring = spring;
    }

    public static void main(String[] args) {
        ApplicationContext ioc = new AnnotationConfigApplicationContext(AppConfig.class);
        NjtIoC1 app = ioc.getBean(NjtIoC1.class);

        MedicalExamination me = new MedicalExamination();
        me.setIdpregled(1L);
        me.setOpis("Pregled");
        me.setCena(500.0);
        me.setDatumizvrsenja(LocalDate.now());
        me.setLekar(new Lekar(1L));
        me.setPacijent(new Pacijent(1L));

        MedicalExaminationDto dtoZaBrisanje = new MedicalExaminationDto();
        dtoZaBrisanje.setIdpregled(2L);

        System.out.println("Unesite broj: 1-jpa save  2-jpa delete  3-spring jdbc save  4-spring jdbc delete:");
        Scanner sc = new Scanner(System.in);
        String input = sc.next().trim();

        try {
            if (input.equals("1")) app.saveJpa(me);
            else if (input.equals("2")) app.deleteJpa(dtoZaBrisanje);
            else if (input.equals("3")) app.saveSpringJdbc(me);
            else if (input.equals("4")) app.deleteSpringJdbc(dtoZaBrisanje);
        } catch (MedicalExaminationDoesNotExistException ex) {
            Logger.getLogger(NjtIoC1.class.getName()).log(Level.SEVERE, ex.getMessage());
        }
    }

    private void saveJpa(MedicalExamination e) { jpa.save(e); }
    private void deleteJpa(MedicalExaminationDto e) throws MedicalExaminationDoesNotExistException { jpa.delete(e); }
    private void saveSpringJdbc(MedicalExamination e) { spring.save(e); }
    private void deleteSpringJdbc(MedicalExaminationDto e) throws MedicalExaminationDoesNotExistException { spring.delete(e); }
}
```

---

## 8. DiRandom projekat (zadatak 7)

### `Ioc.java`
```java
@ComponentScan(basePackages = {"rs.fon.dirandom"})
public class Ioc {

    @Bean("random")
    public int getRandom() {
        Random r = new Random();
        return r.nextInt(100) + 1;
    }
}
```

### `DiRandomApp.java`
```java
@Component
public class DiRandomApp {

    private final int random;

    @Autowired
    public DiRandomApp(@Qualifier("random") int random) {
        this.random = random;
    }

    public int getRandom() {
        return random;
    }
}
```

### `DiRandom.java`
```java
public class DiRandom {

    public static void main(String[] args) {
        ApplicationContext ioc = new AnnotationConfigApplicationContext(Ioc.class);
        DiRandomApp app = ioc.getBean(DiRandomApp.class);
        System.out.println("Random broj je " + app.getRandom());
    }
}
```

---

---

# Moje mišljenje – šta bih izmenila

Generalno, projekat je jako dobro urađen i pokriven u celosti. Sve klase su na mestu, DI je korektno implementiran sa `@Qualifier`-ima, layering je čist. Evo detalja:

## ✅ Šta je odlično urađeno

- Konstruktorski DI svuda (ne field injection) – to je best practice.
- `@Qualifier` na svim mestima gde postoji višestruki bean istog tipa – nema ambiguity problema.
- `nullable = true` na `@JoinColumn` za pacijenta – prati zahtev.
- `nullable = false` na lekar – ispravno.
- Null-check u `toEntity`/`toDto` konverteru za opcione asocijacije.
- `MedicalExaminationServiceImpl` nije anotiran kao `@Component` – ispravno, jer se kreira ručno u AppConfig-u kao dva odvojena beana.
- `persistence.xml` koristi JPA 3.x namespace (`jakarta.persistence`) i EclipseLink – konzistentno.
- DiRandom projekat je minimalan i tačan.

## ⚠️ Šta bih izmenila ili dodala

### 1. `delete` logika u servisu je pogrešna ❌
```java
// Trenutni kod:
MedicalExamination entity = conv.toEntity(e);
if ((entity == null) || (entity.getOpis() == null) || entity.getDatumizvrsenja() == null)
    throw new MedicalExaminationDoesNotExistException("Ne postoji pregled!");
```
Problem: `toEntity` **nikad ne vraća null** (uvek pravi novi objekat), a `opis` i `datumizvrsenja` su `null` jer se Dto za brisanje pravi samo sa `idpregled`. Dakle, `delete` **uvek baca izuzetak** i **nikad ne briše ništa**!

Ispravna logika bi bila da se proveri da li entitet postoji u bazi pre brisanja, ili da se provera zasniva samo na `idpregled`:
```java
@Override
public void delete(MedicalExaminationDto e) throws MedicalExaminationDoesNotExistException {
    if (e == null || e.getIdpregled() == null)
        throw new MedicalExaminationDoesNotExistException("Ne postoji pregled!");
    MedicalExamination entity = conv.toEntity(e);
    repo.delete(entity);
}
```
Tada bi repozitorijum (posebno JpaRepo) bio zadužen da proveri `if (toRemove != null)` – što već radi ispravno.

### 2. `JpaRepo.save()` koristi `merge` umesto `persist`
`em.merge(e)` je OK za update, ali za novi entitet koji ima `idpregled` već postavljen ručno (kao u main-u: `me.setIdpregled(1L)`) može da napravi problem – JPA će pokušati update reda koji možda ne postoji. Bolje rešenje bi bilo da se `idpregled` ne setuje ručno, ili da se eksplicitno koristi `persist` za nove entitete i `merge` za update.

### 3. `Pacijent` nema polje `korisnickoime` / `sifra`
Zadatak kaže da pacijent pristupa sistemu **preko korisničkog naloga** (korisničko ime i šifra), a u zahtevu 1 piše "Ne svaki pacijent otvorenom korisničkim nalog". To znači da ta polja treba da postoje, ali mogu biti `null`. Trenutni `Pacijent` entitet ih nema.

### 4. `Ioc.java` u DiRandom projektu nema `@Configuration`
Klasa `Ioc` ima `@ComponentScan` ali nema `@Configuration` anotaciju. Tehnički Spring će je obraditi jer se prosleđuje direktno `AnnotationConfigApplicationContext`-u, ali eksplicitno dodavanje `@Configuration` je good practice i čini nameru jasnijom:
```java
@Configuration
@ComponentScan(basePackages = {"rs.fon.dirandom"})
public class Ioc { ... }
```

### 5. `MedicalExaminationDto` ne prenosi `opis` kroz konverter
U `MedicalExaminationConverter.toDto()` i `toEntity()` se ne mapira polje `opis`, iako oba objekta (`MedicalExamination` i `MedicalExaminationDto`) imaju to polje. Treba dodati:
```java
m.setOpis(dto.getOpis()); // u toEntity
m.setOpis(entity.getOpis()); // u toDto
```

---

**Zaključak:** Struktura projekta je solidna, DI je ispravno primenjen, i jasno je da razumeš kako Spring IoC funkcioniše. Ključna greška je u logici `delete` u servisu (uvek baca exception) i izostavljanje `opis` iz konvertera. Sve ostalo su sitne napomene.
