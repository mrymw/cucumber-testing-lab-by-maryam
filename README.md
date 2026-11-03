# Pet Clinic – Cucumber Lab

A REST API for a vet clinic, built with Spring Boot, Spring Data JPA, PostgreSQL and JWT authentication.
Owners log in, register their **pets**, and book **appointments** for each pet.

The API works, but it has **no tests yet**. In this lab you set up Cucumber yourself and write the tests, step by step.

---

## 1. Get the app running

1. Clone this repo and open it in IntelliJ.
2. Create a PostgreSQL database called `petclinic`. The app expects it on port `5432`, with a user and password. Change these in `src/main/resources/application.properties` if your setup is different.
3. Run `PetClinicApplication` once. The tables are created and the sample data is added.
4. In Postman, log in as Layla and list her pets, so you know what the API sends back:
   - `POST http://localhost:8080/auth/users/login` with `{ "email": "layla@mail.com", "password": "password123" }`
   - copy the token from `message`
   - `GET http://localhost:8080/api/pets` with the header `Authorization: Bearer <token>`

### Sample accounts

| Login email | Password | Pets |
|---|---|---|
| layla@mail.com | password123 | Mishmish (cat, 3 appointments), Coco (dog, 2) |
| omar@mail.com | password123 | Rocky (dog, 2) |
| huda@mail.com | password123 | Kiwi (parrot, 1), Snowball (rabbit, 1) |

Each owner only sees their own pets. To reset the data, drop the tables (or the `petclinic` database) and restart the app.

---

## 2. Lab rules (read these first)

1. **Story first.** Write the scenario in the `.feature` file, run it, and see it fail or show as *undefined*. Only then write the Java.
2. **It must pass twice in a row.** Run `mvn test` two times. Anything a scenario creates must be deleted afterwards.
3. **Never delete or change the sample pets** (Mishmish, Coco, Rocky, Kiwi, Snowball).
4. **No hard-coded ids** like `/api/pets/3`. Get ids from responses and keep them in `ScenarioState`.
5. **Reuse steps.** If a sentence you need already exists, use it. Don't write a second version.
6. **Readable stories.** Every feature file starts with `Feature:` and an *As a / I want / So that* description.

---

## 3. The lab, step by step

Tick each box as you go. Run the tests after every step.

### Part 0 – Set up Cucumber

-  **0.1 Look at the libraries.** Open `pom.xml` and find these 5 test libraries. Make sure they exist:
  `cucumber-java`, `cucumber-spring`, `cucumber-junit-platform-engine`, `junit-platform-suite`, `rest-assured`.
  
-  **0.2 Create the folders.**
  ```
  src/test/java/com/ga/petclinic/cucumber   ← your Java test classes
  src/test/resources/features              ← your .feature files
  ```
-  **0.3 Create the runner** in the `cucumber` package. It needs:
  - `@Suite`
  - `@IncludeEngines("cucumber")`
  - `@SelectClasspathResource("features")`
  - `@ConfigurationParameter(key = "cucumber.glue", value = "com.ga.petclinic.cucumber")`
  - `@ConfigurationParameter(key = "cucumber.plugin", value = "pretty, html:target/cucumber-report.html")`

  ⚠️ The class name must **end in `Test`** (for example `RunCucumberTest`), or `mvn test` skips it without any error.
-  **0.4 Create the Spring setup class.**
  - Mark it `@CucumberContextConfiguration` (only ONE class may have this).
  - Start the real app with `@SpringBootTest(webEnvironment = DEFINED_PORT, properties = "server.port=8081")`.
  - Add a Cucumber `@Before` hook that sets `RestAssured.baseURI = "http://localhost"` and `RestAssured.port = 8081`.
-  **0.5 Create `ScenarioState` (the notebook).**
  - Annotate it with `@ScenarioScope` (a fresh notebook for every scenario).
  - Fields: `token`, `response`, `petId`, `appointmentId`.
  - A method `request()` that returns `RestAssured.given()` with the header `Authorization: Bearer <token>` already added.
  - Register it with `@Import(ScenarioState.class)` on your Spring setup class.

  ⚠️ Don't put `@Component` on it, or `PetClinicApplicationTests` fails with *"No Scope registered for scope name 'cucumber-glue'"*.
-  **0.6 Write the first story.** Create `src/test/resources/features/pets.feature` with exactly this:
  ```gherkin
  Feature: My pets
    As a pet owner
    I want to keep track of my pets
    So that the clinic always has their details

    Background:
      Given I am logged in as "layla@mail.com" with password "password123"

    Scenario: Layla sees her own pets
      When I ask for my pets
      Then the response status is 200
      And I see a pet called "Mishmish"
      And I see a pet called "Coco"
  ```
  Run it. Every step should show as **undefined**.
-  **0.7 Write the first steps.**
  - `AuthSteps`: `Given I am logged in as {string} with password {string}` (fail clearly if login isn't `200`, and save the token), and `Then the response status is {int}`.
  - `PetSteps`: `When I ask for my pets` and `Then I see a pet called {string}`.
  - In `PetSteps`, add an `@After` hook that deletes the pet in `petId` if there is one.
-  **✅ Checkpoint:** `mvn test` runs 1 scenario and it's green.

### Part 1 – Logging in (`auth.feature`)

-  **Rule 1:** all three sample owners can log in and get a real JWT. Write it as **one `Scenario Outline`** with an `Examples` table of emails. Check the status is `200` *and* that the token has 3 parts separated by dots.
-  **Rule 2:** wrong details give `401`. **One `Scenario Outline`** with at least 3 rows: a wrong password, an email that doesn't exist, and an empty password.
-  **Rule 3:** asking for pets without a token gives `403`. (Start from `given()`, not `state.request()`.)
-  **✅ Checkpoint:** all scenarios green, twice in a row.

### Part 2 – Pets (`pets.feature`)

Write one step that adds a pet from a sentence like
`When I add a Hamster called "Peanut" weighing 0.2 kg`
(`{word}` matches one word, `{double}` matches a decimal), and reuse it below.

-  **Rule 4:** Layla adds a new pet → `201`, and it appears in her list.
-  **Rule 5:** Layla can't add a second pet called "Mishmish" → `409`.
-  **Rule 6:** a pet must have a name → `400`.
-  **Rule 7:** weight must be more than 0. **`Scenario Outline`** with the weights `0` and `-5` → `400` for both.
-  **Rule 8:** Layla renames one of her new pets (`200`, the new name is saved), then deletes it (`200`, and afterwards opening it gives `404`).
  Tip: our PUT replaces **every** field, so GET the pet first and send it all back with only the name changed.
-  **Rule 9:** Omar's dog "Rocky" isn't in Layla's list, and opening it with Layla's token gives `404`.
  Tip: log in as Omar inside the step only to find Rocky's id. GPath can search a list: `find { it.name == 'Rocky' }.id`
-  **✅ Checkpoint:** green twice in a row, and the sample pets are untouched.

### Part 3 – Appointments (`appointments.feature` + `AppointmentSteps`)

Log in as Layla in the `Background`.

-  **Rule 10:** Mishmish's **upcoming** appointments (`GET /api/pets/{petId}/appointments?done=false`) are, in order: *Dental cleaning*, then *Yearly check-up*. Use the sample pet Mishmish, and make sure your cleanup never deletes her.
  Tip: in REST Assured, add `.queryParam("done", false)` before `.when()`.
-  **Rule 11:** appointments booked in a random order come back sorted by date, then time. Create a **new** pet and book all of these in **one step** using a data table:
  ```gherkin
  | date       | time  | reason         | vet        | cost |
  | 2027-02-10 | 11:00 | Booster shot   | Dr. Amina  | 25.0 |
  | 2027-01-05 | 09:00 | Microchipping  | Dr. Salman | 40.0 |
  | 2027-01-05 | 08:00 | First check-up | Dr. Salman | 30.0 |
  ```
  Then check the order with a one-column table: *First check-up*, *Microchipping*, *Booster shot*.
-  **Rule 12:** after a visit is marked as **done**, it no longer shows in the upcoming list. Find the appointment to mark **by its reason**, not by "the last one I booked".
-  **Rule 13:** a visit can't have a negative cost → `400`.
-  **✅ Checkpoint:** everything green, twice in a row.

#### Hints for Part 3
- A table with a header row arrives in Java as `List<Map<String, String>>`. A one-column table with no header arrives as `List<String>`.
- A `Map<String, Long>` in `ScenarioState` can remember appointment ids by their reason.
- To work with Mishmish without deleting her, keep "the pet I'm working with" separate from "the pets I created".

---

## Troubleshooting

| What you see | Fix |
|---|---|
| `mvn test` runs 0 scenarios | The runner's name must end in `Test` |
| Every step is "undefined" | `cucumber.glue` must match your package exactly, and the sentence must match word for word |
| "No Scope registered for scope name 'cucumber-glue'" | Remove `@Component` from `ScenarioState` and use `@Import` |
| "Port 8081 was already in use" | Stop the previous test run in IntelliJ |
| Login fails in every scenario | Start the app once so the sample data exists |
| `409` when creating your test pet | An earlier run crashed before cleanup. Delete that pet in Postman, then rerun |

---

## API reference

### Auth (public)
| Method | URL | Body |
|---|---|---|
| POST | `/auth/users/register` | `{ "username": "layla", "emailAddress": "layla@mail.com", "password": "password123" }` |
| POST | `/auth/users/login` | `{ "email": "layla@mail.com", "password": "password123" }` → the JWT is in `message`. Wrong details give `401`. |

Every other request needs the header `Authorization: Bearer <token>`.

### Pets (only your own pets)
| Method | URL | Body |
|---|---|---|
| POST | `/api/pets` | `{ "name": "Peanut", "species": "Hamster", "breed": "Syrian", "birthDate": "2025-01-10", "weightKg": 0.2 }` → `201` |
| GET | `/api/pets` | – |
| GET | `/api/pets/{petId}` | – |
| PUT | `/api/pets/{petId}` | same shape as POST (every field is replaced) |
| DELETE | `/api/pets/{petId}` | – |

Rules: a pet needs a `name`, and `weightKg` must be more than 0 (otherwise `400`). You can't have two pets with the same name (`409`).

### Appointments (inside a pet)
| Method | URL | Body |
|---|---|---|
| POST | `/api/pets/{petId}/appointments` | `{ "date": "2027-01-05", "time": "09:00", "reason": "Vaccination", "vetName": "Dr. Amina", "cost": 25.0, "done": false }` → `201` |
| GET | `/api/pets/{petId}/appointments` | – (sorted by date, then time) |
| GET | `/api/pets/{petId}/appointments?done=false` | only visits that haven't happened yet |
| GET | `/api/pets/{petId}/appointments/{appointmentId}` | – |
| PUT | `/api/pets/{petId}/appointments/{appointmentId}` | same shape as POST |
| DELETE | `/api/pets/{petId}/appointments/{appointmentId}` | – |

Rules: an appointment needs a `date`, and `cost` can't be negative (otherwise `400`).

### Status codes
| Code | Meaning |
|---|---|
| `200` / `201` | It worked / it was created |
| `400` | Missing or invalid data |
| `401` | Wrong email or password at login |
| `403` | No token, or an invalid one |
| `404` | The pet or appointment doesn't exist, or belongs to someone else |
| `409` | You already have a pet with that name, or the email is already registered |

### Project structure
```
com.ga.petclinic
├── controller   → REST endpoints (UserController, PetController, AppointmentController)
├── service      → business logic and validation
├── repository   → database access (Spring Data JPA)
├── model        → entities: User, UserProfile, Pet, Appointment
├── security     → JWT login and request filter
├── exception    → 400, 404 and 409 errors
└── seeder       → DataSeeder (sample data)
```
