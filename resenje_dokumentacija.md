# Dokumentacija – NjtIoC1 & DiTime (Spring IoC/DI)

---

## PROJEKAT 1: NjtIoC1 – PrviZadatak

---

## Korak 1 – `pom.xml`

Definiše sve Maven zavisnosti koje projekat koristi: JPA API, EclipseLink (JPA provider), MySQL konektor, Spring Context i Spring JDBC.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.andjmel</groupId>
    <artifactId>PrviZadatak</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.release>21</maven.compiler.release>
        <exec.mainClass>com.andjmel.prvizadatak.PrviZadatak</exec.mainClass>
    </properties>

    <dependencies>

        <!-- JPA API (Jakarta) -->
        <dependency>
            <groupId>jakarta.persistence</groupId>
            <artifactId>jakarta.persistence-api</artifactId>
            <version>3.1.0</version>
        </dependency>

        <!-- EclipseLink – JPA implementacija -->
        <dependency>
            <groupId>org.eclipse.persistence</groupId>
            <artifactId>org.eclipse.persistence.jpa</artifactId>
            <version>4.0.2</version>
        </dependency>

        <!-- MySQL konektor -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>9.7.0</version>
            <scope>compile</scope>
        </dependency>

        <!-- Spring Context (IoC/DI) -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>7.0.7</version>
            <scope>compile</scope>
        </dependency>

        <!-- Spring JDBC (JdbcTemplate) -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>7.0.7</version>
            <scope>compile</scope>
        </dependency>

    </dependencies>
</project>
```

---

## Korak 2 – Domain (entiteti)

Entiteti su JPA klase koje se mapiraju na tabele u bazi. Svaka klasa je anotirana sa `@Entity`.

### `Korisnik.java`

Predstavlja korisnički nalog pisca (email + šifra). Svaki pisac ima tačno jedan korisnički nalog.

```java
package com.andjmel.prvizadatak.domain;

import jakarta.persistence.*;
import java.io.Serializable;

@Entity
public class Korisnik implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column
    private Long idkorisnik;

    @Column
    private String sifra;

    @Column
    private String email;

    public Korisnik() {}

    // getteri i setteri...
}
```

> **Objašnjenje:** `@Entity` → JPA tabela. `@Id` + `@GeneratedValue` → auto-increment primarni ključ. `email` i `sifra` su kredencijali za prijavu.

---

### `Autor.java`

Predstavlja pisca. Ima vezu 1:1 ka `Korisnik` (svaki autor ima jedan nalog).

```java
package com.andjmel.prvizadatak.domain;

import jakarta.persistence.*;
import java.io.Serializable;

@Entity
public class Autor implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column
    private Long idautor;

    @Column
    private String ime;

    @Column
    private String prezime;

    @Column
    private Integer godiste;

    @OneToOne
    @JoinColumn(name = "idkorisnik")
    private Korisnik korisnik;

    public Autor() {}

    public Autor(Long idautor) {
        this.idautor = idautor;
    }

    // getteri i setteri...
}
```

> **Objašnjenje:** `@OneToOne` → veza jedan-na-jedan sa `Korisnik`. `@JoinColumn(name = "idkorisnik")` → strani ključ u tabeli `autor`. Konstruktor sa `idautor` koristi se u konverteru kada treba samo referenca (bez punih podataka).

---

### `Izdavac.java`

Predstavlja izdavačku kuću.

```java
package com.andjmel.prvizadatak.domain;

import jakarta.persistence.*;
import java.io.Serializable;

@Entity
public class Izdavac implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column
    private Long idizdavac;

    @Column
    private String naziv;

    @Column
    private String pib;

    @Column
    private String maticnibroj;

    @Column
    private String srediste;

    public Izdavac() {}

    public Izdavac(Long idizdavac) {
        this.idizdavac = idizdavac;
    }

    // getteri i setteri...
}
```

> **Objašnjenje:** Polja `pib`, `maticnibroj` i `srediste` odgovaraju zahtevima zadatka. Konstruktor sa `idizdavac` koristi se za postavljanje FK reference bez učitavanja celog objekta.

---

### `Knjiga.java`

Centralni entitet. Ima ManyToOne veze ka `Autor` i `Izdavac`.

```java
package com.andjmel.prvizadatak.domain;

import jakarta.persistence.*;
import java.io.Serializable;
import java.time.LocalDate;

@Entity
public class Knjiga implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idknjiga;

    @Column
    private String naziv;

    @Column
    private LocalDate datumizdavanja;

    @Column
    private Integer tiraz;

    @ManyToOne
    @JoinColumn(name = "idautor")
    private Autor autor;

    @ManyToOne
    @JoinColumn(name = "idizdavac")
    private Izdavac izdavac;

    public Knjiga() {}

    public Knjiga(Long idknjiga, String naziv, LocalDate datumizdavanja,
                  Integer tiraz, Autor autor, Izdavac izdavac) {
        this.idknjiga = idknjiga;
        this.naziv = naziv;
        this.datumizdavanja = datumizdavanja;
        this.tiraz = tiraz;
        this.autor = autor;
        this.izdavac = izdavac;
    }

    // getteri i setteri...
}
```

> **Objašnjenje:** `@ManyToOne` → više knjiga može imati istog autora / istog izdavača. `@JoinColumn` definiše ime stranog ključa u tabeli. `LocalDate` se koristi za datum izdavanja jer JPA/EclipseLink to podržava direktno.

---

## Korak 3 – DTO sloj

DTO (Data Transfer Object) se koristi da se ne izlažu direktno entiteti, već samo potrebni podaci.

### `Converter.java` (interfejs)

Generički interfejs za konverziju između DTO i entiteta.

```java
package com.andjmel.prvizadatak.dto;

public interface Converter<Dto, Entity> {
    Entity toEntity(Dto dto);
    Dto toDto(Entity entity);
}
```

> **Objašnjenje:** Parametrizovani interfejs – `Dto` je tip DTO klase, `Entity` je tip entiteta. Svaka konkretna konverzija implementira ovaj interfejs.

---

### `KnjigaDto.java`

Prenos podataka o knjizi između slojeva – ne sadrži JPA anotacije.

```java
package com.andjmel.prvizadatak.dto;

import java.time.LocalDate;

public class KnjigaDto {

    private Long idknjiga;
    private String naziv;
    private LocalDate datumizdavanja;
    private Integer tiraz;
    private Long izdavacId;
    private Long autorId;

    public KnjigaDto() {}

    public KnjigaDto(String naziv, LocalDate datumizdavanja, Integer tiraz) {
        this.naziv = naziv;
        this.datumizdavanja = datumizdavanja;
        this.tiraz = tiraz;
    }

    // getteri i setteri...
}
```

> **Objašnjenje:** Umjesto da DTO nosi cijele objekte `Autor` i `Izdavac`, nosi samo njihove ID-eve (`autorId`, `izdavacId`). Ovo sprječava nepotrebno učitavanje relacionih podataka.

---

### `KnjigaConverter.java`

Implementacija `Converter` interfejsa za konverziju `KnjigaDto ↔ Knjiga`.

```java
package com.andjmel.prvizadatak.dto;

import com.andjmel.prvizadatak.domain.Autor;
import com.andjmel.prvizadatak.domain.Izdavac;
import com.andjmel.prvizadatak.domain.Knjiga;
import org.springframework.stereotype.Component;

@Component("conv")
public class KnjigaConverter implements Converter<KnjigaDto, Knjiga> {

    @Override
    public Knjiga toEntity(KnjigaDto dto) {
        Knjiga k = new Knjiga();
        k.setDatumizdavanja(dto.getDatumizdavanja());
        k.setIdknjiga(dto.getIdknjiga());
        k.setNaziv(dto.getNaziv());
        k.setTiraz(dto.getTiraz());
        if (dto.getIzdavacId() != null) {
            k.setIzdavac(new Izdavac(dto.getIzdavacId()));
        }
        if (dto.getautorId() != null) {
            k.setAutor(new Autor(dto.getautorId()));
        }
        return k;
    }

    @Override
    public KnjigaDto toDto(Knjiga entity) {
        KnjigaDto k = new KnjigaDto();
        k.setDatumizdavanja(entity.getDatumizdavanja());
        k.setIdknjiga(entity.getIdknjiga());
        k.setNaziv(entity.getNaziv());
        k.setTiraz(entity.getTiraz());
        if (entity.getIzdavac() != null) {
            k.setIzdavacId(entity.getIzdavac().getIdizdavac());
        }
        if (entity.getAutor() != null) {
            k.setautorId(entity.getAutor().getIdautor());
        }
        return k;
    }
}
```

> **Objašnjenje:** `@Component("conv")` → Spring registruje ovu klasu kao bean sa imenom `conv`. U `toEntity`, za autor i izdavač se kreiraju objekti samo sa ID-em (tzv. proxy referenca) jer JPA to prepoznaje pri `merge()`. Ovako se izbjegava slanje punih podataka koji JPA ionako ne bi upotrijebio.

---

## Korak 4 – Izuzeci

### `BookExistException.java`

Baca se kada pokušamo snimiti knjigu koja već postoji.

```java
package com.andjmel.prvizadatak.exception;

public class BookExistException extends Exception {

    public BookExistException(String message) {
        super(message);
    }
}
```

---

### `BookDoesntExistException.java`

Baca se kada pokušamo obrisati knjigu koja ne postoji.

```java
package com.andjmel.prvizadatak.exception;

public class BookDoesntExistException extends Exception {

    public BookDoesntExistException(String message) {
        super(message);
    }
}
```

> **Objašnjenje:** Oba izuzetka nasljeđuju `Exception` (checked exceptions) – znači pozivajući kod mora eksplicitno da ih hvata ili deklarira. Ovo je u skladu sa zahtjevima zadatka.

---

## Korak 5 – Repozitorijum

### `KnjigaRep.java` (interfejs)

Interfejs koji definiše ugovor za sve implementacije repozitorijuma.

```java
package com.andjmel.prvizadatak.repository;

import com.andjmel.prvizadatak.domain.Knjiga;

public interface KnjigaRep {
    Knjiga save(Knjiga k);
    void delete(Knjiga k);
}
```

> **Objašnjenje:** Princip programiranja prema interfejsu – servis ne zna koja je konkretna implementacija, samo zna da postoji `save` i `delete`.

---

### `JpaRep.java` (JPA implementacija)

Koristi `EntityManagerFactory` za direktnu JPA komunikaciju s bazom.

```java
package com.andjmel.prvizadatak.repository.impl;

import com.andjmel.prvizadatak.domain.Knjiga;
import com.andjmel.prvizadatak.repository.KnjigaRep;
import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Repository;

@Repository(value = "jpa_repo")
public class JpaRep implements KnjigaRep {

    private EntityManagerFactory emf;

    @Autowired
    public JpaRep(@Qualifier("emf") EntityManagerFactory emf) {
        this.emf = emf;
    }

    @Override
    public Knjiga save(Knjiga k) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        em.merge(k);
        em.getTransaction().commit();
        em.close();
        return k;
    }

    @Override
    public void delete(Knjiga k) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        Knjiga toRemove = em.find(Knjiga.class, k.getIdknjiga());
        if (toRemove != null) {
            em.remove(toRemove);
        }
        em.getTransaction().commit();
        em.close();
    }
}
```

> **Objašnjenje:** `@Repository("jpa_repo")` → Spring bean sa imenom `jpa_repo`. `em.merge(k)` se koristi umjesto `persist` jer knjiga može imati već postavljene FK reference (Autor, Izdavac) koji postoje u bazi – `merge` ih prepoznaje. `em.find` je potreban przed `remove` jer objekat mora biti "managed" (priključen u kontekstu) da bi se obrisao.

---

### `SpringRep.java` (Spring JDBC implementacija)

Koristi `JdbcTemplate` za direktno izvršavanje SQL upita.

```java
package com.andjmel.prvizadatak.repository.impl;

import com.andjmel.prvizadatak.domain.Knjiga;
import com.andjmel.prvizadatak.repository.KnjigaRep;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

@Repository(value = "spring_repo")
public class SpringRep implements KnjigaRep {

    private JdbcTemplate jdbcTemplate;

    @Autowired
    public SpringRep(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Knjiga save(Knjiga k) {
        jdbcTemplate.update(
            "insert into knjiga(naziv, datumizdavanja) values (?,?)",
            k.getNaziv(), k.getDatumizdavanja()
        );
        return k;
    }

    @Override
    public void delete(Knjiga k) {
        jdbcTemplate.update(
            "delete from knjiga where idknjiga = ?",
            k.getIdknjiga()
        );
    }
}
```

> **Objašnjenje:** `@Repository("spring_repo")` → drugi Spring bean za repozitorijum. `JdbcTemplate.update()` se koristi za INSERT, UPDATE i DELETE. Parametri se prosleđuju kao varargs – Spring JDBC automatski sprječava SQL injection. Ovo je jednostavnija alternativa JPA-u (nema upravljanja kontekstom).

---

## Korak 6 – Servis

### `KnjigaService.java` (interfejs)

Definiše poslovnu logiku nad knjigama.

```java
package com.andjmel.prvizadatak.service;

import com.andjmel.prvizadatak.dto.KnjigaDto;
import com.andjmel.prvizadatak.exception.BookDoesntExistException;
import com.andjmel.prvizadatak.exception.BookExistException;

public interface KnjigaService {
    void delete(KnjigaDto k) throws BookDoesntExistException;
    KnjigaDto save(KnjigaDto k) throws BookExistException;
}
```

---

### `ServiceImpl.java` (implementacija servisa)

Sadrži poslovnu logiku – provjera izuzetaka, konverzija DTO ↔ entitet, poziv repozitorijuma.

```java
package com.andjmel.prvizadatak.service.impl;

import com.andjmel.prvizadatak.domain.Knjiga;
import com.andjmel.prvizadatak.dto.KnjigaConverter;
import com.andjmel.prvizadatak.dto.KnjigaDto;
import com.andjmel.prvizadatak.exception.BookDoesntExistException;
import com.andjmel.prvizadatak.exception.BookExistException;
import com.andjmel.prvizadatak.repository.KnjigaRep;
import com.andjmel.prvizadatak.service.KnjigaService;
import org.springframework.beans.factory.annotation.Autowired;

public class ServiceImpl implements KnjigaService {

    private final KnjigaRep repo;
    private final KnjigaConverter conv;

    @Autowired
    public ServiceImpl(KnjigaRep repo, KnjigaConverter conv) {
        this.repo = repo;
        this.conv = conv;
    }

    @Override
    public void delete(KnjigaDto k) throws BookDoesntExistException {
        if (k.getIdknjiga() == null) {
            throw new BookDoesntExistException("Knjiga ne postoji");
        }
        Knjiga knj = conv.toEntity(k);
        repo.delete(knj);
    }

    @Override
    public KnjigaDto save(KnjigaDto k) throws BookExistException {
        if (k.getIdknjiga() != null) {
            throw new BookExistException("Knjiga već postoji u bazi!");
        }
        Knjiga knj = conv.toEntity(k);
        repo.save(knj);
        return k;
    }
}
```

> **Objašnjenje:** `ServiceImpl` nije anotiran sa `@Service` jer se instancira ručno u `AppConfig` (dva puta – jednom za JPA, jednom za Spring JDBC). Provjera za `save`: ako DTO već ima `idknjiga` → znači knjiga postoji → baca `BookExistException`. Provjera za `delete`: ako DTO nema `idknjiga` → knjiga ne postoji → baca `BookDoesntExistException`. Nakon provjere, DTO se konvertuje u entitet i prosleđuje repozitorijumu.

---

## Korak 7 – Konfiguracija Spring IoC kontejnera

### `AppConfig.java`

Centralna konfiguracija – definiše sve Spring beanove i skeniranje komponenti.

```java
package com.andjmel.prvizadatak.config;

import com.andjmel.prvizadatak.dto.KnjigaConverter;
import com.andjmel.prvizadatak.repository.KnjigaRep;
import com.andjmel.prvizadatak.service.KnjigaService;
import com.andjmel.prvizadatak.service.impl.ServiceImpl;
import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;
import javax.sql.DataSource;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

@ComponentScan(basePackages = {"com.andjmel.prvizadatak"})
public class AppConfig {

    // Bean za JPA EntityManagerFactory (učitava persistence.xml)
    @Bean("emf")
    public EntityManagerFactory getEntityManagerFactory() {
        return Persistence.createEntityManagerFactory("PrviZadatakPU");
    }

    // Bean za KnjigaService koji koristi JPA repozitorijum
    @Bean("jpa-service")
    public KnjigaService getJpaKnjigaService(
            @Qualifier("jpa_repo") KnjigaRep repo, KnjigaConverter conv) {
        return new ServiceImpl(repo, conv);
    }

    // Bean za KnjigaService koji koristi Spring JDBC repozitorijum
    @Bean("spring-service")
    public KnjigaService getSpringKnjigaService(
            @Qualifier("spring_repo") KnjigaRep repo, KnjigaConverter conv) {
        return new ServiceImpl(repo, conv);
    }

    // DataSource – konekcija ka MySQL bazi
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource source = new DriverManagerDataSource();
        source.setUrl("jdbc:mysql://localhost:3306/njt");
        source.setUsername("root");
        source.setPassword("");
        return source;
    }

    // JdbcTemplate bean koji koristi dataSource
    @Bean("jdbc")
    public JdbcTemplate getJdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

> **Objašnjenje:** `@ComponentScan` → Spring automatski pronalazi sve klase sa `@Component`, `@Repository`, `@Service` u datom paketu. `EntityManagerFactory` se kreira iz `persistence.xml` fajla. Dva servisa (`jpa-service`, `spring-service`) su iste klase (`ServiceImpl`) ali sa različitim repozitorijumima – klasičan primjer DI fleksibilnosti. `DataSource` i `JdbcTemplate` su potrebni za `SpringRep`.

---

## Korak 8 – `persistence.xml`

JPA konfiguracija – definiše persistence unit, provider i konekciju ka bazi.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="3.1"
             xmlns="https://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/persistence
             https://jakarta.ee/xml/ns/persistence/persistence_3_0.xsd">

  <persistence-unit name="PrviZadatakPU" transaction-type="RESOURCE_LOCAL">
    <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
    <class>com.andjmel.prvizadatak.domain.Knjiga</class>
    <class>com.andjmel.prvizadatak.domain.Korisnik</class>
    <class>com.andjmel.prvizadatak.domain.Izdavac</class>
    <class>com.andjmel.prvizadatak.domain.Autor</class>
    <properties>
      <property name="jakarta.persistence.jdbc.url"
                value="jdbc:mysql://localhost:3306/njt?zeroDateTimeBehavior=CONVERT_TO_NULL"/>
      <property name="jakarta.persistence.jdbc.user" value="root"/>
      <property name="jakarta.persistence.jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
      <property name="jakarta.persistence.jdbc.password" value=""/>
      <property name="jakarta.persistence.schema-generation.database.action" value="create"/>
    </properties>
  </persistence-unit>
</persistence>
```

> **Objašnjenje:** `transaction-type="RESOURCE_LOCAL"` → transakcije se kontrolišu ručno (bez JTA/application servera). `schema-generation.database.action=create` → JPA automatski kreira tabele u bazi ako ne postoje (korisno za razvoj). `zeroDateTimeBehavior=CONVERT_TO_NULL` → MySQL specifičan parametar koji konvertuje nulte datume umjesto da baci grešku.

---

## Korak 9 – Glavna klasa aplikacije

### `PrviZadatak.java`

Startna klasa – inicijalizuje Spring IoC kontejner i nudi korisniku izbor operacije.

```java
package com.andjmel.prvizadatak;

import com.andjmel.prvizadatak.config.AppConfig;
import com.andjmel.prvizadatak.dto.KnjigaDto;
import com.andjmel.prvizadatak.exception.BookDoesntExistException;
import com.andjmel.prvizadatak.exception.BookExistException;
import com.andjmel.prvizadatak.service.KnjigaService;
import java.time.LocalDate;
import java.util.Scanner;
import java.util.logging.Level;
import java.util.logging.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.stereotype.Component;

@Component
public class PrviZadatak {

    private KnjigaService knjigaJpaService;
    private KnjigaService knjigaSpringService;

    @Autowired
    public PrviZadatak(@Qualifier("jpa-service") KnjigaService knjigaJpaService,
                       @Qualifier("spring-service") KnjigaService knjigaSpringService) {
        this.knjigaJpaService = knjigaJpaService;
        this.knjigaSpringService = knjigaSpringService;
    }

    public static void main(String[] args) {
        try {
            ApplicationContext container =
                new AnnotationConfigApplicationContext(AppConfig.class);
            PrviZadatak app = container.getBean(PrviZadatak.class);

            KnjigaDto dtoJpa = new KnjigaDto("Romeo i Julija", LocalDate.EPOCH, 50);
            dtoJpa.setautorId(1L);
            dtoJpa.setIzdavacId(1L);

            KnjigaDto dtoSpring = new KnjigaDto("Hamlet", LocalDate.EPOCH, 70);

            try {
                System.out.println("Unesite broj: 1-jpa save  2-jpa delete" +
                                   "  3-spring jdbc save  4-spring jdbc delete:");
                Scanner sc = new Scanner(System.in);
                String input = sc.next().trim();

                KnjigaDto knjigaZaBrisanje = new KnjigaDto();

                if (input.equals("1")) {
                    app.saveKnjigaJpa(dtoJpa);
                } else if (input.equals("2")) {
                    app.deleteKnjigaJpa(knjigaZaBrisanje);
                } else if (input.equals("3")) {
                    app.saveKnjigaSpring(dtoSpring);
                } else if (input.equals("4")) {
                    app.deleteKnjigaSpring(knjigaZaBrisanje);
                }

            } catch (BookExistException ex) {
                Logger.getLogger(PrviZadatak.class.getName())
                      .log(Level.SEVERE, null, ex);
            }
        } catch (BookDoesntExistException ex) {
            Logger.getLogger(PrviZadatak.class.getName())
                  .log(Level.SEVERE, null, ex);
        }
    }

    private void saveKnjigaJpa(KnjigaDto dto) throws BookExistException {
        knjigaJpaService.save(dto);
    }

    private void deleteKnjigaJpa(KnjigaDto dto) throws BookDoesntExistException {
        knjigaJpaService.delete(dto);
    }

    private void saveKnjigaSpring(KnjigaDto dto) throws BookExistException {
        knjigaSpringService.save(dto);
    }

    private void deleteKnjigaSpring(KnjigaDto dto) throws BookDoesntExistException {
        knjigaSpringService.delete(dto);
    }
}
```

> **Objašnjenje:** `@Component` → Spring prepoznaje klasu i može je instancirati kao bean. `AnnotationConfigApplicationContext(AppConfig.class)` → pokretanje IoC kontejnera sa Java konfiguracijom (umjesto XML). `@Qualifier` → jer postoje dva beana tipa `KnjigaService`, Spring ne zna koji da ubrizga bez eksplicitnog kvalifikera. Korisnik bira između 4 opcije → aplikacija delegira na odgovarajući servis/repozitorijum.

---

---

## PROJEKAT 2: DiTime

---

## Korak 1 – `pom.xml`

Jednostavna zavisnost – samo Spring Context za IoC/DI.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>fon.bg.ac.rs</groupId>
    <artifactId>DiTime</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <exec.mainClass>fon.bg.ac.rs.ditime.DiTime</exec.mainClass>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>7.0.7</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <mainClass>${exec.mainClass}</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Korak 2 – `SpringIoc.java` (konfiguracija kontejnera)

Konfiguracijska klasa – definiše bean za `LocalTime` i skenira komponente.

```java
package fon.bg.ac.rs.ditime;

import java.time.LocalTime;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;

@ComponentScan(basePackages = {"fon.bg.ac.rs.ditime"})
public class SpringIoc {

    @Bean("time")
    public LocalTime getTime() {
        return LocalTime.now();
    }
}
```

> **Objašnjenje:** `@ComponentScan` → Spring traži `@Component` klase u paketu. `@Bean("time")` → kreira bean tipa `LocalTime` sa imenom `time`. `LocalTime.now()` se poziva u trenutku pokretanja kontejnera, što znači da bean "zamrzne" trenutno vreme.

---

## Korak 3 – `DiTimeApp.java`

Komponenta koja prima `LocalTime` bean putem konstruktorske injekcije.

```java
package fon.bg.ac.rs.ditime;

import java.time.LocalTime;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class DiTimeApp {

    private final LocalTime time;

    @Autowired
    public DiTimeApp(LocalTime time) {
        this.time = time;
    }

    public LocalTime getCurrentTime() {
        return time;
    }
}
```

> **Objašnjenje:** `@Component` → Spring automatski registruje ovu klasu. `@Autowired` na konstruktoru → Spring injektuje bean tipa `LocalTime` (koji je definisan u `SpringIoc` kao `"time"`). Qualifier nije potreban jer postoji samo jedan bean tog tipa. `getCurrentTime()` vraća injektirano vreme.

---

## Korak 4 – `DiTime.java` (main klasa)

Pokretač aplikacije – inicijalizuje kontejner i ispisuje trenutno vreme.

```java
package fon.bg.ac.rs.ditime;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class DiTime {

    public static void main(String[] args) {
        ApplicationContext ioc =
            new AnnotationConfigApplicationContext(SpringIoc.class);

        DiTimeApp app = ioc.getBean(DiTimeApp.class);

        System.out.println("Trenutno vreme je: " + app.getCurrentTime());
    }
}
```

> **Objašnjenje:** `AnnotationConfigApplicationContext(SpringIoc.class)` → pokretanje IoC kontejnera koristeći `SpringIoc` kao konfiguracionu klasu. `ioc.getBean(DiTimeApp.class)` → dohvatanje bean instance iz kontejnera. Rezultat je ispis trenutnog vremena koje je bilo injektovano kroz DI.

---

## Pregled arhitekture (PrviZadatak)

```
PrviZadatak (main / @Component)
    │
    ├── KnjigaService (interfejs)
    │       └── ServiceImpl
    │               ├── KnjigaRep ──► JpaRep (@Repository "jpa_repo")
    │               │            └── SpringRep (@Repository "spring_repo")
    │               └── KnjigaConverter (@Component "conv")
    │
    ├── AppConfig (@ComponentScan + @Bean definicije)
    │       ├── EntityManagerFactory ("emf")
    │       ├── DataSource
    │       ├── JdbcTemplate ("jdbc")
    │       ├── KnjigaService ("jpa-service")
    │       └── KnjigaService ("spring-service")
    │
    └── domain/
            ├── Knjiga (@Entity)
            ├── Autor (@Entity)
            ├── Izdavac (@Entity)
            └── Korisnik (@Entity)
```
