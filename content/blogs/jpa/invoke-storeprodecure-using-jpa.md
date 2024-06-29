+++
categories = "blogs"
disableToc = false
title = "Invoking Oracle Stored Procedure with Spring Data JPA"
weight = 3
+++


### Introduction

In this guide, I will walk you through the steps to invoke an Oracle stored procedure from a Spring Boot application using Spring Data JPA. The stored procedure retrieves course details along with associated subjects using a cursor.

### Prerequisites

Ensure you have the following setup:
- Oracle Database with schema "IAMROOT"
- Java Development Kit (JDK) installed
- Maven or Gradle for dependency management
- Spring Boot project initialized

##### Step 1: Database Setup

Create the required tables in Oracle using the following DDL queries:

```sql
CREATE TABLE IAMROOT.COURSE_TBL (
    COURSE_ID NUMBER(10,0) NOT NULL,
    COURSE_CODE VARCHAR2(255 CHAR),
    COURSE_NAME VARCHAR2(255 CHAR),
    SUBJECTS_FINALIZED NUMBER(1,0),
    PRIMARY KEY (COURSE_ID)
);

CREATE TABLE IAMROOT.SUBJECT_TBL (
    SUBJECT_ID NUMBER(10,0) NOT NULL,
    SUBJECT_CODE VARCHAR2(255 CHAR),
    SUBJECT_NAME VARCHAR2(255 CHAR),
    PRIMARY KEY (SUBJECT_ID),
    CONSTRAINT UK_SUBJECT_CONSTRAINT_1 UNIQUE (SUBJECT_CODE)
);

CREATE TABLE IAMROOT.COURSE_SUBJECT_MAPPING_TBL (
    ID NUMBER(10,0) NOT NULL,
    COURSE_ID NUMBER(10,0),
    SUBJECT_ID NUMBER(10,0),
    PRIMARY KEY (ID),
    CONSTRAINT FKMQ7WCF3AB0LQ4F15SMTBNILF4 FOREIGN KEY (SUBJECT_ID) REFERENCES IAMROOT.SUBJECT_TBL(SUBJECT_ID),
    CONSTRAINT FK7X1P4MSBKW794AJSDE9XCJ31Q FOREIGN KEY (COURSE_ID) REFERENCES IAMROOT.COURSE_TBL(COURSE_ID)
);

```

##### Step 2: Insert Data

Use the following DML queries to populate the tables with sample data:

```sql
INSERT INTO COURSE_TBL (COURSE_ID, COURSE_CODE, COURSE_NAME, SUBJECTS_FINALIZED)
SELECT 1, 'STD-1', 'STD-1', 0 FROM dual UNION ALL
SELECT 2, 'STD-2', 'STD-2', 0 FROM dual;

INSERT INTO SUBJECT_TBL (SUBJECT_ID, SUBJECT_CODE, SUBJECT_NAME)
SELECT 1, 'MATH', 'MATHEMATICS' FROM dual UNION ALL
SELECT 2, 'SCI', 'SCIENCE' FROM dual UNION ALL
SELECT 3, 'GK', 'GENERAL KNOWLEDGE' FROM dual UNION ALL
SELECT 4, 'ENG', 'ENGLISH' FROM dual UNION ALL
SELECT 5, 'HIN', 'HINDI' FROM dual UNION ALL
SELECT 6, 'ORI', 'ORIYA' FROM dual;

--This query will insert mappings for all subjects (SUBJECT_ID) to all courses (COURSE_ID) listed in COURSE_TBL and SUBJECT_TBL. The ROW_NUMBER() function is used to generate unique IDs for each mapping row based on the order of COURSE_ID and SUBJECT_ID.

INSERT INTO COURSE_SUBJECT_MAPPING_TBL (ID, COURSE_ID, SUBJECT_ID)
SELECT
    ROW_NUMBER() OVER (ORDER BY c.COURSE_ID, s.SUBJECT_ID) AS ID,
    c.COURSE_ID,
    s.SUBJECT_ID
FROM
    COURSE_TBL c
    CROSS JOIN SUBJECT_TBL s;

```

##### Step 3: Create Oracle Stored Procedure

Define the stored procedure GETCOURSEWITHSUBJECTS in Oracle:

```sql
CREATE OR REPLACE PROCEDURE GETCOURSEWITHSUBJECTS(
    COURSEID IN NUMBER,
    STATUSCODE OUT NUMBER,
    STATUSMESSAGE OUT VARCHAR2,
    COURSECURSOR OUT SYS_REFCURSOR
) AS
BEGIN
    BEGIN
        OPEN COURSECURSOR FOR
            SELECT
                C.COURSE_ID,
                C.COURSE_CODE,
                C.COURSE_NAME,
                S.SUBJECT_ID,
                S.SUBJECT_CODE,
                S.SUBJECT_NAME
            FROM
                COURSE_TBL C
            JOIN
                COURSE_SUBJECT_MAPPING_TBL CM ON C.COURSE_ID = CM.COURSE_ID
            JOIN
                SUBJECT_TBL S ON CM.SUBJECT_ID = S.SUBJECT_ID
            WHERE
                C.COURSE_ID = COURSEID;
        STATUSCODE := 0;
        STATUSMESSAGE := 'Success';
    EXCEPTION
        WHEN OTHERS THEN
            STATUSCODE := 1;
            STATUSMESSAGE := 'SQL Error occurred: ' || SQLERRM;
    END;
END GETCOURSEWITHSUBJECTS;
```

##### Step 4: Java Entities and DTO
Create a DTO class CourseSubjectDTO to map the result set:

```java
import javax.persistence.*;

@Data
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode
@ToString
public class CourseSubjectDTO {
    @Column(name = "COURSE_ID")
    private String courseId;

    @Column(name = "COURSE_CODE")
    private String courseCode;

    @Column(name = "COURSE_NAME")
    private String courseName;

    @Column(name = "SUBJECT_ID")
    private String subjectId;

    @Column(name = "SUBJECT_CODE")
    private String subjectCode;

    @Column(name = "SUBJECT_NAME")
    private String subjectName;
}
```

##### Step 5: Spring Data JPA Configuration

Configure the stored procedure call in CourseProcedureConfig:


```java
import javax.persistence.*;

@Entity
@SqlResultSetMapping(
    name = "CourseWithSubjectsDTOMapping",
    classes = {
        @ConstructorResult(
            targetClass = CourseSubjectDTO.class,
            columns = {
                @ColumnResult(name = "COURSE_ID", type = String.class),
                @ColumnResult(name = "COURSE_CODE", type = String.class),
                @ColumnResult(name = "COURSE_NAME", type = String.class),
                @ColumnResult(name = "SUBJECT_ID", type = String.class),
                @ColumnResult(name = "SUBJECT_CODE", type = String.class),
                @ColumnResult(name = "SUBJECT_NAME", type = String.class)
            }
        )
    }
)
@NamedStoredProcedureQuery(
    name = "CourseProcedureConfig.getCourseWithSubjects",
    procedureName = "GETCOURSEWITHSUBJECTS",
    resultSetMappings = "CourseWithSubjectsDTOMapping",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "COURSEID", type = Integer.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "STATUSCODE", type = Integer.class),
        @StoredProcedureParameter(mode = ParameterMode.OUT, name = "STATUSMESSAGE", type = String.class),
        @StoredProcedureParameter(mode = ParameterMode.REF_CURSOR, name = "COURSECURSOR", type = Void.class)
    }
)
public class CourseProcedureConfig {
    @Id
    private Long id; // Dummy field for JPA entity requirement
}
```
##### Step 6: Spring Data JPA Repository
Create a repository interface to call the stored procedure:

```java
import org.springframework.data.jpa.repository.query.Procedure;
import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Transactional
public interface CourseRepository extends CrudRepository<CourseProcedureConfig, Long> {

    @Procedure(name = "CourseProcedureConfig.getCourseWithSubjects")
    List<CourseSubjectDTO> getCourseWithSubjects(@Param("COURSEID") Integer courseId);
}
```

Step 7: Controller
Create a controller to expose the stored procedure invocation as a REST endpoint:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping(value = "/api/v1/courses")
@Transactional
public class CourseController {

    @Autowired
    private CourseRepository courseRepository;

    @GetMapping("/{courseId}")
    public ResponseEntity<List<CourseSubjectDTO>> findCourseById(@PathVariable(name = "courseId") Integer courseId) {
        List<CourseSubjectDTO> courseDTOS = courseRepository.getCourseWithSubjects(courseId);
        return new ResponseEntity<>(courseDTOS, HttpStatus.OK);
    }
}
```

### Conclusion
This guide has walked you through setting up an Oracle database schema, creating tables, inserting sample data, defining a stored procedure, and invoking it using Spring Data JPA in a Spring Boot application. You should now have a working setup to retrieve course details with associated subjects using Oracle stored procedures.