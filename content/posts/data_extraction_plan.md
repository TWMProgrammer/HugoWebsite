# Data Extraction Plan for ETutoring Application

## 1. Introduction

This document outlines a comprehensive technical investigation into extracting specific data points from the backend of the "ETutoring" application. The focus is on feasibility, implementation strategies, and frontend presentation.

## 2. Data Extraction Requirements

The following data points need to be extracted:

- **Message Activity:** Determine the total number of messages sent within the last 7 days.
- **Tutor Performance:** Calculate the average number of messages per tutor.
- **Student Status:** Identify a list of students who are not currently assigned to a tutor.
- **Student Engagement:** Generate lists of students who have not had any interactions (e.g., messages, assignments) within the last 7 and 28 days.

## 3. Implementation Strategy

### 3.1. Message Activity

- **API Endpoint:** Create a new endpoint `GET api/messages/activity`.
- **Database Query:** Query the database to retrieve messages sent within the last 7 days.
- **Implementation:**
  1.  Add a new method to the `IMessageService` interface:
      ```csharp
      Task<int> GetTotalMessagesLast7Days();
      ```
  2.  Implement the `GetTotalMessagesLast7Days` method in the `MessageService` class:
      ```csharp
      public async Task<int> GetTotalMessagesLast7Days()
      {
          return await _context.Messages.CountAsync(m => m.SentDate >= DateTime.UtcNow.AddDays(-7));
      }
      ```
  3.  Add a new endpoint to the `MessageController`:
      ```csharp
      [HttpGet("activity/total-messages")]
      public async Task<IActionResult> GetTotalMessagesLast7Days()
      {
          try
          {
              var totalMessages = await _messageService.GetTotalMessagesLast7Days();
              return Ok(new { TotalMessages = totalMessages });
          }
          catch (Exception ex)
          {
              Console.WriteLine($"Error in GetTotalMessagesLast7Days: {ex.Message}");
              return StatusCode(500, new { Message = "Internal Server Error", Error = ex.Message });
          }
      }
      ```
- **Data Processing:** Calculate the total number of messages.
- **Technology Stack:** .NET, Entity Framework Core, SQL Server or another database.
- **Pros:**
  - Simple and efficient query.
  - Easy to implement.
- **Cons:**
  - Requires a new API endpoint.

### 3.2. Tutor Performance

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
- **Cons:**
  - Requires a more complex database query involving joins and aggregations.
  - Requires a new API endpoint.

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
          // Assuming StudentTutorManagement table exists and has StudentId
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
- **Cons:**
  - Requires a new API endpoint.
  - The query's performance depends on the database indexes and the size of the tables.

### 3.4. Student Engagement

- **API Endpoint:** Create a new endpoint `GET api/students/inactive` in the `StudentController`.
- **Database Query:**
  - Retrieve a list of all students.
  - Check for interactions (messages, assignments, etc.) within the last 7 and 28 days.
  - Filter the list to include students with no interactions within the specified timeframes.
- **Implementation:**
  1.  Add a new method to the `IStudentService` interface:
      ```csharp
      Task<List<StudentResponse>> GetInactiveStudents(int days);
      ```
  2.  Implement the `GetInactiveStudents` method in the `StudentService` class:
      ```csharp
      public async Task<List<StudentResponse>> GetInactiveStudents(int days)
      {
          var inactiveStudents = await _context.Students
              .Where(s => !_context.Messages.Any(m => m.StudentId == s.Id && m.SentDate >= DateTime.UtcNow.AddDays(-days)) &&
                          !_context.Assignments.Any(a => a.StudentId == s.Id && a.SubmissionDate >= DateTime.UtcNow.AddDays(-days))) // Add other interaction types
              .Select(s => new StudentResponse { /* Map student properties */ })
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
              if (days != 7 && days != 28)
              {
                  return BadRequest("Days must be 7 or 28.");
              }
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
- **Data Processing:** Filter the student list.
- **Technology Stack:** .NET, Entity Framework Core, SQL Server or another database.
- **Pros:**
  - Identifies students who may need attention.
- **Cons:**
  - The most complex query, potentially involving multiple joins and date comparisons.
  - Requires a new API endpoint.
  - Performance is critical; database indexes are essential.

## 4. Technology Stack

- **.NET:** The backend is built using .NET.
- **Entity Framework Core:** The ORM for database interaction.
- **SQL Server (or similar):** The database.

## 5. Technology Evaluation

- **Pros:**
  - Mature and well-supported technologies.
  - Good performance and scalability.
  - Strong security features.
  - Large community and extensive documentation.
- **Cons:**
  - Requires .NET development expertise.
  - Can be more complex than some other options.

## 6. Frontend Integration

The frontend integration will focus on displaying the extracted data in a clear and user-friendly manner. The specific implementation will depend on the existing frontend framework (Next.js, based on the file structure).

- **Message Activity:**
  - **Display:** Show the total number of messages sent within the last 7 days.
  - **UI Element:** Use a simple number display, a card, or a dashboard widget.
  - **Data Refresh:** Implement a periodic refresh (e.g., every 5 minutes) using `setInterval` or a similar mechanism to fetch the data from the `api/messages/activity` endpoint. Consider using a loading indicator while the data is being fetched.
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
