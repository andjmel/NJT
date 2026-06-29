# DrugiKlk – Web aplikacija za praćenje troškova
> Jakarta Servlets + JSP/JSTL + Spring IoC (bez Spring Boot)  
> Podaci čuvani u memoriji (`Map`, `List`), bez baze podataka.

---

## Struktura projekta

```
rs.fon.drugiklk
├── domain/          → entiteti (User, Trosak, Status)
├── service/         → interfejsi servisa
│   └── impl/        → implementacije servisa
├── action/          → Action klase (jedna po URL putanji)
├── config/          → Spring konfiguracija (AppConfig)
├── listener/        → AppContextListener (startup)
├── servlet/         → AppServlet (front controller)
└── webapp/          → JSP stranice + web.xml
```

---

## pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <groupId>rs.fon</groupId>
    <artifactId>DrugiKlk</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <jakartaee>11.0.0-M1</jakartaee>
    </properties>

    <dependencies>
        <!-- Jakarta EE API (Servlet, JSP...) – provided jer server već ima -->
        <dependency>
            <groupId>jakarta.platform</groupId>
            <artifactId>jakarta.jakartaee-api</artifactId>
            <version>${jakartaee}</version>
            <scope>provided</scope>
        </dependency>

        <!-- Spring IoC kontejner (compile = pakuje se u WAR) -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>6.2.18</version>
            <scope>compile</scope>
        </dependency>

        <!-- JSTL API i implementacija (za c:forEach, c:if tagove) -->
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
            <!-- Kompajliranje na Java 17 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.12.1</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>
            <!-- Pravljenje WAR arhive -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Domain sloj

### `Status.java` – enum

```java
public enum Status {
    PLANIRAN, REALIZOVAN
}
```
> Dva moguća statusa troška. Koristi se u `Trosak` klasi i servisnoj logici.

---

### `Trosak.java`

```java
public class Trosak implements Serializable {
    String naziv;
    double iznos;
    Status status;

    public Trosak(String naziv) { this.naziv = naziv; }

    public Trosak(String naziv, double iznos) {
        this.naziv = naziv;
        this.iznos = iznos;
        this.status = Status.PLANIRAN; // novi trosak uvek PLANIRAN
    }

    public Trosak(String naziv, double iznos, Status status) {
        this.naziv = naziv;
        this.iznos = iznos;
        this.status = status;
    }
}
```
> Model troška. Implementira `Serializable` jer se čuva u HTTP sesiji. Novi trosak po defaultu dobija status `PLANIRAN`.

---

### `User.java`

```java
public class User implements Serializable {
    String username;
    String password;
    private boolean online;

    public User(String username, String password, boolean online) {
        this.username = username;
        this.password = password;
        this.online = online;
    }
}
```
> Model korisnika. Polje `online` prati da li je korisnik trenutno prijavljen na sistem — koristi se u `online.jsp` fragmentu i sprečava dvostruku prijavu.

---

## Servisni sloj

### `TrosakService.java` – interfejs

```java
public interface TrosakService {
    List<Trosak> dodajNovi(List<Trosak> expenses, String name, double value);
    List<Trosak> izmeniStatus(List<Trosak> expenses, String name);
    List<Trosak> izbrisi(List<Trosak> expenses, String name);
}
```

### `UserService.java` – interfejs

```java
public interface UserService {
    User pronadjiKorisnika(List<User> users, String username, String password);
    void logout(List<User> users, String username);
    double ukupnoRealizovano(List<Trosak> troskovi);
}
```

---

### `TrosakImpl.java`

```java
public class TrosakImpl implements TrosakService {

    @Override
    public List<Trosak> dodajNovi(List<Trosak> expenses, String name, double value) {
        Trosak t = new Trosak(name, value);
        expenses.add(t);
        return expenses;
    }

    @Override
    public List<Trosak> izmeniStatus(List<Trosak> expenses, String name) {
        for (Trosak expense : expenses) {
            if (expense.getNaziv().equals(name)) {
                expense.setStatus(
                    expense.getStatus() == Status.PLANIRAN ? Status.REALIZOVAN : Status.PLANIRAN
                );
            }
        }
        return expenses;
    }

    @Override
    public List<Trosak> izbrisi(List<Trosak> expenses, String name) {
        Trosak e = null;
        for (Trosak expense : expenses) {
            if (expense.getNaziv().equals(name)) e = expense;
        }
        if (e != null) expenses.remove(e);
        return expenses;
    }
}
```
> Implementacija servisa za troškove. `izmeniStatus` toggle-uje status između PLANIRAN/REALIZOVAN. `izbrisi` traži trosak po nazivu, pa ga uklanja.

---

### `UserServiceImpl.java`

```java
public class UserServiceImpl implements UserService {

    @Override
    public User pronadjiKorisnika(List<User> users, String username, String password) {
        for (User u : users) {
            if (u.getUsername().equals(username) && u.getPassword().equals(password))
                return u;
        }
        return null;
    }

    @Override
    public void logout(List<User> users, String username) {
        for (User user : users) {
            if (user.getUsername().equals(username)) user.setOnline(false);
        }
    }

    @Override
    public double ukupnoRealizovano(List<Trosak> troskovi) {
        double ukupno = 0;
        for (Trosak t : troskovi) {
            if (t.getStatus() == Status.REALIZOVAN) ukupno += t.getIznos();
        }
        return ukupno;
    }
}
```
> `pronadjiKorisnika` vraća korisnika ili `null`. `ukupnoRealizovano` sabira iznose samo onih troškova koji su u statusu REALIZOVAN.

---

## Spring konfiguracija

### `AppConfig.java`

```java
@Configuration
@ComponentScan(basePackages = {"rs.fon.drugiklk"})
public class AppConfig {

    @Bean
    public UserService getUserService() {
        return new UserServiceImpl();
    }

    @Bean
    public TrosakService getTrosakService() {
        return new TrosakImpl();
    }

    @Bean(name = "mapaAkcija")
    public Map<String, AbstractAction> mapaAkcija(
            LoginAction loginAction, LogoutAction logout,
            StatusAction status, AddAction add, ConfirmAddAction confirmadd,
            DeleteAction del, ConfirmDeleteAction cd) {
        Map<String, AbstractAction> mapa = new HashMap<>();
        mapa.put("/login",         loginAction);
        mapa.put("/logout",        logout);
        mapa.put("/status",        status);
        mapa.put("/add",           add);
        mapa.put("/confirmadd",    confirmadd);
        mapa.put("/delete",        del);
        mapa.put("/confirmdelete", cd);
        return mapa;
    }
}
```
> `@Configuration` – označava Spring konfiguracionu klasu.  
> `@ComponentScan` – Spring automatski pronalazi sve `@Component` klase u paketu.  
> Bean `mapaAkcija` mapira URL putanje na action objekte — servlet je koristi da zna koju akciju da pozove.

---

## Listener

### `AppContextListener.java`

```java
public class AppContextListener implements ServletContextListener {

    @Override
    public void contextInitialized(ServletContextEvent sce) {
        // Pokretanje Spring IoC kontejnera
        ApplicationContext ioc = new AnnotationConfigApplicationContext(AppConfig.class);
        sce.getServletContext().setAttribute("ioc", ioc);

        // Inicijalni podaci – korisnici
        List<User> korisnici = new ArrayList<>();
        korisnici.add(new User("ana", "ana123", false));
        korisnici.add(new User("marko", "marko123", false));
        sce.getServletContext().setAttribute("korisnici", korisnici);

        // Troškovi po korisniku (Map<username, List<Trosak>>)
        List<Trosak> aniniTroskovi = new ArrayList<>();
        aniniTroskovi.add(new Trosak("Struja", 5000.0, Status.REALIZOVAN));
        aniniTroskovi.add(new Trosak("Internet", 2000.0));
        aniniTroskovi.add(new Trosak("Kirija", 30000.0));

        Map<String, List<Trosak>> troskoviPoKorisniku = new HashMap<>();
        troskoviPoKorisniku.put("ana", aniniTroskovi);
        troskoviPoKorisniku.put("marko", new ArrayList<>());
        sce.getServletContext().setAttribute("troskoviPoKorisniku", troskoviPoKorisniku);
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {}
}
```
> Izvršava se pri **pokretanju aplikacije** (pre bilo kakvog zahteva). Kreira Spring kontejner i čuva ga u `ServletContext` pod ključem `"ioc"`. Takođe inicijalizuje listu korisnika i mapu troškova koji su dostupni svim delovima aplikacije (`applicationScope` u JSP-u).

---

## Servlet

### `AppServlet.java`

```java
@WebServlet(name = "AppServlet", urlPatterns = {"/app/*"})
public class AppServlet extends HttpServlet {

    @Autowired
    @Qualifier(value = "mapaAkcija")
    Map<String, AbstractAction> mapa;

    @Override
    public void init(ServletConfig config) throws ServletException {
        ApplicationContext app =
            (ApplicationContext) config.getServletContext().getAttribute("ioc");
        app.getAutowireCapableBeanFactory().autowireBean(this);
    }

    protected void processRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        response.setContentType("text/html;charset=UTF-8");
        String path = request.getPathInfo();         // npr. "/login"
        AbstractAction action = mapa.get(path);
        String page = action.execute(request, response);
        request.getRequestDispatcher(page).forward(request, response);
    }

    @Override
    protected void doGet(...) { processRequest(request, response); }

    @Override
    protected void doPost(...) { processRequest(request, response); }
}
```
> **Front Controller** – prima sve zahteve na `/app/*`. U `init()` ručno uzima Spring kontejner iz `ServletContext` i injektuje sebe (`autowireBean(this)`) da bi `@Autowired` polje `mapa` bilo popunjeno. `processRequest` određuje putanju, nalazi odgovarajuću akciju u mapi i forwarduje na JSP koji akcija vrati.

---

## Action sloj

### `AbstractAction.java`

```java
public abstract class AbstractAction {
    @Autowired
    protected UserService userService;
    @Autowired
    protected TrosakService trosakService;

    public String execute(HttpServletRequest request, HttpServletResponse response) {
        HttpSession session = request.getSession();
        String path = request.getPathInfo();
        boolean isLogIn    = path.equals("/login");
        boolean jeUlogovan = (session.getAttribute("ulogovan") != null);

        if (!isLogIn && !jeUlogovan) return "/login.jsp"; // guard – redirect na login

        return doExecute(request, response, session);
    }

    protected abstract String doExecute(HttpServletRequest request,
                                        HttpServletResponse response,
                                        HttpSession session);
}
```
> Bazna klasa svih akcija. `execute()` proverava autentikaciju pre svakog zahteva — ako korisnik nije ulogovan, vraća `/login.jsp`. Konkretna logika ide u `doExecute()` koji svaka podklasa implementira.

---

### `LoginAction.java` `@Component`

```java
@Component
public class LoginAction extends AbstractAction {

    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        String username = request.getParameter("username");
        String password = request.getParameter("password");
        if (username == null || password == null) return "/login.jsp";

        User user = userService.pronadjiKorisnika(
            (List<User>) session.getServletContext().getAttribute("korisnici"), username, password);

        if (user == null) {
            request.setAttribute("greska", "Ovaj korisnik ne postoji");
            return "/login.jsp";
        }
        if (user.isOnline()) {
            request.setAttribute("greska", "Korisnik je vec prijavljen");
            return "/login.jsp";
        }

        user.setOnline(true);
        session.setAttribute("ulogovan", user);

        Map<String, List<Trosak>> mapaTroskova =
            (Map<String, List<Trosak>>) session.getServletContext().getAttribute("troskoviPoKorisniku");
        List<Trosak> listaTroskova = mapaTroskova.get(username);
        session.setAttribute("ulogovanTroskovi", listaTroskova);

        double ukupno = userService.ukupnoRealizovano(listaTroskova);
        session.setAttribute("ukupno", ukupno);

        return "/WEB-INF/main.jsp";
    }
}
```
> Autentikuje korisnika. Proverava i da li je korisnik već online (sprečava dvostruku prijavu). Učitava listu troškova i ukupni iznos realizovanih u sesiju.

---

### `LogoutAction.java` `@Component`

```java
@Component
public class LogoutAction extends AbstractAction {

    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        User u = (User) session.getAttribute("ulogovan");
        List<User> users = (List<User>) session.getServletContext().getAttribute("korisnici");
        userService.logout(users, u.getUsername()); // postavi online = false
        session.invalidate();                        // uništi sesiju
        return "/login.jsp";
    }
}
```
> Postavlja `online = false` za korisnika u globalnoj listi, pa invaliduje HTTP sesiju.

---

### `AddAction.java` `@Component`

```java
@Component
public class AddAction extends AbstractAction {

    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        Trosak e = new Trosak(
            request.getParameter("naziv"),
            Double.parseDouble(request.getParameter("iznos"))
        );
        session.setAttribute("trosak", e);
        return "/WEB-INF/add.jsp"; // stranica za potvrdu
    }
}
```
> Čita parametre forme, kreira `Trosak` objekat i čuva ga u sesiji kao privremeni. Vraća stranicu za potvrdu — trosak se ne dodaje odmah.

---

### `ConfirmAddAction.java` `@Component`

```java
@Component
public class ConfirmAddAction extends AbstractAction {

    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        List<Trosak> troskovi = (List<Trosak>) session.getAttribute("ulogovanTroskovi");
        Trosak t = (Trosak) session.getAttribute("trosak");
        troskovi = trosakService.dodajNovi(troskovi, t.getNaziv(), t.getIznos());
        session.setAttribute("ulogovanTroskovi", troskovi);
        return "/WEB-INF/main.jsp";
    }
}
```
> Potvrda dodavanja. Uzima privremeni trosak iz sesije i zvanično ga dodaje u listu korisnikovih troškova.

---

### `DeleteAction.java` `@Component`

```java
@Component
public class DeleteAction extends AbstractAction {

    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        Trosak t = new Trosak((String) request.getParameter("naziv"));
        session.setAttribute("trosak1", t);
        return "/WEB-INF/delete.jsp"; // stranica za potvrdu brisanja
    }
}
```
> Prima naziv troška iz URL parametra, čuva ga u sesiji i forwarduje na stranicu za potvrdu brisanja.

---

### `ConfirmDeleteAction.java` `@Component`

```java
@Component
public class ConfirmDeleteAction extends AbstractAction {

    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        List<Trosak> troskovi = (List<Trosak>) session.getAttribute("ulogovanTroskovi");
        Trosak t = (Trosak) session.getAttribute("trosak1");
        troskovi = trosakService.izbrisi(troskovi, t.getNaziv());
        session.setAttribute("ulogovanTroskovi", troskovi);
        Double ukupno = userService.ukupnoRealizovano(troskovi);
        session.setAttribute("ukupno", ukupno);
        return "/WEB-INF/main.jsp";
    }
}
```
> Briše trosak iz liste i ažurira ukupan iznos realizovanih.

---

### `StatusAction.java` `@Component`

```java
@Component
public class StatusAction extends AbstractAction {

    @Override
    protected String doExecute(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
        List<Trosak> troskovi = (List<Trosak>) session.getAttribute("ulogovanTroskovi");
        troskovi = trosakService.izmeniStatus(troskovi, request.getParameter("naziv"));
        session.setAttribute("ulogovanTroskovi", troskovi);
        Double ukupno = userService.ukupnoRealizovano(troskovi);
        session.setAttribute("ukupno", ukupno);
        return "/WEB-INF/main.jsp";
    }
}
```
> Toggle-uje status troška (PLANIRAN ↔ REALIZOVAN) i odmah ažurira ukupan iznos.

---

## Web stranice (JSP)

> **Ključni tagovi koji se koriste:**
> - `<%@page contentType="text/html" pageEncoding="UTF-8"%>` – definiše tip sadržaja i encoding JSP stranice
> - `<%@taglib prefix="c" uri="jakarta.tags.core" %>` – uvozi JSTL core tag biblioteku (potrebno za `c:forEach`, `c:if`)
> - `${...}` – EL (Expression Language) za čitanje atributa iz request/session/application scope-a
> - `${sessionScope.xxx}` – eksplicitno čitanje iz HTTP sesije
> - `${applicationScope.xxx}` – čitanje iz `ServletContext` (dostupno svim korisnicima)
> - `${pageContext.request.contextPath}` – dinamički prefix URL-a aplikacije (npr. `/DrugiKlk-1.0-SNAPSHOT`)
> - `<c:forEach var="x" items="${lista}">` – iteracija po listi
> - `<c:if test="${uslov}">` – uslovno prikazivanje sadržaja
> - `<jsp:include page="..." flush="true"/>` – uključuje drugi JSP fragment u stranicu

---

### `login.jsp` – forma za prijavu

```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<%@taglib prefix="c" uri="jakarta.tags.core" %>
<!DOCTYPE html>
<html>
<body>
    <h1>Prijava</h1>
    <form method="POST" action="${pageContext.request.contextPath}/app/login">
        <label>Username</label><input type="text" name="username">
        <label>Password</label><input type="text" name="password">
        <button type="submit">Prijavi se</button>
    </form>

    <%-- Prikazuje grešku samo ako postoji atribut "greska" u request scope-u --%>
    <c:if test="${not empty greska}">
        <h5 style="color: red">${greska}</h5>
    </c:if>
</body>
</html>
```
> Jedina javno dostupna stranica (van `WEB-INF`). `${not empty greska}` – JSTL proverava da li je atribut postoji i nije prazan string.

---

### `main.jsp` – glavni prikaz troškova

```jsp
<%@taglib prefix="c" uri="jakarta.tags.core" %>
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<body>
    <p><a href="${pageContext.request.contextPath}/app/logout">Odjava</a></p>
    <h1>Prikaz troskova</h1>
    <h2>Ukupan iznos realizovanih troskova: ${ukupno}</h2>

    <table border="1">
        <thead>
            <tr>
                <th>Naziv</th><th>Iznos</th><th>Status</th><th colspan="2">Akcija</th>
            </tr>
        </thead>
        <tbody>
            <%-- Iterira po listi troškova iz sesije --%>
            <c:forEach var="trosak" items="${ulogovanTroskovi}">
                <tr>
                    <td>${trosak.naziv}</td>
                    <td>${trosak.iznos}</td>
                    <td>${trosak.status}</td>
                    <td><a href="${pageContext.request.contextPath}/app/status?naziv=${trosak.naziv}">Izmeni status</a></td>
                    <td><a href="${pageContext.request.contextPath}/app/delete?naziv=${trosak.naziv}">Izbrisi</a></td>
                </tr>
            </c:forEach>
        </tbody>
    </table>

    <h2>Dodaj novi zadatak</h2>
    <form method="POST" action="${pageContext.request.contextPath}/app/add">
        <label>Naziv </label><input type="text" name="naziv"/><br/>
        <label>Iznos </label><input type="text" name="iznos"/><br/>
        <input type="submit" value="Dodaj"/>
    </form>

    <%-- Fragment sa listom korisnika (online/offline) --%>
    <jsp:include page="fragment/online.jsp" flush="true"/>
</body>
</html>
```
> `${ulogovanTroskovi}` – lista iz sesije, popunjena u `LoginAction`. Linkovi za status i brisanje šalju naziv troška kao query parametar. `<jsp:include>` ubacuje `online.jsp` fragment u stranicu.

---

### `add.jsp` – potvrda dodavanja troška

```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<body>
    <h1>Potvrda unosa</h1>
    <form method="POST" action="${pageContext.request.contextPath}/app/confirmadd">
        <button type="submit">Potvrdi dodavanje troska</button>
    </form>
    <div>
        <jsp:include page="fragment/online.jsp" flush="true"/>
    </div>
</body>
</html>
```
> Međukorak pre stvarnog dodavanja — korisnik mora kliknuti "Potvrdi". Trosak je već u sesiji (`trosak`), `ConfirmAddAction` ga zvanično doda.

---

### `delete.jsp` – potvrda brisanja troška

```jsp
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<body>
    <%-- Ime troška se čita direktno iz sesijskog atributa "trosak1" --%>
    <h3>Trosak koji ste izabrali je ${sessionScope.trosak1.naziv}</h3>
    <form method="POST" action="${pageContext.request.contextPath}/app/confirmdelete">
        <button type="submit">Potvrdi brisanje troska</button>
    </form>
    <div>
        <jsp:include page="fragment/online.jsp" flush="true"/>
    </div>
</body>
</html>
```
> `${sessionScope.trosak1.naziv}` – eksplicitno čitanje iz sesije (ekvivalentno sa `${trosak1.naziv}`, ali jasnije). Prikazuje koji trosak se briše pre potvrde.

---

### `fragment/online.jsp` – lista korisnika (online/offline)

```jsp
<%@taglib prefix="c" uri="jakarta.tags.core" %>
<%@page contentType="text/html" pageEncoding="UTF-8"%>
<h2>Korisnici</h2>
<ul>
    <%-- applicationScope = ServletContext, vidljiv svim korisnicima --%>
    <c:forEach var="u" items="${applicationScope.korisnici}">
        <li>${u.username}
            <c:if test="${u.online}">online</c:if>
            <c:if test="${not u.online}">offline</c:if>
        </li>
    </c:forEach>
</ul>
```
> Čita listu svih korisnika iz `applicationScope` (postavljeno u `AppContextListener`). Prikazuje se na svim zaštićenim stranicama. `${u.online}` poziva `isOnline()` metodu na `User` objektu — EL automatski prepoznaje boolean getter.

---

## web.xml

```xml
<web-app version="6.0" xmlns="https://jakarta.ee/xml/ns/jakartaee" ...>
    <!-- Registruje AppContextListener koji se poziva pri startu aplikacije -->
    <listener>
        <listener-class>rs.fon.drugiklk.listener.AppContextListener</listener-class>
    </listener>

    <!-- HTTP sesija ističe nakon 30 minuta neaktivnosti -->
    <session-config>
        <session-timeout>30</session-timeout>
    </session-config>
</web-app>
```
> Minimalni deployment descriptor. `<listener>` je ključan — bez njega `AppContextListener.contextInitialized()` se nikad ne pozove i Spring kontejner se ne kreira.

---

## Tok jednog zahteva (primer: dodavanje troška)

```
Korisnik popuni formu na main.jsp
    → POST /app/add  (naziv, iznos parametri)
        → AppServlet.processRequest()
            → mapa.get("/add") → AddAction
                → AbstractAction.execute() → provjera sesije → OK
                    → AddAction.doExecute()
                        → kreira Trosak, stavlja u sesiju kao "trosak"
                        → vraća "/WEB-INF/add.jsp"
            → forward na add.jsp (stranica za potvrdu)

Korisnik klikne "Potvrdi"
    → POST /app/confirmadd
        → ConfirmAddAction.doExecute()
            → uzima "trosak" iz sesije
            → trosakService.dodajNovi(troskovi, naziv, iznos)
            → ažurira "ulogovanTroskovi" u sesiji
            → vraća "/WEB-INF/main.jsp"
```
