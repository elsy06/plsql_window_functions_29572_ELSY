 **SQL JOINs & Window Functions Project**

 **Course**:INSY 8311 -  Database Development with PL/SQL 

**Student**: KAMANZI SANGWA Elsy |29572| Group A

**Instructor**:Eric Maniraguha

 
 
 # Business Context 
**Organization**: African University of Excellence (AUE) 

**Department**: Academic Affairs & Student Success Office

**Industry**: Higher Education

# Data Challenge 

African University of Excellence enrolls 5,000+ students across multiple faculties and programs.

The Academic Affairs office currently tracks student performance, course enrollments, and 
faculty assignments using disconnected Excel spreadsheets. This creates significant 
challenges: 

● Unable to identify at-risk students who need academic intervention

● No visibility into course performance trends across different semesters

● Difficulty tracking faculty workload distribution and teaching effectiveness

● Cannot analyze which courses have the highest failure rates by department

● No systematic way to identify top-performing students for scholarships and awards

● Impossible to compare semester-to-semester performance improvements 

### success criteria

**1. Top 10 Students Per Department Per Semester**
    
 Window Function: RANK() or DENSE_RANK() OVER (PARTITION BY 
Department, Semester) 

 Metric: GPA ranking within department and semester

 Business Value: Identify candidates for Dean's List, scholarships, and academic awards

**2. Semester-to-Semester GPA Improvement Analysis**

 Window Function: LAG(GPA) and LEAD(GPA) 

 Metric: Percentage change in GPA between consecutive semesters 

 Business Value: Identify students needing intervention or showing improvement

**3. Student Performance Quartile Segmentation**

 Window Function: NTILE(4) OVER (ORDER BY Cumulative GPA) 

 Metric: Four performance tiers (Excellent, Good, Average, At-Risk)

 Business Value: Allocate tutoring resources and academic support strategically

**4. 3-Semester Rolling Average Course Grade**

 Window Function: AVG(Grade) OVER(PARTITION BY CourseID ORDER BY 
Semester ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) 

 Metric: Smoothed course performance trend

 Business Value: Detect courses with declining quality or consistently poor

### Database Schema Design
```sql
CREATE TABLE Student ( 
    StudentID VARCHAR(20) PRIMARY KEY, 
    FirstName VARCHAR(50) NOT NULL, 
    LastName VARCHAR(50) NOT NULL, 
    Email VARCHAR(100) UNIQUE NOT NULL, 
    DateOfBirth DATE, 
    Gender VARCHAR(10), 
    Department VARCHAR(50) NOT NULL, 
    Program VARCHAR(100) NOT NULL, 
    EntryYear INT NOT NULL, 
    CurrentLevel VARCHAR(20), -- Freshman, Sophomore, Junior, Senior 
    Status VARCHAR(20) DEFAULT 'Active' -- Active, Graduated, Suspended, Withdrawn 
); 
```
```sql
CREATE TABLE Instructor ( 
    InstructorID VARCHAR(20) PRIMARY KEY, 
    FirstName VARCHAR(50) NOT NULL, 
    LastName VARCHAR(50) NOT NULL, 
    Email VARCHAR(100) UNIQUE NOT NULL, 
    Department VARCHAR(50) NOT NULL, 
    Rank VARCHAR(30), -- Professor, Associate Professor, Assistant Professor, Lecturer 
    HireDate DATE, 
    OfficeLocation VARCHAR(50), 
    PhoneNumber VARCHAR(15) 
);
```
```sql
CREATE TABLE Course ( 
    CourseID VARCHAR(20) PRIMARY KEY, 
    CourseName VARCHAR(150) NOT NULL, 
    CourseCode VARCHAR(20) UNIQUE NOT NULL, 
    Credits INT NOT NULL CHECK (Credits BETWEEN 1 AND 6), 
    Department VARCHAR(50) NOT NULL, 
    Level INT CHECK (Level BETWEEN 100 AND 400), -- 100=Freshman, 200=Sophomore, etc. 
    Description TEXT, 
    Prerequisites VARCHAR(200) 
);
```
```sql
CREATE TABLE Enrollment ( 
    EnrollmentID INT PRIMARY KEY, 
    StudentID VARCHAR(20) NOT NULL, 
    CourseID VARCHAR(20) NOT NULL, 
    InstructorID VARCHAR(20) NOT NULL, 
    Semester VARCHAR(20) NOT NULL, -- Fall2024, Spring2024, Summer2024 
    AcademicYear INT NOT NULL, 
    Grade VARCHAR(5), -- A, A-, B+, B, B-, C+, C, D, F 
    NumericGrade DECIMAL(4,2), -- 4.0 scale 
    Status VARCHAR(20) DEFAULT 'Enrolled', -- Enrolled, Completed, Dropped, Withdrawn 
    EnrollmentDate DATE, 
    CompletionDate DATE, 
    Attendance DECIMAL(5,2), -- Percentage 
    FOREIGN KEY (StudentID) REFERENCES Student(StudentID), 
    FOREIGN KEY (CourseID) REFERENCES Course(CourseID), 
    FOREIGN KEY (InstructorID) REFERENCES Instructor(InstructorID), 
    UNIQUE (StudentID, CourseID, Semester) 
); 
```
<img width="1080" height="770" alt="ERD DIAGRAM" src="https://github.com/user-attachments/assets/bbd9fba3-b146-4ae4-90d7-b2ae7bb1bfb6" />


### PART A: SQL JOINS



### 1. INNER JOIN
```sql
SELECT

e.EnrollmentID,

s.StudentID, 

s.FirstName || ' ' || s.LastName AS StudentName,

s.Department AS StudentDept,

c.CourseCode,

c.CourseName,

c.Credits, 

i.FirstName || ' ' || i.LastName AS InstructorName, 

e.Semester,

e.Grade,

e.NumericGrade

FROM Enrollment e

INNER JOIN Student s ON e.StudentID = s.StudentID

INNER JOIN Course c ON e.CourseID = c.CourseID

INNER JOIN Instructor i ON e.InstructorID = i.InstructorID

WHERE e.Status = 'Completed'

ORDER BY e.Semester DESC, s.StudentID;
```
<img width="805" height="147" alt="INNER JOIN" src="https://github.com/user-attachments/assets/ce34c2fc-a013-4ba8-8b63-662ba2f4e15b" />

### Business Interpretation:

This query shows all completed course enrollments with full student, course, and instructor 
information. Academic advisors use this to review student transcripts, verify degree 
requirements, and analyze course completion patterns. The INNER JOIN ensures only valid 
records with existing students, courses, and instructors are displayed, eliminating any data 
integrity issues.


### 2. LEFT JOIN 
```sql
SELECT  
    s.StudentID,
    
    s.FirstName || ' ' || s.LastName AS StudentName,
    
    s.Email,
    
    s.Department,
    
    s.Program, 
    
    s.EntryYear,
    
    s.Status,
    
    COUNT(e.EnrollmentID) AS TotalEnrollments 
FROM Student s 
LEFT JOIN Enrollment e ON s.StudentID = e.StudentID

GROUP BY s.StudentID, s.FirstName, s.LastName, s.Email, s.Department, s.Program,

s.EntryYear, s.Status

HAVING COUNT(e.EnrollmentID) = 0
```
ORDER BY s.EntryYear DESC;

<img width="755" height="86" alt="LEFT JOIN" src="https://github.com/user-attachments/assets/080efce9-a3b0-4ec8-bd38-505b1773f924" />

### Business interpretation:

 This identifies registered students who have never enrolled in courses—a critical red flag for 
student retention. These students may have registered but never attended, indicating issues 
with orientation, registration processes, or financial barriers. The Student Success Office can 
proactively reach out to re-engage these students before they become dropouts. 

### RIGHT JOIN
```sql
SELECT  
    c.CourseID, 
    c.CourseCode, 
    c.CourseName, 
    c.Department, 
    c.Credits, 
    c.Level, 
    COUNT(e.EnrollmentID) AS TimesOffered 
FROM Enrollment e 
RIGHT JOIN Course c ON e.CourseID = c.CourseID 
GROUP BY c.CourseID, c.CourseCode, c.CourseName, c.Department, c.Credits, c.Level 
HAVING COUNT(e.EnrollmentID) = 0 
ORDER BY c.Department, c.Level;
```
<img width="504" height="35" alt="RIGHT JOIN" src="https://github.com/user-attachments/assets/f6f2cee3-1843-493e-a9fb-995972fd74bd" />
### Business interpretation:

 This reveals courses in the catalog that have never attracted students. These courses may be 
outdated, poorly marketed, have scheduling conflicts, or lack relevance to current programs. 
The curriculum committee should review these for potential cancellation or redesign. This helps 
optimize faculty resources and course offerings. 

### FULL OUTER JOIN
```sql
SELECT  
    s.StudentID, 
    s.FirstName || ' ' || s.LastName AS StudentName, 
    s.Department AS StudentDept, 
    c.CourseID, 
    c.CourseCode, 
    c.CourseName, 
    c.Department AS CourseDept, 
    e.EnrollmentID, 
    e.Grade 
FROM Student s 
FULL OUTER JOIN Enrollment e ON s.StudentID = e.StudentID 
FULL OUTER JOIN Course c ON e.CourseID = c.CourseID 
ORDER BY s.StudentID NULLS LAST, c.CourseID NULLS LAST;
```
<img width="859" height="235" alt="OUTER JOIN" src="https://github.com/user-attachments/assets/ab0d5e78-2428-49f5-b67b-8be937e4ef5e" />

### Business interpretation:

 This comprehensive view shows all students and all courses, highlighting both active 
enrollments and gaps. It reveals students without any courses (need intervention) and courses 
without any students (need review). This holistic perspective helps the Registrar's Office 
understand the complete enrollment landscape and identify improvement opportunities. 

### SELF JOIN 
```sql
SELECT  
    s1.StudentID AS Student1_ID, 
    s1.FirstName || ' ' || s1.LastName AS Student1_Name, 
    s2.StudentID AS Student2_ID, 
    s2.FirstName || ' ' || s2.LastName AS Student2_Name, 
    s1.Department, 
    c.CourseCode, 
    c.CourseName, 
    e1.Semester, 
    e1.Grade AS Student1_Grade, 
    e2.Grade AS Student2_Grade 
FROM Enrollment e1 
INNER JOIN Student s1 ON e1.StudentID = s1.StudentID 
INNER JOIN Enrollment e2 ON e1.CourseID = e2.CourseID  
    AND e1.Semester = e2.Semester  
    AND e1.StudentID < e2.StudentID 
INNER JOIN Student s2 ON e2.StudentID = s2.StudentID 
INNER JOIN Course c ON e1.CourseID = c.CourseID 
WHERE s1.Department = s2.Department 
ORDER BY e1.Semester DESC, s1.Department, c.CourseCode;
```
<img width="929" height="51" alt="INNER JOIN 2" src="https://github.com/user-attachments/assets/2414b4d2-6f4c-47c2-a56c-27db571fb078" />

### Business Interpretation:

 This analysis identifies peer learning opportunities by finding students from the same 
department taking the same courses together. Academic advisors can use this to form study 
groups, assign peer tutors, or create cohort-based learning communities. Comparing grades 
between peers also helps identify students who might need additional support when their 
classmates are performing better. 

### PART B- Window Functions Implementation

### Ranking Functions (dense rank) 
```sql
SELECT  
    c.CourseID, 
    c.CourseCode, 
    c.CourseName, 
    c.Department, 
    AVG(e.NumericGrade) AS AvgGrade, 
    COUNT(e.EnrollmentID) AS Enrollments, 
    DENSE_RANK() OVER (ORDER BY AVG(e.NumericGrade) ASC) AS DifficultyRank 
FROM Course c 
JOIN Enrollment e ON c.CourseID = e.CourseID 
WHERE e.Status = 'Completed' AND e.NumericGrade IS NOT NULL 
GROUP BY c.CourseID, c.CourseCode, c.CourseName, c.Department 
ORDER BY DifficultyRank;
```
<img width="732" height="125" alt="DENSE RANK" src="https://github.com/user-attachments/assets/110bd3db-4866-4f3a-bc87-03119cedc8d5" />

### Business Interprentation:

 **DENSE_RANK** reveals the most 
difficult courses based on average grades without gaps in ranking. The Dean's Office uses 
these rankings for academic honors, scholarship allocation, and curriculum difficulty 
assessment. 
 
 ### Aggregate Window Functions (SUM)
```sql
SELECT  
    e.StudentID, 
    s.FirstName || ' ' || s.LastName AS StudentName, 
    e.Semester, 
    c.Credits, 
    SUM(c.Credits) OVER ( 
        PARTITION BY e.StudentID  
        ORDER BY e.AcademicYear, e.Semester 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW 
    ) AS CumulativeCredits 
FROM Enrollment e 
JOIN Student s ON e.StudentID = s.StudentID 
JOIN Course c ON e.CourseID = c.CourseID 
WHERE e.Status = 'Completed' 
ORDER BY e.StudentID, e.AcademicYear, e.Semester;
```
<img width="460" height="158" alt="SUM" src="https://github.com/user-attachments/assets/d7eb7978-64d0-4b60-98d4-d1bab81f2031" />

### Business Interpretation:

 The running total of credits shows graduation progress for each student—critical for academic 
advising. Students with insufficient credit accumulation need intervention. The 3-semester 
moving average for courses smooths out grade fluctuations to reveal underlying teaching quality 
trends. Declining averages signal curriculum issues or instructor challenges requiring attention. 
 
 ### Navigation Functions (LAG/LEAD)
```sql
 WITH SemesterGPA AS ( 
    SELECT  
        e.StudentID, 
        s.FirstName || ' ' || s.LastName AS StudentName, 
        e.Semester, 
        e.AcademicYear, 
        AVG(e.NumericGrade) AS SemesterGPA 
    FROM Enrollment e 
    JOIN Student s ON e.StudentID = s.StudentID 
    WHERE e.Status = 'Completed' AND e.NumericGrade IS NOT NULL 
    GROUP BY e.StudentID, s.FirstName, s.LastName, e.Semester, e.AcademicYear 
) 
SELECT  
    StudentID, 
    StudentName, 
    Semester, 
    ROUND(SemesterGPA, 2) AS CurrentGPA, 
    ROUND(LAG(SemesterGPA) OVER (PARTITION BY StudentID ORDER BY AcademicYear, 
Semester), 2) AS PreviousGPA, 
    ROUND(SemesterGPA - LAG(SemesterGPA) OVER (PARTITION BY StudentID ORDER BY 
AcademicYear, Semester), 2) AS GPAChange, 
    CASE  
        WHEN SemesterGPA > LAG(SemesterGPA) OVER (PARTITION BY StudentID ORDER 
BY AcademicYear, Semester) THEN 'Improving' 
        WHEN SemesterGPA < LAG(SemesterGPA) OVER (PARTITION BY StudentID ORDER 
BY AcademicYear, Semester) THEN 'Declining' 
        ELSE 'Stable' 
    END AS Trend 
FROM SemesterGPA 
ORDER BY StudentID, AcademicYear, Semester; 
```
<img width="576" height="143" alt="navigation LAG" src="https://github.com/user-attachments/assets/257c122b-f988-4fe5-89fb-f89b96f1db85" />

### Business Interpretation:

 LAG enables semester-to-semester GPA tracking, identifying students whose performance is 
improving or declining. Students with declining GPAs trigger academic alerts for intervention. 
Consistently improving students may qualify for scholarships or advanced placement. This early 
warning system prevents academic probation and supports student success initiatives. 

### Distribution Functions(NTILE)
```sql
WITH StudentPerformance AS ( 
    SELECT  
        s.StudentID, 
        s.FirstName || ' ' || s.LastName AS StudentName, 
        s.Department, 
        s.Program, 
        AVG(e.NumericGrade) AS CumulativeGPA, 
        SUM(c.Credits) AS TotalCredits 
    FROM Student s 
    JOIN Enrollment e ON s.StudentID = e.StudentID 
    JOIN Course c ON e.CourseID = c.CourseID 
    WHERE e.Status = 'Completed' AND e.NumericGrade IS NOT NULL 
    GROUP BY s.StudentID, s.FirstName, s.LastName, s.Department, s.Program 
) 
SELECT  
    StudentID, 
    StudentName, 
    Department, 
    ROUND(CumulativeGPA, 2) AS GPA, 
    TotalCredits, 
    NTILE(4) OVER (ORDER BY CumulativeGPA DESC) AS PerformanceQuartile, 
    CASE  
        WHEN NTILE(4) OVER (ORDER BY CumulativeGPA DESC) = 1 THEN 'Top Performers - 
Honors Track' 
        WHEN NTILE(4) OVER (ORDER BY CumulativeGPA DESC) = 2 THEN 'Above Average - 
Scholar Program' 
        WHEN NTILE(4) OVER (ORDER BY CumulativeGPA DESC) = 3 THEN 'Average - 
Standard Support' 
        ELSE 'At-Risk - Intensive Tutoring' 
    END AS SupportCategory 
FROM StudentPerformance 
ORDER BY PerformanceQuartile, GPA DESC;
```
<img width="779" height="138" alt="Disribution functions (NTILE)" src="https://github.com/user-attachments/assets/969f24ac-7318-4e8a-882d-0c4a83690a64" />

### Business Interpretation:

NTILE(4) divides students into performance quartiles, enabling targeted resource allocation. Top 
performers receive scholarship opportunities and research positions. At-risk students (bottom 
quartile) receive intensive tutoring and academic counseling.

### **STEP 7: Results Analysis** 

**Key Academic Metrics:**

- Total Students Analyzed: 8 active students - 

-Total Enrollments: 15 course registrations

- Average GPA Across All Students: 3.57 (B+ average)

-Top Performing Department: Computer Science (3.68 average GPA)
 
- Most Enrolled Course: BUS201 Financial Accounting (4 students)
 
- Highest Performing Student: Amina Uwase (STU001) with 3.72 GPA

- Course Completion Rate: 73% (11 completed out of 15 enrolled)

- Average Credits Per Student: 10.5 credits completed
 
**Department Performance:** - Computer Science: 3.68 GPA - Medicine: 4.00 GPA (limited data) - Business: 3.58 GPA - Engineering: 3.15 GPA (needs attention)

#### **2. Diagnostic Analysis - Why Did It Happen?**

**Root Causes of Performance Patterns:**

**High Computer Science Performance:**

- Quality instruction from experienced faculty (Dr. Mugisha, Mr. Maniraguha)

- Students enrolled in CS programs are highly motivated
  
- Strong prerequisite enforcement ensures prepared students

- Modern curriculum aligned with industry needs

  #### **3. Prescriptive Analysis - What Should Be Done Next?** 
 
**Immediate Actions (Next Semester):** 
 
 **Engineering Department Intervention**
    
   - Implement mandatory peer tutoring for ENG101 students
     
   - Add supplementary mathematics workshops
      
   - Review course difficulty with faculty committee
      
   - Target: Increase average grade from 3.15 to 3.40
     


### **STEP 8: References & Integrity Statement**

#### **References** 

1. Oracle Corporation. (2024). *Oracle Database SQL Language Reference: Window 
Functions*. Retrieved from 
https://docs.oracle.com/en/database/oracle/oracle-database/21/sqlrf/Analytic-Functions.html

2. Molinaro, A. (2020). *SQL Cookbook: Query Solutions and Techniques for Database 
Developers* (2nd ed.). O'Reilly Media. ISBN: 978-1492077442

3. Forta, B. (2021). *SQL in 10 Minutes a Day, Sams Teach Yourself* (5th ed.). Sams 
Publishing. ISBN: 978-0135182796

4. Date, C. J. (2019). *SQL and Relational Theory: How to Write Accurate SQL Code* (3rd ed.). 
O'Reilly Media. ISBN: 978-1491941171

#### **Academic Integrity Statement** 

"All sources were properly cited. Implementations and analysis represent original work. No AI-generated content was copied without attribution or adaptation".

### Signature: 
KAMANZI SANGWA ELSY

### Date:
February 06, 2026

### Contact

### Email
elsykamanzi@gmail.com

### GitHub:
git.com/elsy06
