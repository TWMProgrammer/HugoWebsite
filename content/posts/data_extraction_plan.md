# Data Extraction Plan for ETutoring Application

## 1. Introduction

This document outlines a comprehensive technical investigation into extracting specific data points from the backend of the "ETutoring" application. The focus is on feasibility, implementation strategies, and frontend presentation.

## 2. Data Extraction Requirements

The following data points need to be extracted:

- **Tutor Performance:** Calculate the average number of messages per tutor.
- **Student Status:** Identify a list of students who are not currently assigned to a tutor.
- **Student Engagement:** Generate lists of students who have not had any interactions (e.g., messages, assignments) within the last 7 and 28 days.
- **PDF Files:** Efficiently extract relevant PDF files from the backend.
- **Excel Database Reports:** Generate and export database reports in Excel format.

## 3. Implementation Strategy

### 3.1. Tutor Performance

- **API Endpoint:** Create a new endpoint `GET api/messages/tutor-performance`.
- **Database Query:**

  - Join the `Message` and `ApplicationUser` tables.
  - Group messages by tutor (user ID).
  - Count the number of messages per tutor.
  - Calculate the average number of messages across all tutors.

- **Implementation:**

  1.  Add a new method to the `IMessageService` interface:
      ```csharp
      Task<double> GetAverageMessagesPerTutor();
      ```
  2.  Implement the `GetAverageMessagesPerTutor` method in the `MessageService` class:

      ```csharp
      public async Task<double> GetAverageMessagesPerTutor()
      {
          var tutorMessageCounts = await _context.Messages
              .Include(m => m.Sender) // Assuming Sender is the tutor
              .GroupBy(m => m.SenderId)
              .Select(g => new { TutorId = g.Key, MessageCount = g.Count() })
              .ToListAsync();

          if (tutorMessageCounts.Count == 0)
          {
              return 0;
          }

          return tutorMessageCounts.Average(t => t.MessageCount);
      }
      ```

  3.  Add a new endpoint to the `MessageController`:
      ```csharp
      [HttpGet("tutor-performance")]
      public async Task<IActionResult> GetAverageMessagesPerTutor()
      {
          try
          {
              var averageMessages = await _messageService.GetAverageMessagesPerTutor();
              return Ok(new { AverageMessages = averageMessages });
          }
          catch (Exception ex)
          {
              Console.WriteLine($"Error in GetAverageMessagesPerTutor: {ex.Message}");
              return StatusCode(500, new { Message = "Internal Server Error", Error = ex.Message });
          }
      }
      ```

- **Data Processing:** Perform the average calculation.
- **Technology Stack:** .NET, Entity Framework Core, SQL Server or another database.
- **Pros:**
  - Provides valuable insights into tutor performance.
  - Relatively easy to implement with Entity Framework Core.
- **Cons:**
  - Requires a more complex database query involving joins and aggregations, potentially impacting performance, especially with large datasets.
  - Requires a new API endpoint.
  - Scalability: Performance can degrade with a large number of tutors and messages. Requires careful indexing and query optimization.
  - Security: Requires proper authentication and authorization to prevent unauthorized access to sensitive performance data.

### 3.3. Student Status

- **API Endpoint:** Create a new endpoint `GET api/students/unassigned` in the `StudentController`.
- **Database Query:**

  - Retrieve a list of all students.
  - Check if each student is assigned to a tutor (using `StudentTutorManagement` entity).
  - Filter the list to include only students without a tutor assignment.

- **Implementation:**

  1.  Add a new method to the `IStudentService` interface:
      ```csharp
      Task<List<StudentResponse>> GetUnassignedStudents();
      ```
  2.  Implement the `GetUnassignedStudents` method in the `StudentService` class:
      ```csharp
      public async Task<List<StudentResponse>> GetUnassignedStudents()
      {
          var unassignedStudents = await _context.Students
              .Where(s => !_context.StudentTutorManagements.Any(stm => stm.StudentId == s.Id))
              .Select(s => new StudentResponse { /* Map student properties */ })
              .ToListAsync();
          return unassignedStudents;
      }
      ```
  3.  Add a new endpoint to the `StudentController`:
      ```csharp
      [HttpGet("unassigned")]
      public async Task<IActionResult> GetUnassignedStudents()
      {
          try
          {
              var unassignedStudents = await _studentService.GetUnassignedStudents();
              return Ok(unassignedStudents);
          }
          catch (Exception ex)
          {
              Console.WriteLine($"Error in GetUnassignedStudents: {ex.Message}");
              return StatusCode(500, new { Message = "Internal Server Error", Error = ex.Message });
          }
      }
      ```

- **Data Processing:** Filter the student list.
- **Technology Stack:** .NET, Entity Framework Core, SQL Server or another database.
- **Pros:**
  - Provides a clear view of unassigned students.
  - Relatively straightforward to implement.
- **Cons:**
  - Requires a new API endpoint.
  - The query's performance depends on the database indexes and the size of the tables. Performance can degrade significantly with a large number of students and a poorly indexed `StudentTutorManagement` table.
  - Scalability: Performance can be a concern with a large number of students. Requires careful indexing and query optimization.
  - Security: Requires proper authentication and authorization to prevent unauthorized access to student data.

### 3.4. Student Engagement

- **API Endpoint:** Create a new endpoint `GET api/students/inactive` in the `StudentController`.
- **Database Query:**

  - Retrieve a list of all students.
  - Check the LastLoginTime field to identify students who haven't logged in within the specified timeframe.
  - Filter the list to include students with no recent login activity.

- **Implementation:**

  1.  Add a new method to the `IStudentService` interface:
      ```csharp
      Task<List<StudentResponse>> GetInactiveStudents(int days);
      ```
  2.  Implement the `GetInactiveStudents` method in the `StudentService` class:
      ```csharp
      public async Task<List<StudentResponse>> GetInactiveStudents(int days)
      {
          var cutoffDate = DateTime.UtcNow.AddDays(-days);
          var inactiveStudents = await _context.Users
              .Where(u => u.UserType == "Student")
              .Where(u => u.LastLoginTime == null || u.LastLoginTime < cutoffDate)
              .Select(u => new StudentResponse
              {
                  Id = u.Id,
                  FullName = u.FullName,
                  Email = u.Email,
                  LastLoginTime = u.LastLoginTime
              })
              .ToListAsync();
          return inactiveStudents;
      }
      ```
  3.  Add a new endpoint to the `StudentController`:
      ```csharp
      [HttpGet("inactive")]
      public async Task<IActionResult> GetInactiveStudents([FromQuery] int days = 7)
      {
          try
          {
              var inactiveStudents = await _studentService.GetInactiveStudents(days);
              return Ok(inactiveStudents);
          }
          catch (Exception ex)
          {
              Console.WriteLine($"Error in GetInactiveStudents: {ex.Message}");
              return StatusCode(500, new { Message = "Internal Server Error", Error = ex.Message });
          }
      }
      ```

- **Data Processing:** Filter the student list based on LastLoginTime.
- **Technology Stack:** .NET, Entity Framework Core, SQL Server or another database.
- **Pros:**
  - Simple and efficient query using LastLoginTime field.
  - No complex joins or multiple date comparisons needed.
  - Better performance with a single field comparison.
  - Easy to maintain and understand.
- **Cons:**
  - Requires a new API endpoint.
  - Security: Requires proper authentication and authorization to prevent unauthorized access to student data.

### 3.4.1. 7-Day Inactivity

- **API Endpoint:** `GET /api/students/inactive?days=7`

### 3.4.2. 28-Day Inactivity

- **API Endpoint:** `GET /api/students/inactive?days=28`

### 3.4.3. Implementation Plan for 'lastlogin' field

- **Database Schema Changes:**
  - Add a `lastlogin` field of type `DateTime` (or `datetime2` in SQL Server) to the `Students` table.
  - Consider adding a default value (e.g., `NULL` or a specific date) for existing student records.
- **Data Population Strategies:**
  - **On Login Updates:**
    - Implement a mechanism to update the `lastlogin` field whenever a student successfully logs into the application. This can be done within the login process in the backend.
    - Ensure that the update is performed atomically (e.g., within a transaction) to maintain data consistency.
  - **Background Jobs (for initial population or data correction):**
    - If there's a need to populate the `lastlogin` field for existing student records, implement a background job (e.g., using a queue or scheduled task).
    - The background job can iterate through all student records and set the `lastlogin` field to the current date and time, or to the date of their registration if available.
    - Consider batching the updates to improve performance.
- **Performance Considerations:**
  - **Indexing:** Create an index on the `lastlogin` field to optimize queries for inactive students.
  - **Query Optimization:** Ensure that the queries for inactive students are optimized (e.g., using appropriate indexes, avoiding full table scans).
  - **Data Partitioning (if applicable):** If the student table is very large, consider data partitioning to improve query performance.
  - **Caching:** Cache the results of the inactive student queries to reduce database load.
- **Data Consistency:**
  - Ensure that the `lastlogin` field is updated accurately and consistently.
  - Implement proper error handling to prevent data loss or corruption.
  - Consider using database transactions to ensure atomicity of updates.
- **API Endpoint Design:**
  - The existing `/api/students/inactive` endpoint will be used.
  - The database query will use the `lastlogin` field to determine inactivity.
- **Pros:**
  - Provides a reliable way to track student login activity.
  - Enables efficient querying for inactive students.
- **Cons:**
  - Requires database schema changes.
  - Requires modifications to the login process.
  - Adds a small overhead to the login process (updating the `lastlogin` field).
  - Data consistency is crucial; requires careful implementation to avoid data corruption.

### 3.5. Data Export Requirements

#### 3.5.1. PDF Files

- **Methodology:**

  - Generate PDF reports from data and templates using optimized database queries
  - Implement PDF file storage with filesystem-based caching
  - Provide API endpoints for PDF generation/download with JWT authentication
  - Use EF Core compiled queries for performance
  - Implement retry policies for database access
  - Add validation for template parameters

- **Query Optimization:**
  - Use AsNoTracking() for read-only operations
  - Implement Redis caching for frequently accessed templates
  - Use pagination for large datasets
  - Create covering indexes on message timestamps and tutor IDs
- **Technology Stack:**
  - .NET
  - PDF Generation Libraries:
    - **iText 7:**
      - AGPL license (requires commercial license for closed-source applications)
      - Comprehensive PDF manipulation capabilities
      - High performance and reliability
      - Advanced features like digital signatures and encryption
      - Pros:
        - Extensive feature set
        - Excellent performance
        - Strong security features
        - Regular updates and support
      - Cons:
        - Expensive commercial licensing
        - Complex API for beginners
        - Large library size
    - **PdfSharp:**
      - MIT License (free for commercial use)
      - Lightweight and easy to use
      - Basic PDF creation and manipulation
      - .NET Core support via PdfSharpCore
      - Pros:
        - Free for commercial use
        - Simple and intuitive API
        - Small library footprint
        - Good for basic PDF operations
      - Cons:
        - Limited advanced features
        - Less active development
        - Limited documentation
    - **Recommended Choice: PdfSharp**
      - PdfSharp is recommended for this project because:
        1. Free and open-source (MIT license)
        2. Sufficient features for report generation
        3. Lightweight and easy to integrate
        4. Good performance for basic PDF operations
        5. Lower learning curve for developers
  - File system access libraries (System.IO)
  - Cloud storage SDK (if applicable)
- **Implementation Considerations:**
  - Use template-based approach for consistent PDF formatting
  - Implement caching for frequently generated reports
  - Use asynchronous operations for large PDF generation
  - Implement proper error handling and logging
- **Security Considerations:**
  - Implement proper authentication and authorization
  - Validate and sanitize input parameters
  - Use secure file handling practices
  - Implement rate limiting for PDF generation
- **Frontend Integration Plan:**
  - **API Endpoint:** `GET /api/files/pdf/{filename}` (or similar, with appropriate security).
  - **UI Element:** Provide a button or link to download the PDF file.
  - **Data Formatting:** The API will return the PDF file as a stream.
  - **Error Handling:** Display appropriate error messages if the file is not found or cannot be downloaded.
- **API Endpoint Design:**
  - **Endpoint:** `GET /api/files/pdf/{filename}`
  - **Request:**
    - `filename`: (string, required) The name of the PDF file.
  - **Response:**
    - `200 OK`: Returns the PDF file as a stream with `Content-Type: application/pdf`.
    - `404 Not Found`: If the file is not found.
    - `500 Internal Server Error`: For other errors.

#### 3.5.2. Excel Database Reports

- **Methodology:**
  - Use a reporting framework to generate Excel reports from database queries.
  - Define report templates or dynamic report generation logic.
  - Provide an API endpoint to download the generated Excel reports.
- **Technology Stack:**
  - .NET
  - Excel Generation Libraries:
    - **EPPlus:**
      - Commercial license required for commercial use (LicenseContext.NonCommercial for free use)
      - Extensive features including charts, formulas, and styling
      - High performance and memory efficiency
      - Active development and community support
      - Built-in template support
      - Pros:
        - Rich feature set and excellent documentation
        - High performance with large datasets
        - Strong community support and regular updates
        - Built-in template engine
      - Cons:
        - Commercial licensing cost
        - Learning curve for advanced features
        - Memory usage can be high with very large datasets
    - **ClosedXML:**
      - MIT License (free for commercial use)
      - Simpler API compared to EPPlus
      - Good performance for basic operations
      - No built-in template support
      - Pros:
        - Free for commercial use
        - Simple and intuitive API
        - Good documentation
        - Active community
      - Cons:
        - Fewer advanced features compared to EPPlus
        - Limited template support
        - Can be slower for complex operations
    - **Recommended Choice: ClosedXML**
      - ClosedXML is recommended for this project because:
        1. Free for commercial use under MIT license
        2. Simple and intuitive API for basic report generation
        3. Good documentation and active community support
        4. Sufficient features for standard Excel report generation
        5. No licensing costs or restrictions
  - Database connection (Entity Framework Core)
- **Implementation Considerations:**
  - Use template-based approach for consistent report formatting
  - Implement caching for frequently generated reports
  - Consider asynchronous report generation for large datasets
  - Implement proper error handling and logging
- **Security Considerations:**
  - Implement proper authentication and authorization
  - Validate and sanitize input parameters
  - Use secure file handling practices
  - Implement rate limiting for report generation
- **Frontend Integration Plan:**
  - **API Endpoint:** `GET /api/reports/excel/{reportType}` (e.g., `tutorPerformance`, `inactiveStudents`).
  - **UI Element:** Provide a button or link to download the Excel report.
  - **Data Formatting:** The API will return the Excel file as a stream with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.
  - **Error Handling:** Display appropriate error messages if the report generation fails.
- **API Endpoint Design:**
  - **Endpoint:** `GET /api/reports/excel/{reportType}`
  - **Request:**
    - `reportType`: (string, required) The type of report to generate (e.g., `tutorPerformance`, `inactiveStudents`).
  - **Response:**
    - `200 OK`: Returns the Excel file as a stream with `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`.
    - `400 Bad Request`: If the `reportType` is invalid.
    - `500 Internal Server Error`: For other errors.

## 4. Example Implementations

### 4.1 Average Messages per Tutor Calculation

```csharp
public async Task<decimal> CalculateAverageMessagesPerTutor()
{
    var result = await _context.Tutors
        .Select(t => new {
            t.Id,
            MessageCount = _context.Messages.Count(m => m.TutorId == t.Id)
        })
        .GroupBy(t => 1)
        .Select(g => g.Average(t => t.MessageCount))
        .FirstOrDefaultAsync();

    return result;
}
```

### 4.2 Student-Tutor Assignment Check

```csharp
public async Task<List<UnassignedStudentDto>> GetUnassignedStudents()
{
    return await _context.Students
        .Where(s => s.TutorId == null && s.IsActive)
        .Select(s => new UnassignedStudentDto {
            Id = s.Id,
            Name = s.FullName,
            RegisteredAt = s.CreatedAt
        })
        .AsNoTracking()
        .ToListAsync();
}
```

### 4.3 Engagement Tracking with Date Filters

```csharp
public async Task<EngagementStats> GetEngagementStats(DateTime startDate, DateTime endDate)
{
    return await _context.Students
        .Where(s => s.LastLogin >= startDate && s.LastLogin <= endDate)
        .GroupBy(s => 1)
        .Select(g => new EngagementStats {
            ActiveUsers = g.Count(),
            AvgLoginFrequency = g.Average(s => s.LoginCount),
            TotalSessions = g.Sum(s => s.SessionCount)
        })
        .FirstOrDefaultAsync() ?? new EngagementStats();
}
```

## 5. Technology Stack

- **.NET:** The backend is built using .NET.
- **Entity Framework Core:** The ORM for database interaction.
- **SQL Server (PostgreSQL):** The database.

## 6. Technology Evaluation

- **Pros:**
  - Mature and well-supported technologies.
  - Good performance and scalability.
  - Strong security features.
  - Large community and extensive documentation.
- **Cons:**
  - Requires .NET development expertise.
  - Can be more complex than some other options.
  - Potential vendor lock-in with Microsoft technologies.

## 7. Frontend Integration

The frontend integration will focus on displaying the extracted data in a clear and user-friendly manner. The specific implementation will depend on the existing frontend framework (Next.js, based on the file structure).

- **Tutor Performance:**
  - **Display:** Show the average number of messages per tutor.
  - **UI Element:** Use a number display, a card, or a dashboard widget.
  - **Data Refresh:** Implement a periodic refresh (e.g., every 15 minutes) using `setInterval` or a similar mechanism to fetch the data from the `api/messages/tutor-performance` endpoint.
- **Student Status:**
  - **Display:** Display a list or table of students who are not assigned to a tutor.
  - **UI Element:** Use a table component (e.g., from a UI library like Material UI, Ant Design, or a custom implementation). Include student names, and potentially other relevant information (e.g., registration date).
  - **Data Refresh:** Implement a refresh mechanism (e.g., on a button click or periodically) to fetch the data from the `api/students/unassigned` endpoint.
- **Student Engagement:**
  - **Display:** Display two lists or tables: one for students with no interactions in the last 7 days and another for students with no interactions in the last 28 days.
  - **UI Element:** Use table components. Include student names and potentially other relevant information.
  - **Data Refresh:** Implement a refresh mechanism (e.g., on a button click or periodically) to fetch the data from the `api/students/inactive` endpoint, passing the appropriate `days` parameter (7 or 28).
- **General Considerations:**
  - **Error Handling:** Implement proper error handling to display informative messages to the user if the API calls fail.
  - **Loading Indicators:** Use loading indicators to provide feedback to the user while data is being fetched.
  - **Accessibility:** Ensure the UI is accessible to users with disabilities (e.g., using ARIA attributes, providing sufficient color contrast).
  - **Responsiveness:** Ensure the UI is responsive and adapts to different screen sizes.
  - **Authentication/Authorization:** Ensure that the frontend correctly handles authentication and authorization to protect the data.

## 7. Conclusion

This plan provides a detailed strategy for extracting the required data points from the ETutoring application. The proposed implementation leverages the existing .NET backend and utilizes appropriate database queries and API endpoints. The frontend integration will focus on clear data visualization and user-friendly presentation, with considerations for data refresh, error handling, and accessibility.
