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

#### PDF Export Implementation

```csharp
public async Task<byte[]> GenerateTutorPerformancePdfReport()
{
    var tutors = await _messageService.GetAverageMessagesPerTutor();

    using var document = new PdfDocument();
    var page = document.AddPage();
    var gfx = XGraphics.FromPdfPage(page);
    var font = new XFont("Arial", 12);

    // Add title and headers
    gfx.DrawString("Tutor Performance Report", new XFont("Arial", 16, XFontStyle.Bold),
        XBrushes.Black, new XRect(50, 50, page.Width, 30), XStringFormats.TopLeft);

    var yPos = 100;
    foreach (var tutor in tutors)
    {
        gfx.DrawString($"Tutor: {tutor.Name} - Messages: {tutor.MessageCount}",
            font, XBrushes.Black, 50, yPos);
        yPos += 20;
    }

    using var stream = new MemoryStream();
    document.Save(stream);
    return stream.ToArray();
}
```

#### Excel Export Implementation

```csharp
public async Task<byte[]> GenerateTutorPerformanceExcelReport()
{
    var tutors = await _messageService.GetAverageMessagesPerTutor();

    using var package = new ExcelPackage();
    var worksheet = package.Workbook.Worksheets.Add("Tutor Performance");

    // Add headers
    worksheet.Cells["A1"].Value = "Tutor Name";
    worksheet.Cells["B1"].Value = "Message Count";
    worksheet.Cells["C1"].Value = "Average Messages";

    var row = 2;
    foreach (var tutor in tutors)
    {
        worksheet.Cells[row, 1].Value = tutor.Name;
        worksheet.Cells[row, 2].Value = tutor.MessageCount;
        worksheet.Cells[row, 3].Formula = $"=B{row}/COUNT(B:B)";
        row++;
    }

    worksheet.Cells[worksheet.Dimension.Address].AutoFitColumns();
    return package.GetAsByteArray();
}
```

### 4.2 Student-Tutor Assignment Check

#### PDF Export Implementation

```csharp
public async Task<byte[]> GenerateUnassignedStudentsPdfReport()
{
    var students = await _studentService.GetUnassignedStudents();

    using var document = new PdfDocument();
    var page = document.AddPage();
    var gfx = XGraphics.FromPdfPage(page);
    var font = new XFont("Arial", 12);

    gfx.DrawString("Unassigned Students Report", new XFont("Arial", 16, XFontStyle.Bold),
        XBrushes.Black, new XRect(50, 50, page.Width, 30), XStringFormats.TopLeft);

    var yPos = 100;
    foreach (var student in students)
    {
        gfx.DrawString($"Student: {student.FullName} ({student.Email})",
            font, XBrushes.Black, 50, yPos);
        yPos += 20;
    }

    using var stream = new MemoryStream();
    document.Save(stream);
    return stream.ToArray();
}
```

#### Excel Export Implementation

```csharp
public async Task<byte[]> GenerateUnassignedStudentsExcelReport()
{
    var students = await _studentService.GetUnassignedStudents();

    using var package = new ExcelPackage();
    var worksheet = package.Workbook.Worksheets.Add("Unassigned Students");

    worksheet.Cells["A1"].Value = "Student Name";
    worksheet.Cells["B1"].Value = "Email";
    worksheet.Cells["C1"].Value = "Registration Date";

    var row = 2;
    foreach (var student in students)
    {
        worksheet.Cells[row, 1].Value = student.FullName;
        worksheet.Cells[row, 2].Value = student.Email;
        worksheet.Cells[row, 3].Value = student.RegistrationDate;
        row++;
    }

    worksheet.Cells[worksheet.Dimension.Address].AutoFitColumns();
    return package.GetAsByteArray();
}
```

### 4.3 Engagement Tracking with Date Filters

#### PDF Export Implementation

```csharp
public async Task<byte[]> GenerateInactiveStudentsPdfReport(int days)
{
    var students = await _studentService.GetInactiveStudents(days);

    using var document = new PdfDocument();
    var page = document.AddPage();
    var gfx = XGraphics.FromPdfPage(page);
    var font = new XFont("Arial", 12);

    gfx.DrawString($"Inactive Students Report (Last {days} days)",
        new XFont("Arial", 16, XFontStyle.Bold),
        XBrushes.Black, new XRect(50, 50, page.Width, 30), XStringFormats.TopLeft);

    var yPos = 100;
    foreach (var student in students)
    {
        gfx.DrawString($"Student: {student.FullName} - Last Login: {student.LastLoginTime}",
            font, XBrushes.Black, 50, yPos);
        yPos += 20;
    }

    using var stream = new MemoryStream();
    document.Save(stream);
    return stream.ToArray();
}
```

#### Excel Export Implementation

```csharp
public async Task<byte[]> GenerateInactiveStudentsExcelReport(int days)
{
    var students = await _studentService.GetInactiveStudents(days);

    using var package = new ExcelPackage();
    var worksheet = package.Workbook.Worksheets.Add("Inactive Students");

    worksheet.Cells["A1"].Value = "Student Name";
    worksheet.Cells["B1"].Value = "Email";
    worksheet.Cells["C1"].Value = "Last Login";
    worksheet.Cells["D1"].Value = "Days Inactive";

    var row = 2;
    foreach (var student in students)
    {
        worksheet.Cells[row, 1].Value = student.FullName;
        worksheet.Cells[row, 2].Value = student.Email;
        worksheet.Cells[row, 3].Value = student.LastLoginTime;
        worksheet.Cells[row, 4].Formula = $"=DATEDIF(C{row},TODAY(),\"d\")";
        row++;
    }

    worksheet.Cells[worksheet.Dimension.Address].AutoFitColumns();
    return package.GetAsByteArray();
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

```typescript
// src/components/dashboard/TutorPerformance.tsx
import { Card, Statistic, Spin } from "antd";
import { useQuery } from "react-query";
import { fetchTutorPerformance } from "@/lib/api";

interface TutorPerformanceData {
  averageMessages: number;
}

export const TutorPerformance = () => {
  const { data, isLoading, error } = useQuery<TutorPerformanceData>(
    "tutorPerformance",
    fetchTutorPerformance,
    { refetchInterval: 900000 } // 15 minutes refresh
  );

  if (error) {
    return (
      <Card>
        <div>Error loading tutor performance data</div>
      </Card>
    );
  }

  return (
    <Card title="Tutor Performance">
      <Spin spinning={isLoading}>
        <Statistic
          title="Average Messages per Tutor"
          value={data?.averageMessages || 0}
          precision={2}
        />
      </Spin>
    </Card>
  );
};
```

- **Student Status:**
  - **Display:** Display a list or table of students who are not assigned to a tutor.
  - **UI Element:** Use a table component (e.g., from a UI library like Material UI, Ant Design, or a custom implementation). Include student names, and potentially other relevant information (e.g., registration date).
  - **Data Refresh:** Implement a refresh mechanism (e.g., on a button click or periodically) to fetch the data from the `api/students/unassigned` endpoint.

```typescript
// src/components/students/UnassignedStudents.tsx
import { Table, Button, message } from "antd";
import { useQuery } from "react-query";
import { fetchUnassignedStudents } from "@/lib/api";

interface Student {
  id: string;
  name: string;
  registrationDate: string;
}

export const UnassignedStudents = () => {
  const { data, isLoading, error, refetch } = useQuery<Student[]>(
    "unassignedStudents",
    fetchUnassignedStudents
  );

  const columns = [
    {
      title: "Name",
      dataIndex: "name",
      key: "name",
    },
    {
      title: "Registration Date",
      dataIndex: "registrationDate",
      key: "registrationDate",
      render: (date: string) => new Date(date).toLocaleDateString(),
    },
  ];

  if (error) {
    message.error("Failed to load unassigned students");
  }

  return (
    <div>
      <div style={{ marginBottom: 16 }}>
        <Button onClick={() => refetch()}>Refresh Data</Button>
      </div>
      <Table
        columns={columns}
        dataSource={data}
        loading={isLoading}
        rowKey="id"
      />
    </div>
  );
};
```

- **Student Engagement:**
  - **Display:** Display two lists or tables: one for students with no interactions in the last 7 days and another for students with no interactions in the last 28 days.
  - **UI Element:** Use table components. Include student names and potentially other relevant information.
  - **Data Refresh:** Implement a refresh mechanism (e.g., on a button click or periodically) to fetch the data from the `api/students/inactive` endpoint, passing the appropriate `days` parameter (7 or 28).

```typescript
// src/components/students/StudentEngagement.tsx
import { Tabs, Table, Button, message } from "antd";
import { useQuery } from "react-query";
import { fetchInactiveStudents } from "@/lib/api";

interface InactiveStudent {
  id: string;
  name: string;
  lastInteraction: string;
}

export const StudentEngagement = () => {
  const sevenDaysQuery = useQuery<InactiveStudent[]>(
    ["inactiveStudents", 7],
    () => fetchInactiveStudents(7)
  );

  const twentyEightDaysQuery = useQuery<InactiveStudent[]>(
    ["inactiveStudents", 28],
    () => fetchInactiveStudents(28)
  );

  const columns = [
    {
      title: "Name",
      dataIndex: "name",
      key: "name",
    },
    {
      title: "Last Interaction",
      dataIndex: "lastInteraction",
      key: "lastInteraction",
      render: (date: string) => new Date(date).toLocaleDateString(),
    },
  ];

  const handleError = (error: Error) => {
    message.error("Failed to load inactive students data");
  };

  return (
    <Tabs defaultActiveKey="7days">
      <Tabs.TabPane tab="7 Days Inactive" key="7days">
        <div style={{ marginBottom: 16 }}>
          <Button onClick={() => sevenDaysQuery.refetch()}>Refresh</Button>
        </div>
        <Table
          columns={columns}
          dataSource={sevenDaysQuery.data}
          loading={sevenDaysQuery.isLoading}
          rowKey="id"
        />
      </Tabs.TabPane>
      <Tabs.TabPane tab="28 Days Inactive" key="28days">
        <div style={{ marginBottom: 16 }}>
          <Button onClick={() => twentyEightDaysQuery.refetch()}>
            Refresh
          </Button>
        </div>
        <Table
          columns={columns}
          dataSource={twentyEightDaysQuery.data}
          loading={twentyEightDaysQuery.isLoading}
          rowKey="id"
        />
      </Tabs.TabPane>
    </Tabs>
  );
};
```

- **Export to Excel/CSV:**

  - **Display:** Provide buttons to export the data to Excel or CSV format.
  - **UI Element:** Use buttons or icons.
  - **Data Export:** Implement API endpoints for exporting the data to Excel or CSV format. Use appropriate libraries (e.g., ExcelDataReader, EPPlus) to generate the Excel or CSV file.

```tsx
// src/components/ExportButtons.tsx
import { Button } from "antd";
import { FileExcelOutlined } from "@ant-design/icons";

interface ExportButtonsProps {
  days: number;
  onExport: (format: "excel" | "csv") => void;
}

export const ExportButtons: React.FC<ExportButtonsProps> = ({
  days,
  onExport,
}) => {
  return (
    <div className="flex gap-2">
      <Button
        type="primary"
        icon={<FileExcelOutlined />}
        onClick={() => onExport("excel")}
      >
        Export to Excel
      </Button>
    </div>
  );
};

// Usage in parent component:
const ParentComponent = () => {
  const handleExport = async (format: "excel" | "csv") => {
    try {
      const response = await fetch(
        `/api/reports/export?days=${days}&format=${format}`,
        {
          method: "GET",
          headers: {
            Authorization: `Bearer ${token}`,
          },
        }
      );

      if (!response.ok) {
        throw new Error("Export failed");
      }

      const blob = await response.blob();
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = `report_${days}_days.${format}`;
      document.body.appendChild(a);
      a.click();
      window.URL.revokeObjectURL(url);
      document.body.removeChild(a);
    } catch (error) {
      message.error("Failed to export data");
      console.error("Export error:", error);
    }
  };

  return <ExportButtons days={days} onExport={handleExport} />;
};
```

- **API Endpoints:**

```typescript
// src/lib/api.ts
const API_BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL;

export const fetchTutorPerformance = async () => {
  const response = await fetch(
    `${API_BASE_URL}/api/messages/tutor-performance`
  );
  if (!response.ok) throw new Error("Failed to fetch tutor performance");
  return response.json();
};

export const fetchUnassignedStudents = async () => {
  const response = await fetch(`${API_BASE_URL}/api/students/unassigned`);
  if (!response.ok) throw new Error("Failed to fetch unassigned students");
  return response.json();
};

export const fetchInactiveStudents = async (days: number) => {
  const response = await fetch(
    `${API_BASE_URL}/api/students/inactive?days=${days}`
  );
  if (!response.ok) throw new Error("Failed to fetch inactive students");
  return response.json();
};
```

- **General Considerations:**
  - **Error Handling:** Implement proper error handling to display informative messages to the user if the API calls fail.
  - **Loading Indicators:** Use loading indicators to provide feedback to the user while data is being fetched.
  - **Accessibility:** Ensure the UI is accessible to users with disabilities (e.g., using ARIA attributes, providing sufficient color contrast).
  - **Responsiveness:** Ensure the UI is responsive and adapts to different screen sizes.
  - **Authentication/Authorization:** Ensure that the frontend correctly handles authentication and authorization to protect the data.

## 7. Conclusion

## Implementation Conclusion

Our data extraction strategy employs a backend-focused approach using the .NET framework, leveraging two specialized libraries for report generation:

**PDF Generation with PdfSharp:**

- _Why Chosen:_ Provides MIT-licensed, lightweight PDF creation capabilities ideal for basic report requirements
- _Advantages:_
  - Zero licensing costs
  - Low memory footprint
  - Straightforward API for core PDF operations
- _Limitations:_
  - Lacks advanced features like digital signatures
  - Limited documentation compared to commercial alternatives

**Excel Reporting with ClosedXML:**

- _Why Selected:_ Offers MIT-licensed Excel manipulation with balance between features and complexity
- _Implementation:_ Database-driven report generation through Entity Framework Core queries
- _Strengths:_
  - Commercial-use friendly licensing
- _Constraints:_
  - Slower performance with datasets >100k rows
  - Basic charting capabilities

**Technical Recommendations:**

1. Use asynchronous processing for reports exceeding 50 pages/10k rows
2. Implement caching for frequently requested reports
3. Monitor memory usage through ASP.NET Core diagnostics

This backend-centric approach ensures secure, scalable data processing while maintaining frontend performance through optimized API responses and downloading the said Excel or PDF file.
