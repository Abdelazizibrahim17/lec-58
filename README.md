# University Course Management System

Java 17, Spring Boot 3.5.16, Spring Web, Spring Data JPA, Bean Validation, Oracle JDBC, Lombok, and Maven.

## Project structure

```text
.
├── pom.xml
├── mvnw
├── mvnw.cmd
├── .mvn/wrapper/maven-wrapper.properties
├── README.md
├── requests.http
└── src
    ├── main
    │   ├── java/com/example/university
    │   │   ├── UniversityApplication.java
    │   │   ├── controller
    │   │   │   ├── StudentController.java
    │   │   │   ├── CourseController.java
    │   │   │   └── InstructorController.java
    │   │   ├── service
    │   │   │   ├── StudentService.java
    │   │   │   ├── CourseService.java
    │   │   │   ├── InstructorService.java
    │   │   │   └── ResponseMapper.java
    │   │   ├── repository
    │   │   │   ├── StudentRepository.java
    │   │   │   ├── CourseRepository.java
    │   │   │   └── InstructorRepository.java
    │   │   ├── entity
    │   │   │   ├── Student.java
    │   │   │   ├── Course.java
    │   │   │   └── Instructor.java
    │   │   ├── dto
    │   │   │   ├── StudentRequest.java
    │   │   │   ├── CourseRequest.java
    │   │   │   ├── InstructorRequest.java
    │   │   │   ├── StudentResponse.java
    │   │   │   ├── CourseResponse.java
    │   │   │   ├── InstructorResponse.java
    │   │   │   ├── StudentBasicResponse.java
    │   │   │   ├── CourseBasicResponse.java
    │   │   │   └── InstructorBasicResponse.java
    │   │   └── exception
    │   │       ├── ResourceNotFoundException.java
    │   │       ├── DuplicateEnrollmentException.java
    │   │       └── GlobalExceptionHandler.java
    │   └── resources/application.properties
    └── test
        ├── java/com/example/university/UniversityApiTest.java
        └── resources/application-test.properties
```

Controllers handle HTTP and validation; transactional services contain business logic. Repositories extend JpaRepository<Entity, Long>. ResponseMapper builds finite DTO graphs inside transactions; open-in-view is disabled.

Student owns the bidirectional many-to-many using @JoinTable(name = "student_course"). Its student_id and course_id columns reference students.id and courses.id; the pair is unique. Course.students uses mappedBy = "courses". Course.instructor uses @ManyToOne and instructor_id; Instructor.courses uses @OneToMany(mappedBy = "instructor"). Both sides are maintained by helper methods. There is no cascading deletion. Separate Oracle sequences generate each entity's IDs.

StudentResponse contains CourseBasicResponse objects with instructor details. CourseResponse contains basic students and an instructor. InstructorResponse contains full course responses, including enrolled students. No controller returns an entity.

Registration locks the student row during the duplicate check and insert; the join table additionally enforces uniqueness. A course may initially have no instructor, represented as null. Assigning the same instructor again is valid; assigning another replaces the previous instructor.

## Configure Oracle and run

Install JDK 17+ and have an Oracle Database service available (for example Oracle Free with service FREEPDB1). Set JAVA_HOME to your JDK if needed. Use a dedicated schema account with CREATE SESSION, CREATE TABLE, CREATE SEQUENCE privileges and a tablespace quota. The database/service and account must already exist.

In PowerShell, replace these placeholders with your connection details:

```powershell
$env:ORACLE_URL = 'jdbc:oracle:thin:@//localhost:1521/FREEPDB1'
$env:ORACLE_USERNAME = '<schema-user>'
$env:ORACLE_PASSWORD = '<schema-password>'
$env:JPA_DDL_AUTO = 'update'
.\mvnw.cmd spring-boot:run
```

In Bash:

```bash
export ORACLE_URL='jdbc:oracle:thin:@//localhost:1521/FREEPDB1'
export ORACLE_USERNAME='<schema-user>'
export ORACLE_PASSWORD='<schema-password>'
export JPA_DDL_AUTO=update
sh mvnw spring-boot:run
```

No real credentials are stored. application.properties uses environment placeholders, OracleDialect, and oracle.jdbc.OracleDriver. Hibernate update creates the tables, sequences, join table and foreign keys on first startup. For a schema managed separately, set JPA_DDL_AUTO=validate after provisioning it. This project does not require manually executing join-table SQL: its schema is explicitly defined by @JoinTable.

The application listens at http://localhost:8080. POST creation returns 201 with a Location header. Other successful endpoints return 200. List endpoints return [] when empty. Names are required and limited to 100 characters; titles to 200; emails are required, validated and limited to 254; optional descriptions to 2000.

## Build and tests

Verified in this workspace with Maven 3.9.11 on Java 21, compiling for Java 17: BUILD SUCCESS; 12 tests, 0 failures, 0 errors, 0 skipped. The executable artifact is target/university-course-management-system-0.0.1-SNAPSHOT.jar. A live Oracle instance was not available for verification.

```powershell
.\mvnw.cmd clean verify
java -jar target/university-course-management-system-0.0.1-SNAPSHOT.jar
```

On Bash use sh mvnw clean verify. An installed Maven can also run mvn clean verify. First use requires network access to download Maven/dependencies.

Tests run with H2 in Oracle compatibility mode and do not require Oracle credentials. They exercise every API through MockMvc and real services/JPA, without wrapping tests in a transaction, so responses must work with open-in-view disabled. They verify persistence, finite JSON, instructor reassignment, 404/400 responses, duplicate enrollment, and schema uniqueness/foreign keys. H2 tests do not establish connectivity or runtime compatibility with your actual Oracle installation.

## Endpoint examples

All examples use Content-Type: application/json for requests containing JSON. Run these examples in the order below on an empty database. IDs are illustrative: replace 1 with the IDs returned by your create calls. GET, registration and assignment requests have no body.

| Method | Endpoint | Success |
| --- | --- | --- |
| POST | /api/students | 201 |
| POST | /api/instructors | 201 |
| POST | /api/courses | 201 |
| PUT | /api/courses/{courseId}/instructor/{instructorId} | 200 |
| POST | /api/students/{studentId}/courses/{courseId} | 200 |
| GET | /api/students | 200 |
| GET | /api/students/{id} | 200 |
| GET | /api/courses | 200 |
| GET | /api/courses/{id} | 200 |
| GET | /api/instructors | 200 |
| GET | /api/instructors/{id} | 200 |
| GET | /api/instructors/{id}/courses | 200 |

### POST /api/students

Request:

```http
POST /api/students HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}
```

Response: 201 CREATED (Location: /api/students/1)

```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com",
  "courses": []
}
```

### POST /api/instructors

Request:

```http
POST /api/instructors HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "name": "Dr. Smith",
  "email": "smith@example.com"
}
```

Response: 201 CREATED (Location: /api/instructors/1)

```json
{
  "id": 1,
  "name": "Dr. Smith",
  "email": "smith@example.com",
  "courses": []
}
```

### POST /api/courses

Request:

```http
POST /api/courses HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "title": "Java Programming",
  "description": "Java fundamentals"
}
```

Response: 201 CREATED (Location: /api/courses/1)

```json
{
  "id": 1,
  "title": "Java Programming",
  "description": "Java fundamentals",
  "instructor": null,
  "students": []
}
```

### PUT /api/courses/1/instructor/1

Request:

```http
PUT /api/courses/1/instructor/1 HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
{
  "id": 1,
  "title": "Java Programming",
  "description": "Java fundamentals",
  "instructor": {
    "id": 1,
    "name": "Dr. Smith",
    "email": "smith@example.com"
  },
  "students": []
}
```

### POST /api/students/1/courses/1

Request:

```http
POST /api/students/1/courses/1 HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com",
  "courses": [
    {
      "id": 1,
      "title": "Java Programming",
      "description": "Java fundamentals",
      "instructor": {
        "id": 1,
        "name": "Dr. Smith",
        "email": "smith@example.com"
      }
    }
  ]
}
```

### GET /api/students

Request:

```http
GET /api/students HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
[
  {
    "id": 1,
    "name": "John Doe",
    "email": "john@example.com",
    "courses": [
      {
        "id": 1,
        "title": "Java Programming",
        "description": "Java fundamentals",
        "instructor": {
          "id": 1,
          "name": "Dr. Smith",
          "email": "smith@example.com"
        }
      }
    ]
  }
]
```

### GET /api/students/1

Request:

```http
GET /api/students/1 HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
{
  "id": 1,
  "name": "John Doe",
  "email": "john@example.com",
  "courses": [
    {
      "id": 1,
      "title": "Java Programming",
      "description": "Java fundamentals",
      "instructor": {
        "id": 1,
        "name": "Dr. Smith",
        "email": "smith@example.com"
      }
    }
  ]
}
```

### GET /api/courses

Request:

```http
GET /api/courses HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
[
  {
    "id": 1,
    "title": "Java Programming",
    "description": "Java fundamentals",
    "instructor": {
      "id": 1,
      "name": "Dr. Smith",
      "email": "smith@example.com"
    },
    "students": [
      {
        "id": 1,
        "name": "John Doe",
        "email": "john@example.com"
      }
    ]
  }
]
```

### GET /api/courses/1

Request:

```http
GET /api/courses/1 HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
{
  "id": 1,
  "title": "Java Programming",
  "description": "Java fundamentals",
  "instructor": {
    "id": 1,
    "name": "Dr. Smith",
    "email": "smith@example.com"
  },
  "students": [
    {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com"
    }
  ]
}
```

### GET /api/instructors

Request:

```http
GET /api/instructors HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
[
  {
    "id": 1,
    "name": "Dr. Smith",
    "email": "smith@example.com",
    "courses": [
      {
        "id": 1,
        "title": "Java Programming",
        "description": "Java fundamentals",
        "instructor": {
          "id": 1,
          "name": "Dr. Smith",
          "email": "smith@example.com"
        },
        "students": [
          {
            "id": 1,
            "name": "John Doe",
            "email": "john@example.com"
          }
        ]
      }
    ]
  }
]
```

### GET /api/instructors/1

Request:

```http
GET /api/instructors/1 HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
{
  "id": 1,
  "name": "Dr. Smith",
  "email": "smith@example.com",
  "courses": [
    {
      "id": 1,
      "title": "Java Programming",
      "description": "Java fundamentals",
      "instructor": {
        "id": 1,
        "name": "Dr. Smith",
        "email": "smith@example.com"
      },
      "students": [
        {
          "id": 1,
          "name": "John Doe",
          "email": "john@example.com"
        }
      ]
    }
  ]
}
```

### GET /api/instructors/1/courses

Request:

```http
GET /api/instructors/1/courses HTTP/1.1
Host: localhost:8080
```

Response: 200 OK

```json
[
  {
    "id": 1,
    "title": "Java Programming",
    "description": "Java fundamentals",
    "instructor": {
      "id": 1,
      "name": "Dr. Smith",
      "email": "smith@example.com"
    },
    "students": [
      {
        "id": 1,
        "name": "John Doe",
        "email": "john@example.com"
      }
    ]
  }
]
```

## Error examples

Missing students, courses or instructors (including relationship targets) return 404:

```json
{"timestamp":"2026-09-30T12:00:00Z","status":404,"message":"Student not found with id 999","path":"/api/students/999","errors":{}}
```

Repeated registration returns 400:

```json
{"timestamp":"2026-09-30T12:00:00Z","status":400,"message":"Student 1 is already registered in course 1","path":"/api/students/1/courses/1","errors":{}}
```

Invalid student request `{"name":"","email":"invalid"}` returns 400:

```json
{"timestamp":"2026-09-30T12:00:00Z","status":400,"message":"Validation failed","path":"/api/students","errors":{"name":"must not be blank","email":"must be a well-formed email address"}}
```

Malformed JSON and nonnumeric path IDs return 400 with message "Malformed JSON or invalid path parameter". Timestamps vary per request.
