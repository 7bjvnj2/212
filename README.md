-- ============ 1a. DDL ============
CREATE TABLE Student (StudentID INT PRIMARY KEY, Name VARCHAR(40) NOT NULL,
  Department VARCHAR(20), CGPA DECIMAL(4,2), Email VARCHAR(60));
CREATE TABLE Scholarship (ScholarshipID INT PRIMARY KEY, SchName VARCHAR(50) NOT NULL,
  Provider VARCHAR(40), Amount DECIMAL(10,2), MinCGPA DECIMAL(4,2));
ALTER TABLE Student ADD Phone VARCHAR(15);             -- add
ALTER TABLE Student ALTER COLUMN Phone VARCHAR(20);    -- modify
ALTER TABLE Student DROP COLUMN Phone;                 -- drop
EXEC sp_rename 'Student.Email','EmailID','COLUMN';     -- rename
TRUNCATE TABLE TempTest;                               -- all rows, keeps table
DROP TABLE TempTest;                                   -- removes table

-- ============ 1a. DML ============
INSERT INTO Student VALUES (1,'Aarav','CSE',9.10,'a@uni.edu');
SELECT * FROM Student WHERE CGPA > 8;
UPDATE Student SET CGPA = 7.9 WHERE StudentID = 3;
DELETE FROM Student WHERE StudentID = 4;

-- ============ 1b. DCL ============
CREATE LOGIN cs_login WITH PASSWORD = 'Cs@12345';
CREATE USER cs_user FOR LOGIN cs_login;
GRANT SELECT, INSERT, UPDATE ON Student TO cs_user;
DENY DELETE ON Student TO cs_user;
REVOKE UPDATE ON Student FROM cs_user;

-- ============ 1b. TCL ============
BEGIN TRANSACTION;
  UPDATE Student SET CGPA = 8.0 WHERE StudentID = 1;
  SAVE TRANSACTION sp1;
  UPDATE Student SET CGPA = 1.0 WHERE StudentID = 3;
  ROLLBACK TRANSACTION sp1;     -- undoes only the 2nd update
COMMIT TRANSACTION;             -- saves the 1st (use ROLLBACK to undo all)

-- ============ 1b(c). Integrity constraints ============
CREATE TABLE Application (
  AppID INT PRIMARY KEY,                                              -- PK
  StudentID INT NOT NULL, ScholarshipID INT NOT NULL,
  AppDate DATE DEFAULT GETDATE(),                                     -- DEFAULT
  Status VARCHAR(10) CHECK (Status IN ('Pending','Approved','Rejected')),  -- CHECK
  FOREIGN KEY (StudentID) REFERENCES Student(StudentID),              -- FK
  FOREIGN KEY (ScholarshipID) REFERENCES Scholarship(ScholarshipID));
ALTER TABLE Student ADD CONSTRAINT UQ_Email UNIQUE (Email);           -- UNIQUE
ALTER TABLE Student ADD CONSTRAINT CK_CGPA CHECK (CGPA BETWEEN 0 AND 10);
ALTER TABLE Student DROP CONSTRAINT CK_CGPA;

-- ============ 1b(d). Built-in functions ============
-- Date
SELECT GETDATE(), DATEADD(DAY,30,GETDATE()), DATEDIFF(DAY,'2026-01-01',GETDATE()),
       DATEPART(MONTH,GETDATE()), DATENAME(WEEKDAY,GETDATE()), EOMONTH(GETDATE());
-- Time
SELECT CAST(GETDATE() AS TIME), CONVERT(VARCHAR(8),GETDATE(),108), DATEPART(HOUR,GETDATE());
-- Numeric
SELECT ABS(-5), CEILING(7.2), FLOOR(7.8), ROUND(8.567,1), POWER(2,5), SQRT(144);
-- String
SELECT UPPER(Name), LOWER(Name), LEN(Name), LEFT(Name,3), SUBSTRING(Name,1,5),
       REPLACE(Email,'uni','college'), REVERSE(Name), CONCAT(Name,' ',Department) FROM Student;
-- Conversion
SELECT CAST(9.75 AS INT), CONVERT(VARCHAR(10),GETDATE(),103), TRY_CAST('abc' AS INT);

-- ============ 1c(e). Joins ============
SELECT s.Name, sc.SchName, a.Status FROM Student s                    -- INNER
JOIN Application a ON s.StudentID = a.StudentID
JOIN Scholarship sc ON a.ScholarshipID = sc.ScholarshipID;
SELECT s.Name, a.AppID FROM Student s                                 -- LEFT
LEFT JOIN Application a ON s.StudentID = a.StudentID;
SELECT sc.SchName, a.AppID FROM Application a                         -- RIGHT
RIGHT JOIN Scholarship sc ON a.ScholarshipID = sc.ScholarshipID;
SELECT s.Name, a.AppID FROM Student s                                 -- FULL
FULL OUTER JOIN Application a ON s.StudentID = a.StudentID;
SELECT s.Name, sc.SchName FROM Student s CROSS JOIN Scholarship sc;   -- CROSS
SELECT s1.Name, s2.Name FROM Student s1                               -- SELF
JOIN Student s2 ON s1.Department = s2.Department AND s1.StudentID < s2.StudentID;

-- ============ 1c(f). Sub-queries ============
-- Scalar
SELECT Name, CGPA FROM Student WHERE CGPA > (SELECT AVG(CGPA) FROM Student);
-- IN
SELECT Name FROM Student
WHERE StudentID IN (SELECT StudentID FROM Application WHERE Status = 'Approved');
-- NOT IN
SELECT SchName FROM Scholarship
WHERE ScholarshipID NOT IN (SELECT ScholarshipID FROM Application);
-- ALL
SELECT SchName, Amount FROM Scholarship
WHERE Amount > ALL (SELECT Amount FROM Scholarship WHERE Provider = 'Central Govt');
-- Correlated: CGPA above own department's average
SELECT s.Name, s.Department, s.CGPA FROM Student s
WHERE s.CGPA > (SELECT AVG(s2.CGPA) FROM Student s2 WHERE s2.Department = s.Department);
-- Correlated EXISTS
SELECT s.Name FROM Student s
WHERE EXISTS (SELECT 1 FROM Application a WHERE a.StudentID = s.StudentID);
-- Correlated NOT EXISTS (never applied)
SELECT s.Name FROM Student s
WHERE NOT EXISTS (SELECT 1 FROM Application a WHERE a.StudentID = s.StudentID);

-- ============ 2b(a). View, Synonym, Sequence ============
CREATE VIEW vw_Approved AS
SELECT s.Name, sc.SchName, sc.Amount
FROM Student s JOIN Application a ON s.StudentID = a.StudentID
JOIN Scholarship sc ON a.ScholarshipID = sc.ScholarshipID
WHERE a.Status = 'Approved';
GO
SELECT * FROM vw_Approved;
DROP VIEW vw_Approved;
GO
CREATE SYNONYM syn_Student FOR dbo.Student;
SELECT * FROM syn_Student;
DROP SYNONYM syn_Student;
GO
CREATE SEQUENCE seq_Receipt START WITH 1000 INCREMENT BY 1;
SELECT NEXT VALUE FOR seq_Receipt;      -- 1000
SELECT NEXT VALUE FOR seq_Receipt;      -- 1001
DROP SEQUENCE seq_Receipt;
GO

-- ============ 2b(b). T-SQL loops (any 2) ============
-- 1: Odd numbers 1 to N
DECLARE @N INT = 15, @i INT = 1;
WHILE @i <= @N
BEGIN
  IF @i % 2 <> 0 PRINT @i;
  SET @i = @i + 1;
END
GO
-- 2: Fibonacci up to a number
DECLARE @Limit INT = 100, @a INT = 0, @b INT = 1, @c INT, @s VARCHAR(500);
SET @s = '0 1';
SET @c = @a + @b;
WHILE @c <= @Limit
BEGIN
  SET @s = @s + ' ' + CAST(@c AS VARCHAR(10));
  SET @a = @b;
  SET @b = @c;
  SET @c = @a + @b;
END
PRINT @s;
GO

-- ============ 2c(c). Error handling ============
BEGIN TRY
  SELECT 10 / 0;
END TRY
BEGIN CATCH
  SELECT ERROR_NUMBER() AS ErrNo, ERROR_SEVERITY() AS Sev,
         ERROR_LINE() AS Ln, ERROR_MESSAGE() AS Msg;
END CATCH;
GO
-- With transaction + rollback
BEGIN TRY
  BEGIN TRANSACTION;
    INSERT INTO Application (AppID,StudentID,ScholarshipID,Status) VALUES (9,1,101,'Pending');
    INSERT INTO Application (AppID,StudentID,ScholarshipID,Status) VALUES (9,1,101,'Pending'); -- duplicate PK
  COMMIT TRANSACTION;
END TRY
BEGIN CATCH
  IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
  PRINT 'Failed: ' + ERROR_MESSAGE();
END CATCH;
GO
-- Custom errors
BEGIN TRY
  THROW 50001, 'CGPA too low for this scholarship', 1;
END TRY
BEGIN CATCH
  PRINT ERROR_MESSAGE();
END CATCH;
RAISERROR('Custom warning message', 16, 1);

-- ============ 5. JSON ============
CREATE TABLE StudentProfile (
  StudentID INT PRIMARY KEY,
  ProfileJson NVARCHAR(MAX) CHECK (ISJSON(ProfileJson) = 1));
INSERT INTO StudentProfile VALUES
(1, N'{"city":"Bengaluru","hobbies":["chess","coding"],"skills":[{"name":"SQL","level":8}]}'),
(2, N'{"city":"Mysuru","hobbies":["dance"],"skills":[]}');

SELECT ISJSON('{"a":1}');                                              -- 1 = valid
SELECT StudentID, JSON_VALUE(ProfileJson,'$.city') AS City,
       JSON_VALUE(ProfileJson,'$.hobbies[0]') AS FirstHobby FROM StudentProfile;
SELECT StudentID, JSON_QUERY(ProfileJson,'$.hobbies') AS HobbyArray FROM StudentProfile;
UPDATE StudentProfile                                                   -- modify
SET ProfileJson = JSON_MODIFY(ProfileJson,'$.city','Mangaluru') WHERE StudentID = 2;
SELECT p.StudentID, h.[value] AS Hobby                                  -- OPENJSON
FROM StudentProfile p CROSS APPLY OPENJSON(p.ProfileJson,'$.hobbies') h;
SELECT StudentID, Name, CGPA FROM Student FOR JSON PATH;                -- rows to JSON
