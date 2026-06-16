# Пројекти: DiDate + NjtIoC1 (Application)

---

## 1. DiDate пројекат

### `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>rs.fon</groupId>
    <artifactId>DiDate</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.release>21</maven.compiler.release>
        <exec.mainClass>rs.fon.didate.DiDate</exec.mainClass>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>7.0.7</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>
</project>
```

---

### `SpringIoc.java` — конфигурациона класа (Spring IoC контејнер)

```java
package rs.fon.didate;

import java.time.LocalDate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages = {"rs.fon.didate"})
public class SpringIoc {

    @Bean("date")
    public LocalDate getDate() {
        return LocalDate.now();
    }
}
```

---

### `DiDateApp.java` — бин класа

```java
package rs.fon.didate;

import java.time.LocalDate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;

@Component
public class DiDateApp {

    private final LocalDate date;

    @Autowired
    public DiDateApp(@Qualifier("date") LocalDate date) {
        this.date = date;
    }

    public LocalDate getDate() {
        return date;
    }
}
```

---

### `DiDate.java` — главни програм

```java
package rs.fon.didate;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class DiDate {

    public static void main(String[] args) {
        ApplicationContext ioc = new AnnotationConfigApplicationContext(SpringIoc.class);
        DiDateApp app = ioc.getBean(DiDateApp.class);
        System.out.println("Danasnji datum je " + app.getDate());
    }
}
```

---

---

## 2. NjtIoC1 пројекат (Application)

### `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>rs.fon</groupId>
    <artifactId>Application</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>NjtIoc2</name>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.release>21</maven.compiler.release>
        <exec.mainClass>rs.fon.application.NjtIoc2</exec.mainClass>
    </properties>
    <dependencies>
        <!-- JPA API -->
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
        <!-- Spring JDBC -->
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

### `persistence.xml` — `src/main/resources/META-INF/persistence.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="3.1"
             xmlns="https://jakarta.ee/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="https://jakarta.ee/xml/ns/persistence
                                 https://jakarta.ee/xml/ns/persistence/persistence_3_0.xsd">
    <persistence-unit name="ProjectPU" transaction-type="RESOURCE_LOCAL">
        <provider>org.eclipse.persistence.jpa.PersistenceProvider</provider>
        <class>rs.fon.application.domain.Sluzbenik</class>
        <class>rs.fon.application.domain.Project</class>
        <class>rs.fon.application.domain.Investitor</class>
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

---

### Домен — `Investitor.java`

```java
package rs.fon.application.domain;

import jakarta.persistence.*;

@Entity
public class Investitor {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idinvestitor;

    @Column private String naziv;
    @Column private String pib;
    @Column private String maticnibroj;
    @Column private String srediste;

    public Investitor() {}

    public Investitor(Long idinvestitor) {
        this.idinvestitor = idinvestitor;
    }

    // getteri i setteri...
}
```

---

### Домен — `Project.java`

```java
package rs.fon.application.domain;

import jakarta.persistence.*;
import java.time.LocalDate;

@Entity
@Table(name = "project")
public class Project {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idprojekat;

    @Column private String naziv;
    @Column private LocalDate pocetak;
    @Column private LocalDate kraj;
    @Column private String grad;

    @ManyToOne
    @JoinColumn
    private Investitor investitor;

    public Project() {}

    // getteri i setteri...
}
```

---

### Домен — `Sluzbenik.java`

```java
package rs.fon.application.domain;

import jakarta.persistence.*;
import java.time.LocalDate;

@Entity
@Table(name = "sluzbenik")
public class Sluzbenik {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long idsluzbenik;

    @Column private String email;
    @Column private String sifra;
    @Column private String ime;
    @Column private String prezime;
    @Column private LocalDate datumrodjenja;

    public Sluzbenik() {}

    public Sluzbenik(Long idsluzbenik) {
        this.idsluzbenik = idsluzbenik;
    }

    // getteri i setteri...
}
```

---

### DTO — `ProjectDto.java`

```java
package rs.fon.application.dto;

import java.time.LocalDate;

public class ProjectDto {

    private Long idprojekat;
    private String naziv;
    private LocalDate pocetak;
    private LocalDate kraj;
    private String grad;
    private Long idinvestitor;

    public ProjectDto() {}

    // getteri i setteri...
}
```

---

### DTO — `Converter.java` (интерфејс)

```java
package rs.fon.application.dto;

public interface Converter<Dto, Entity> {
    Entity toEntity(Dto dto);
    Dto toDto(Entity entity);
}
```

---

### DTO — `ProjectConverter.java`

```java
package rs.fon.application.dto;

import org.springframework.stereotype.Component;
import rs.fon.application.domain.Investitor;
import rs.fon.application.domain.Project;

@Component
public class ProjectConverter implements Converter<ProjectDto, Project> {

    @Override
    public Project toEntity(ProjectDto dto) {
        Project p = new Project();
        p.setGrad(dto.getGrad());
        p.setIdprojekat(dto.getIdprojekat());
        p.setKraj(dto.getKraj());
        p.setPocetak(dto.getPocetak());
        p.setNaziv(dto.getNaziv());
        if (dto.getIdinvestitor() != null) {
            p.setInvestitor(new Investitor(dto.getIdinvestitor()));
        }
        return p;
    }

    @Override
    public ProjectDto toDto(Project entity) {
        ProjectDto dto = new ProjectDto();
        dto.setGrad(entity.getGrad());
        dto.setIdprojekat(entity.getIdprojekat());
        dto.setKraj(entity.getKraj());
        dto.setNaziv(entity.getNaziv());
        dto.setPocetak(entity.getPocetak());
        if (entity.getInvestitor() != null) {
            dto.setIdinvestitor(entity.getInvestitor().getIdinvestitor());
        }
        return dto;
    }
}
```

---

### Изузеци — `ProjectExistException.java`

```java
package rs.fon.application.ex;

public class ProjectExistException extends Exception {

    public ProjectExistException(String message) {
        super(message);
    }
}
```

---

### Изузеци — `ProjectDoesNotExistException.java`

```java
package rs.fon.application.ex;

public class ProjectDoesNotExistException extends Exception {

    public ProjectDoesNotExistException(String message) {
        super(message);
    }
}
```

---

### Репозиторијум — `ProjectRepo.java` (интерфејс)

```java
package rs.fon.application.repo;

import rs.fon.application.domain.Project;

public interface ProjectRepo {
    Project save(Project p);
    void delete(Project p);
}
```

---

### Репозиторијум — `JpaRepo.java` (JPA имплементација)

```java
package rs.fon.application.repo.impl;

import jakarta.persistence.EntityManager;
import jakarta.persistence.EntityManagerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Repository;
import rs.fon.application.domain.Project;
import rs.fon.application.repo.ProjectRepo;

@Repository("jpa_repo")
public class JpaRepo implements ProjectRepo {

    private final EntityManagerFactory emf;

    @Autowired
    public JpaRepo(@Qualifier("emf") EntityManagerFactory emf) {
        this.emf = emf;
    }

    @Override
    public Project save(Project p) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        em.merge(p);
        em.getTransaction().commit();
        em.close();
        return p;
    }

    @Override
    public void delete(Project p) {
        EntityManager em = emf.createEntityManager();
        em.getTransaction().begin();
        Project toRemove = em.find(Project.class, p.getIdprojekat());
        if (toRemove != null) em.remove(toRemove);
        em.getTransaction().commit();
        em.close();
    }
}
```

---

### Репозиторијум — `SpringRepo.java` (Spring JDBC имплементација)

```java
package rs.fon.application.repo.impl;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;
import rs.fon.application.domain.Project;
import rs.fon.application.repo.ProjectRepo;

@Repository("spring_repo")
public class SpringRepo implements ProjectRepo {

    private final JdbcTemplate jdbc;

    @Autowired
    public SpringRepo(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    @Override
    public Project save(Project p) {
        jdbc.update(
            "INSERT INTO project (naziv, pocetak, kraj, investitor_idinvestitor, grad) VALUES (?, ?, ?, ?, ?)",
            p.getNaziv(),
            p.getPocetak(),
            p.getKraj(),
            p.getInvestitor() != null ? p.getInvestitor().getIdinvestitor() : null,
            p.getGrad()
        );
        return p;
    }

    @Override
    public void delete(Project p) {
        jdbc.update("DELETE FROM project WHERE idprojekat = ?", p.getIdprojekat());
    }
}
```

---

### Сервис — `ProjectService.java` (интерфејс)

```java
package rs.fon.application.serv;

import rs.fon.application.dto.ProjectDto;
import rs.fon.application.ex.ProjectDoesNotExistException;
import rs.fon.application.ex.ProjectExistException;

public interface ProjectService {
    ProjectDto save(ProjectDto b) throws ProjectExistException;
    void delete(ProjectDto b) throws ProjectDoesNotExistException;
}
```

---

### Сервис — `ServiceImpl.java`

> ⚠️ **Напомена:** Оригинална логика провере у `save()` је исправљена — уместо провере по `idprojekat`, треба проверити по **називу** у бази (задатак то захтева). Доле је исправљена верзија.

```java
package rs.fon.application.serv.impl;

import rs.fon.application.domain.Project;
import rs.fon.application.dto.ProjectConverter;
import rs.fon.application.dto.ProjectDto;
import rs.fon.application.ex.ProjectDoesNotExistException;
import rs.fon.application.ex.ProjectExistException;
import rs.fon.application.repo.ProjectRepo;
import rs.fon.application.serv.ProjectService;

public class ServiceImpl implements ProjectService {

    private final ProjectRepo repo;
    private final ProjectConverter conv;

    public ServiceImpl(ProjectRepo repo, ProjectConverter conv) {
        this.repo = repo;
        this.conv = conv;
    }

    @Override
    public ProjectDto save(ProjectDto b) throws ProjectExistException {
       Project p= conv.toEntity(b);
        if(p.getNaziv() == null || b.getIdprojekat()!=null ) throw new ProjectExistException("Projekat postoji");
        repo.save(p);
        return b;
    }

    @Override
    public void delete(ProjectDto b) throws ProjectDoesNotExistException {
       Project p= conv.toEntity(b);
        if(p==null || p.getIdprojekat()==null) throw new ProjectDoesNotExistException("Projekat ne postoji");
        repo.delete(p);
    }
}
```

---

### Конфигурација — `AppConfig.java`

```java
package rs.fon.application.config;

import jakarta.persistence.EntityManagerFactory;
import jakarta.persistence.Persistence;
import javax.sql.DataSource;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;
import rs.fon.application.dto.ProjectConverter;
import rs.fon.application.repo.ProjectRepo;
import rs.fon.application.serv.ProjectService;
import rs.fon.application.serv.impl.ServiceImpl;

@Configuration
@ComponentScan(basePackages = {"rs.fon.application"})
public class AppConfig {

    @Bean("emf")
    public EntityManagerFactory getEmf() {
        return Persistence.createEntityManagerFactory("ProjectPU");
    }

    @Bean
    public DataSource datasource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setUrl("jdbc:mysql://localhost:3306/njt");
        ds.setUsername("root");
        ds.setPassword("");
        return ds;
    }

    @Bean("jdbc")
    public JdbcTemplate jdbc(DataSource ds) {
        return new JdbcTemplate(ds);
    }

    @Bean("jpa_service")
    public ProjectService jpaService(@Qualifier("jpa_repo") ProjectRepo jpa, ProjectConverter conv) {
        return new ServiceImpl(jpa, conv);
    }

    @Bean("spring_service")
    public ProjectService springService(@Qualifier("spring_repo") ProjectRepo spring, ProjectConverter conv) {
        return new ServiceImpl(spring, conv);
    }
}
```

---

### `Application.java` — бин класа

```java
package rs.fon.application;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Component;
import rs.fon.application.dto.ProjectDto;
import rs.fon.application.ex.ProjectDoesNotExistException;
import rs.fon.application.ex.ProjectExistException;
import rs.fon.application.serv.ProjectService;

@Component
public class Application {

    private final ProjectService jpa;
    private final ProjectService spring;

    @Autowired
    public Application(@Qualifier("jpa_service") ProjectService jpa,
                       @Qualifier("spring_service") ProjectService spring) {
        this.jpa = jpa;
        this.spring = spring;
    }

    public void saveJpa(ProjectDto e) throws ProjectExistException {
        jpa.save(e);
    }

    public void deleteJpa(ProjectDto e) throws ProjectDoesNotExistException {
        jpa.delete(e);
    }

    public void saveSpringJdbc(ProjectDto e) throws ProjectExistException {
        spring.save(e);
    }

    public void deleteSpringJdbc(ProjectDto e) throws ProjectDoesNotExistException {
        spring.delete(e);
    }
}
```

---

### `NjtIoc2.java` — главни програм

```java
package rs.fon.application;

import java.time.LocalDate;
import java.time.Month;
import java.util.Scanner;
import java.util.logging.Level;
import java.util.logging.Logger;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import rs.fon.application.config.AppConfig;
import rs.fon.application.dto.ProjectDto;
import rs.fon.application.ex.ProjectDoesNotExistException;
import rs.fon.application.ex.ProjectExistException;

public class NjtIoc2 {

    public static void main(String[] args) {
        ApplicationContext ioc = new AnnotationConfigApplicationContext(AppConfig.class);
        Application app = ioc.getBean(Application.class);

        // Projekat za save
        ProjectDto p = new ProjectDto();
        p.setGrad("Beograd");
        p.setNaziv("Luna Park");
        p.setIdinvestitor(1L);
        p.setKraj(LocalDate.now());
        p.setPocetak(LocalDate.of(2010, Month.MAY, 15));

        // Projekat za delete – mora imati ID
        ProjectDto remove = new ProjectDto();
        remove.setIdprojekat(1L);

        System.out.println("Unesite broj: 1-jpa save  2-jpa delete  3-spring jdbc save  4-spring jdbc delete:");
        Scanner sc = new Scanner(System.in);
        String input = sc.next().trim();

        try {
            if (input.equals("1")) {
                app.saveJpa(p);
            } else if (input.equals("2")) {
                app.deleteJpa(remove);
            } else if (input.equals("3")) {
                app.saveSpringJdbc(p);
            } else if (input.equals("4")) {
                app.deleteSpringJdbc(remove);
            }
        } catch (ProjectDoesNotExistException ex) {
            Logger.getLogger(NjtIoc2.class.getName()).log(Level.SEVERE, ex.getMessage());
        } catch (ProjectExistException ex) {
            Logger.getLogger(NjtIoc2.class.getName()).log(Level.SEVERE, ex.getMessage());
        }
    }
}
```
