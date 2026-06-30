# Priprema — Aplikacija za upravljanje zadacima

Jednostavna veb aplikacija rađena u **Jakarta EE** (Servlet 6.0) + **Spring IoC** (bez Spring Boot-a), bez baze podataka — svi podaci se čuvaju u memoriji (`ServletContext` i `HttpSession`).

## Tehnologije

- Jakarta EE 11 (Servlet API, JSP, JSTL)
- Spring Context 6.2.18 (samo IoC/DI kontejner, `AnnotationConfigApplicationContext`)
- Maven (packaging: `war`)
- Java 17

## Arhitektura — kratak pregled

```
korisnik → AppServlet (/app/*) → mapaAkcija (Map<String, AbstractAction>) → konkretna Action klasa
                                                                                   ↓
                                                                         TaskService / UserService
                                                                                   ↓
                                                                          domain objekti (User, Task)
```

Aplikacija koristi **front-controller** šablon: postoji samo jedan servlet (`AppServlet`) koji prima sve zahteve na `/app/*`, a putanju (npr. `/login`, `/add`) mapira na odgovarajuću `Action` klasu preko mape koju Spring sklapa kao bean (`mapaAkcija`). Svaka `Action` klasa nasleđuje `AbstractAction` i implementira samo svoju poslovnu logiku, dok zajedničku logiku (uzimanje sesije, injektovanje servisa) drži apstraktna klasa.

---

## `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>rs.fon</groupId>
    <artifactId>Priprema</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>
    <name>Priprema-1.0-SNAPSHOT</name>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <jakartaee>11.0.0-M1</jakartaee>
    </properties>

    <dependencies>
        <dependency>
            <groupId>jakarta.platform</groupId>
            <artifactId>jakarta.jakartaee-api</artifactId>
            <version>${jakartaee}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>6.2.18</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>jakarta.servlet.jsp.jstl</groupId>
            <artifactId>jakarta.servlet.jsp.jstl-api</artifactId>
            <version>3.0.1</version>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.glassfish.web</groupId>
            <artifactId>jakarta.servlet.jsp.jstl</artifactId>
            <version>3.0.1</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.12.1</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

**Napomena o zavisnostima:**
- `jakarta.jakartaee-api` — `provided` scope jer server (npr. Payara/GlassFish) već nosi te klase u runtime-u, ne pakuju se u WAR.
- `spring-context` — donosi ceo Spring IoC kontejner (uključuje `spring-beans`, `spring-core`, `spring-aop`...) preko tranzitivnih zavisnosti.
- JSTL zavisnosti (`api` + implementacija od `org.glassfish.web`) — potrebne da bi `<c:forEach>`, `<c:if>` itd. radili u JSP-u.

---

## Konfiguracioni fajlovi

### `web.xml`

```xml
<web-app version="6.0" xmlns="https://jakarta.ee/xml/ns/jakartaee" ...>
    <listener>
        <listener-class>rs.fon.priprema.listener.MyApplicationContextListener</listener-class>
    </listener>
    <session-config>
        <session-timeout>30</session-timeout>
    </session-config>
</web-app>
```

Registruje samo `ServletContextListener` (servlet `AppServlet` se ne mora ovde prijavljivati jer koristi `@WebServlet` anotaciju) i postavlja trajanje sesije na 30 minuta.

### `context.xml`

```xml
<Context path="/Priprema"/>
```

Određuje context path aplikacije na serveru (`/Priprema`).

### `beans.xml`

Prazan CDI deskriptor (`bean-discovery-mode="all"`) — prisutan jer ga Jakarta EE profil po konvenciji očekuje, ali se u ovom projektu DI ne radi kroz CDI nego isključivo kroz **Spring**.

---

## Java klase (servisni i kontrolni sloj)

> Geteri i seteri su izostavljeni iz dela sa domenskim klasama radi preglednosti — fokus je na onome što čini logiku aplikacije.

### `config/AppConfig.java`

```java
@Configuration
@ComponentScan(basePackages = {"rs.fon.priprema"})
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserServiceImpl();
    }

    @Bean
    public TaskService taskService() {
        return new TaskServiceImpl();
    }

    @Bean(name = "mapaAkcija")
    public Map<String, AbstractAction> mapaAkcija(LoginAction loginAction, LogoutAction logoutAction,
            AddAction addAction, ShowAddAction showAddAction,
            ShowEditAction showEditAction, EditAction editAction,
            ChangeStatusAction changestatus, DeleteAction del) {
        Map<String, AbstractAction> mapa = new HashMap<>();
        mapa.put("/login", loginAction);
        mapa.put("/logout", logoutAction);
        mapa.put("/add", addAction);
        mapa.put("/showadd", showAddAction);
        mapa.put("/showedit", showEditAction);
        mapa.put("/edit", editAction);
        mapa.put("/changestatus", changestatus);
        mapa.put("/delete", del);
        return mapa;
    }
}
```

Centralna Spring konfiguracija. `@ComponentScan` nalazi sve klase označene sa `@Component` (sve `Action` klase) i pravi ih kao bean-ove. Metoda `mapaAkcija()` je sama Spring bean fabrika — Spring po imenu/tipu parametra prepoznaje da treba da ubrizga već postojeće `Action` bean-ove (npr. `loginAction`), pa ih ne treba ručno praviti sa `new`. Rezultat je mapa `putanja → Action` koja servletu služi kao "ruter".

### `listener/MyApplicationContextListener.java`

```java
public class MyApplicationContextListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        ApplicationContext ioc = new AnnotationConfigApplicationContext(AppConfig.class);
        sce.getServletContext().setAttribute("ioc", ioc);

        List<User> korisnici = new ArrayList<>();
        List<Task> zadaci = new ArrayList<>();
        zadaci.add(new Task("Spring", "Spring aplikacija", Status.AKTIVAN));
        korisnici.add(new User("ana", zadaci, false));
        korisnici.add(new User("nada", new ArrayList<Task>(), false));
        sce.getServletContext().setAttribute("korisnici", korisnici);
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {}
}
```

Pokreće se jednom, pri podizanju aplikacije. Pravi Spring IoC kontejner (`AnnotationConfigApplicationContext`) iz `AppConfig` klase i čuva ga u `ServletContext`-u pod ključem `"ioc"` — tako svaki servlet kasnije može da mu pristupi i izvuče bean-ove. Takođe ovde se inicijalizuju test podaci (lista korisnika sa po jednim primerom zadatka) i smeštaju u `ServletContext` pod ključem `"korisnici"`, čime postaju vidljivi celoj aplikaciji.

### `servlet/AppServlet.java`

```java
@WebServlet(name = "AppServlet", urlPatterns = {"/app/*"})
public class AppServlet extends HttpServlet {

    @Autowired
    @Qualifier(value = "mapaAkcija")
    Map<String, AbstractAction> mapaAkcija;

    @Override
    public void init(ServletConfig config) throws ServletException {
        ApplicationContext ioc = (ApplicationContext) config.getServletContext().getAttribute("ioc");
        ioc.getAutowireCapableBeanFactory().autowireBean(this);
    }

    protected void processRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html;charset=UTF-8");
        String putanja = request.getPathInfo();
        AbstractAction action = mapaAkcija.get(putanja);
        String page = action.execute(request, response);
        request.getRequestDispatcher(page).forward(request, response);
    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }
}
```

Jedini servlet u aplikaciji (front controller), mapiran na `/app/*`. Pošto servlet *nije* Spring bean (kreira ga sam server preko `@WebServlet`), Spring ne može automatski da mu ubrizga zavisnosti — zato se u `init()` ručno uzima IoC kontejner iz `ServletContext`-a i poziva `autowireBean(this)`, čime se polje `mapaAkcija` ipak popunjava. `doGet` i `doPost` rade identično — oba pozivaju `processRequest`, koji iz putanje (`getPathInfo()`, npr. `/login`) nađe odgovarajuću `Action`, izvrši je i prosledi (forward) na JSP stranicu koju ta akcija vrati.

### `action/AbstractAction.java`

```java
public abstract class AbstractAction {
    @Autowired
    protected TaskService taskService;
    @Autowired
    protected UserService userService;

    public String execute(HttpServletRequest request, HttpServletResponse response){
        return doExecute(request, response, request.getSession());
    }

    protected abstract String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session);
}
```

Bazna klasa za sve akcije — ima zajedničke servise (`taskService`, `userService`) koje Spring automatski ubrizgava u svaku konkretnu akciju. Javna metoda `execute()` se uvek poziva iz servleta i garantuje da svaka akcija dobije sesiju na isti način, dok `doExecute()` je apstraktna metoda — svaka konkretna akcija je implementira sa svojom logikom (template method šablon).

### `action/LoginAction.java`

```java
@Component
public class LoginAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        if (session.getAttribute("ulogovan") != null) {
            return "/WEB-INF/main.jsp";
        }
        String u = request.getParameter("username");
        if (u == null || u.trim().isEmpty()) {
            return "/login.jsp";
        }
        User user = userService.pronadjiKorisnika(
            (List<User>) request.getServletContext().getAttribute("korisnici"), u
        );
        if (user == null) {
            request.setAttribute("greska", "Korisnik ne postoji");
            return "/login.jsp";
        }
        user.setOnline(true);
        session.setAttribute("ulogovan", user);
        return "/WEB-INF/main.jsp";
    }
}
```

Pokriva ceo tok prijave: (1) ako je korisnik već ulogovan, odmah ga šalje na glavnu stranicu (sprečava ponovnu prijavu), (2) ako forma još nije popunjena (GET na `/app/login`), prikazuje formu za prijavu, (3) traži korisnika u listi iz `ServletContext`-a preko `UserService`, i ako ne postoji vraća grešku, (4) ako postoji — obeležava ga kao online i čuva u sesiji pod ključem `"ulogovan"`.

### `action/LogoutAction.java`

```java
@Component
public class LogoutAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        User u = (User) session.getAttribute("ulogovan");
        u.setOnline(false);
        session.invalidate();
        return "/login.jsp";
    }
}
```

Pre nego što uništi sesiju, postavlja korisnika kao offline (jer se evidencija online korisnika čuva u `ServletContext`-u, na koji `Task`/`User` objekat i dalje referenciše i posle invalidacije sesije). `session.invalidate()` briše sve podatke iz sesije, pa korisnik mora ponovo da se prijavi.

### `action/AddAction.java`

```java
@Component
public class AddAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        User u = (User) session.getAttribute("ulogovan");
        taskService.dodajZadatak(u, request.getParameter("naziv"), request.getParameter("opis"),
                Status.valueOf(request.getParameter("status")));
        return "/WEB-INF/main.jsp";
    }
}
```

Čita podatke iz forme (`naziv`, `opis`, `status`) i preko `TaskService` dodaje novi zadatak trenutno ulogovanom korisniku. `Status.valueOf(...)` pretvara string iz `<select>` elementa u `enum` vrednost.

### `action/ShowAddAction.java`

```java
@Component
public class ShowAddAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        return "/WEB-INF/add.jsp";
    }
}
```

Nema poslovnu logiku — samo prikazuje praznu formu za dodavanje zadatka. Postoji kao posebna akcija od `AddAction` zato što GET (prikaz forme) i POST (stvarno dodavanje) imaju potpuno različitu svrhu, pa ih je čistije razdvojiti u dve akcije nego granati po HTTP metodu unutar jedne.

### `action/ShowEditAction.java`

```java
@Component
public class ShowEditAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        User u = (User) session.getAttribute("ulogovan");
        String naziv = request.getParameter("naziv");
        for (Task t : u.getTasks()) {
            if (t.getName().equals(naziv)) {
                request.setAttribute("task", t);
                break;
            }
        }
        return "/WEB-INF/edit.jsp";
    }
}
```

Pronalazi zadatak po nazivu u listi zadataka ulogovanog korisnika i stavlja ga u `request` atribut `"task"`, kako bi `edit.jsp` mogao da prikaže postojeće vrednosti (`${task.name}`, `${task.description}`) na formi za izmenu.

### `action/EditAction.java`

```java
@Component
public class EditAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        taskService.izmeniZadatak((User) session.getAttribute("ulogovan"),
                request.getParameter("naziv"),
                request.getParameter("noviNaziv"),
                request.getParameter("opis"));
        return "/WEB-INF/main.jsp";
    }
}
```

Prima stari naziv (po kome se zadatak pronalazi), novi naziv i novi opis sa forme i prosleđuje ih `TaskService`-u da izvrši izmenu.

### `action/ChangeStatusAction.java`

```java
@Component
public class ChangeStatusAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        taskService.promeniStatus((User) session.getAttribute("ulogovan"), request.getParameter("naziv"));
        return "/WEB-INF/main.jsp";
    }
}
```

Menja status zadatka (aktivan ↔ završen) za zadatak sa datim nazivom kod trenutno ulogovanog korisnika.

### `action/DeleteAction.java`

```java
@Component
public class DeleteAction extends AbstractAction {
    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        taskService.obrisiZadatak((User) session.getAttribute("ulogovan"), request.getParameter("naziv"));
        return "/WEB-INF/main.jsp";
    }
}
```

Briše zadatak sa datim nazivom iz liste zadataka ulogovanog korisnika.

---

## Servisni sloj

### `service/UserService.java` i `service/impl/UserServiceImpl.java`

```java
public interface UserService {
    User pronadjiKorisnika(List<User> korisnici, String username);
}
```

```java
public class UserServiceImpl implements UserService {
    @Override
    public User pronadjiKorisnika(List<User> korisnici, String username) {
        for (User user : korisnici) {
            if (user.getUsername().equals(username)) return user;
        }
        return null;
    }
}
```

Jedna jedina odgovornost — pronalaženje korisnika po korisničkom imenu u listi (linearna pretraga), koristi se isključivo u `LoginAction`.

### `service/TaskService.java` i `service/impl/TaskServiceImpl.java`

```java
public interface TaskService {
    void dodajZadatak(User user, String naziv, String opis, Status status);
    void izmeniZadatak(User user, String stariNaziv, String noviNaziv, String noviOpis);
    void obrisiZadatak(User user, String naziv);
    void promeniStatus(User user, String naziv);
}
```

```java
public class TaskServiceImpl implements TaskService {
    @Override
    public void dodajZadatak(User user, String naziv, String opis, Status status) {
        user.getTasks().add(new Task(naziv, opis, status));
    }

    @Override
    public void izmeniZadatak(User user, String stariNaziv, String noviNaziv, String noviOpis) {
        for (Task task : user.getTasks()) {
            if (task.getName().equals(stariNaziv)) {
                task.setName(noviNaziv);
                task.setDescription(noviOpis);
                return;
            }
        }
    }

    @Override
    public void obrisiZadatak(User user, String naziv) {
        Task t = null;
        for (Task task : user.getTasks()) {
            if (task.getName().equals(naziv)) {
                t = task;
                break;
            }
        }
        if (t != null) user.getTasks().remove(t);
    }

    @Override
    public void promeniStatus(User user, String naziv) {
        for (Task task : user.getTasks()) {
            if (task.getName().equals(naziv)) {
                task.setStatus(task.getStatus() == Status.AKTIVAN ? Status.ZAVRSEN : Status.AKTIVAN);
                return;
            }
        }
    }
}
```

Sva poslovna logika oko zadataka — dodavanje, izmena, brisanje i promena statusa. Sve operacije rade isključivo nad listom `user.getTasks()` koja se nalazi u memoriji (nema baze, nema upita), tako da je suština servisa samo iteriranje i poređenje po nazivu zadatka.

---

## Domenske klase

> Bez getera/setera — prikazana su samo polja i konstruktori, jer je to ono što definiše model.

### `domain/User.java`

```java
public class User implements Serializable {
    private String username;
    private List<Task> tasks;   // svaki User ima svoju zasebnu listu zadataka
    private boolean online;

    public User(String username, List<Task> tasks, boolean online) {
        this.username = username;
        this.tasks = tasks;
        this.online = online;
    }
}
```

`Serializable` jer se objekat čuva u `HttpSession`-u (server po potrebi sesije serijalizuje). Polje `online` je ono na osnovu kog se na `users.jsp` prikazuje da li je korisnik trenutno prijavljen.

### `domain/Task.java`

```java
public class Task {
    private String name;
    private String description;
    private LocalDateTime dateOfCreation;
    private Status status;

    public Task(String name, String description, Status status) {
        this.name = name;
        this.description = description;
        this.status = status;
        this.dateOfCreation = LocalDateTime.now(); // automatski se postavlja
    }
}
```

Datum kreiranja se ne prosleđuje kroz konstruktor nego se automatski postavlja na `LocalDateTime.now()` u trenutku kreiranja objekta — tako se ispunjava zahtev da se datum "automatski postavlja".

### `domain/Status.java`

```java
public enum Status implements Serializable {
    AKTIVAN, ZAVRSEN;
}
```

Enum sa dve moguće vrednosti statusa zadatka.

---

## JSP stranice (`webapp`)

### `index.html`

Statična početna stranica (samo `Hello World!`) — nije deo realnog toka aplikacije, ostala je kao default NetBeans template.

### `login.jsp`

```jsp
<%@taglib prefix="c" uri="jakarta.tags.core"%>
<%@page contentType="text/html" pageEncoding="UTF-8"%>
...
<form action="${pageContext.request.contextPath}/app/login" method="POST">
    Korisničko ime: <input type="text" name="username" />
    <input type="submit" value="Prijavi se" />
</form>
<c:if test="${not empty greska}">
    <p style="color: red">${greska}</p>
</c:if>
```

Forma za prijavu — šalje POST na `/app/login`, što hvata `AppServlet` i prosleđuje `LoginAction`-u. `<c:if>` (JSTL core tag) ispisuje poruku o grešci samo ako request atribut `greska` postoji i nije prazan (tu poruku postavlja `LoginAction` kad korisnik ne postoji). `${pageContext.request.contextPath}` se koristi umesto hardkodovane putanje da bi linkovi/forme radili bez obzira na context path aplikacije.

### `WEB-INF/main.jsp`

```jsp
<%@taglib prefix="c" uri="jakarta.tags.core" %>
...
<h3><a href="${pageContext.request.contextPath}/app/logout">Odjava</a></h3>
<table border="1">
    <tbody>
        <c:forEach var="t" items="${ulogovan.tasks}">
        <tr>
            <td>${t.name}</td>
            <td>${t.description}</td>
            <td>${t.dateOfCreation}</td>
            <td>${t.status}</td>
            <td><a href="${pageContext.request.contextPath}/app/delete?naziv=${t.name}">Obrisi</a></td>
            <td><a href="${pageContext.request.contextPath}/app/changestatus?naziv=${t.name}">Izmeni status</a></td>
            <td><a href="${pageContext.request.contextPath}/app/showedit?naziv=${t.name}">Edit</a></td>
        </tr>
        </c:forEach>
    </tbody>
</table>
<h3><a href="${pageContext.request.contextPath}/app/showadd">Dodaj novi zadatak</a></h3>
<jsp:include page="/WEB-INF/fragment/users.jsp" flush="true"></jsp:include>
```

Glavna stranica posle prijave. `<c:forEach>` iterira kroz `${ulogovan.tasks}` — `ulogovan` se automatski nalazi jer JSP/EL prvo traži atribut tim imenom u `page` → `request` → `session` → `application` scope-u (ovde je u sesiji, postavljen u `LoginAction`). Svaka akcija (`Obrisi`, `Izmeni status`, `Edit`) je obični link sa query parametrom `naziv`, koji ide na odgovarajuću putanju iz `mapaAkcija`. Na dnu, `<jsp:include>` (standardna JSP akcija, ne JSTL tag) ubacuje sadržaj `users.jsp` fragmenta direktno u ovu stranicu u trenutku izvršavanja — `flush="true"` znači da se bafer prethodne stranice ispražnjava pre uključivanja fragmenta.

### `WEB-INF/add.jsp`

```jsp
<form method="POST" action="${pageContext.request.contextPath}/app/add">
    <label for="naziv">Naziv:</label>
    <input type="text" name="naziv" id="naziv" value="" />
    <label for="opis">Opis:</label>
    <input type="text" name="opis" id="opis" value="" />
    <label for="status">Aktivan?</label>
    <select name="status" id="status">
        <option value="AKTIVAN">Aktivan</option>
        <option value="ZAVRSEN">Završen</option>
    </select>
    <button type="submit"/>Dodaj</button>
</form>
<jsp:include page="/WEB-INF/fragment/users.jsp" flush="true"></jsp:include>
```

Forma za dodavanje zadatka, šalje POST na `/app/add` (hvata ga `AddAction`). `<select>` ograničava unos statusa samo na dve dozvoljene `enum` vrednosti, čije se `value` atribute direktno mapiraju preko `Status.valueOf(...)` u `AddAction`-u. `for`/`id` parovi povezuju `<label>` sa odgovarajućim poljem radi pristupačnosti.

### `WEB-INF/edit.jsp`

```jsp
<form method="POST" action="${pageContext.request.contextPath}/app/edit">
    <label for="naziv">Stari naziv:</label>
    <input type="text" name="naziv" id="naziv" value="${task.name}" readonly="readonly"/>
    <label for="noviNaziv">Naziv:</label>
    <input type="text" name="noviNaziv" id="noviNaziv" value="" />
    <label for="opis">Opis:</label>
    <input type="text" name="opis" id="opis" value="" />
    <button type="submit">Izmeni</button>
</form>
```

Forma za izmenu postojećeg zadatka. Polje `naziv` je popunjeno vrednošću iz `${task.name}` (atribut koji je postavio `ShowEditAction`) i obeleženo kao `readonly` — služi samo kao identifikator po kom `EditAction` pronalazi koji zadatak treba izmeniti, dok se stvarna izmena unosi u `noviNaziv` i `opis`.

### `WEB-INF/fragment/users.jsp`

```jsp
<%@taglib prefix="c" uri="jakarta.tags.core" %>
<h1>Korisnici sistema</h1>
<ul>
    <c:forEach var="u" items="${applicationScope.korisnici}">
        <li>${u.username} -
            <c:if test="${u.online}">online</c:if>
            <c:if test="${not u.online}">offline</c:if>
        </li>
    </c:forEach>
</ul>
```

JSP fragment koji se uključuje (`include`) i u `main.jsp` i u `add.jsp` — prikazuje listu **svih** korisnika sistema sa statusom online/offline. `${applicationScope.korisnici}` eksplicitno čita iz `ServletContext`-a (application scope), gde je listu postavio `MyApplicationContextListener` pri pokretanju aplikacije — zato je ova lista vidljiva na svakoj stranici, nezavisno od toga ko je trenutno ulogovan.

---

## Tok aplikacije (sažetak)

1. Korisnik otvara `login.jsp` i unosi korisničko ime → POST `/app/login`.
2. `AppServlet` nalazi `LoginAction` u `mapaAkcija` i izvršava je.
3. Ako korisnik postoji, smešta se u sesiju (`ulogovan`) i korisnik se šalje na `main.jsp`.
4. Sa `main.jsp` korisnik upravlja svojim zadacima (dodaje, menja status, briše, edituje) — svaka akcija ide kroz `/app/*` i odgovarajuću `Action` klasu, pa se vraća nazad na `main.jsp`.
5. `users.jsp` fragment na dnu uvek prikazuje sve korisnike sistema i njihov online status.
6. Klikom na "Odjava" — `LogoutAction` postavlja korisnika kao offline i uništava sesiju.
