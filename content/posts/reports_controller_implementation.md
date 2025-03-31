# Report on Report Category Logic in ETutoring API Backend

## 1. Overview

This report details the logic behind the report generation functionalities in the ETutoring API backend application. It covers the flow from the `ReportController` to the services (`MessageService`, `StudentService`), explains the Data Transfer Objects (DTOs) involved, and describes the use of external libraries such as `ClosedXML` and `PdfSharp` for generating Excel and PDF reports.

The report category in the API backend provides endpoints for moderators and admins to retrieve various reports related to tutor performance and student activity. These reports are generated based on data retrieved from the database and can be returned as JSON responses, PDF documents, or Excel spreadsheets.

## 2. Controller Logic (`ReportController.cs`)

The `ReportController` is located at `ETutoring-BE/ETutoring.API/Controllers/Report/ReportController.cs`. It is an API controller responsible for handling HTTP requests related to report generation.

**Route and Authorization:**

- The controller is decorated with `[Route("api/reports")]`, meaning all endpoints are prefixed with `/api/reports`.
- `[ApiController]` attribute indicates that it's an API controller with features like automatic model validation.
- `[Authorize(Roles = "Moderator,Admin")]` attribute ensures that only users with "Moderator" or "Admin" roles can access these endpoints, enforcing security.

**Dependencies:**

The controller constructor injects two services:

- `IMessageService`: Used for reports related to tutor performance based on messaging activity.
- `IStudentService`: Used for reports related to student activity, inactivity, and unassigned students.

**Endpoints:**

The `ReportController` defines the following endpoints:

- **`POST api/reports/tutor-performance`**: `GetAverageMessagesPerTutor()`

  - Retrieves average messages per tutor.
  - Delegates the logic to `_messageService.GetAverageMessagesPerTutorAsync()`.
  - Returns an `OkObjectResult` (200 OK) with the response data or a `StatusCodeResult` (500 Internal Server Error) in case of an exception.

- **`POST api/reports/tutor-performance/pdf`**: `GetTutorPerformancePdfReport()`

  - Generates a PDF report of tutor performance.
  - Delegates to `_messageService.GenerateTutorPerformancePdfReportAsync()`.
  - Returns a `FileResult` with the PDF content (`application/pdf` MIME type).

- **`POST api/reports/tutor-performance/excel`**: `GetTutorPerformanceExcelReport()`

  - Generates an Excel report of tutor performance.
  - Delegates to `_messageService.GenerateTutorPerformanceExcelReportAsync()`.
  - Returns a `FileResult` with the Excel content (`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` MIME type).

- **`POST api/reports/students/unassigned`**: `GetUnassignedStudents()`

  - Retrieves a list of students who are not assigned to any tutor.
  - Delegates to `_studentService.GetUnassignedStudentsAsync()`.
  - Returns an `OkObjectResult` or `StatusCodeResult` similar to `GetAverageMessagesPerTutor()`.

- **`POST api/reports/students/unassigned/pdf`**: `GetUnassignedStudentsPdfReport()`

  - Generates a PDF report of unassigned students.
  - Delegates to `_studentService.GenerateUnassignedStudentsPdfReportAsync()`.
  - Returns a `FileResult` with PDF content.

- **`POST api/reports/students/unassigned/excel`**: `GetUnassignedStudentsExcelReport()`

  - Generates an Excel report of unassigned students.
  - Delegates to `_studentService.GenerateUnassignedStudentsExcelReportAsync()`.
  - Returns a `FileResult` with Excel content.

- **`POST api/reports/students/inactive`**: `GetInactiveStudents([FromBody] InactiveStudentsRequest request)`

  - Retrieves a list of inactive students based on the number of days provided in the `InactiveStudentsRequest`.
  - Delegates to `_studentService.GetInactiveStudentsAsync(days)`.
  - Defaults to 7 days if `Days` is not provided in the request body.
  - Returns an `OkObjectResult` or `StatusCodeResult`.

- **`POST api/reports/students/inactive/pdf`**: `GetInactiveStudentsPdfReport([FromBody] InactiveStudentsRequest request)`

  - Generates a PDF report of inactive students.
  - Delegates to `_studentService.GenerateInactiveStudentsPdfReportAsync(days)`.
  - Defaults to 7 days for inactivity period.
  - Returns a `FileResult` with PDF content.

- **`POST api/reports/students/inactive/excel`**: `GetInactiveStudentsExcelReport([FromBody] InactiveStudentsRequest request)`

  - Generates an Excel report of inactive students.
  - Delegates to `_studentService.GenerateInactiveStudentsExcelReportAsync(days)`.
  - Defaults to 7 days for inactivity period.
  - Returns a `FileResult` with PDF content.

- **`POST api/reports/students/no-interaction`**: `GetStudentsWithoutInteraction([FromBody] InactiveStudentsRequest request)`

  - Retrieves students without interaction in the last specified days.
  - Delegates to `_studentService.GetStudentsWithoutInteractionAsync(days)`.
  - Uses `InactiveStudentsRequest` to receive the number of days.
  - Defaults to 7 days if `Days` is not provided.
  - Returns an `OkObjectResult` or `StatusCodeResult`.

- **`POST api/reports/students/no-interaction/pdf`**: `GetStudentsWithoutInteractionPdfReport([FromBody] InactiveStudentsRequest request)`

  - Generates a PDF report of students without interaction.
  - Delegates to `_studentService.GenerateStudentsWithoutInteractionPdfReportAsync(days)`.
  - Defaults to 7 days for the period.
  - Returns a `FileResult` with PDF content.

- **`POST api/reports/students/no-interaction/excel`**: `GetStudentsWithoutInteractionExcelReport([FromBody] InactiveStudentsRequest request)`

  - Generates an Excel report of students without interaction.
  - Delegates to `_studentService.GenerateStudentsWithoutInteractionExcelReportAsync(days)`.
  - Defaults to 7 days for the period.
  - Returns a `FileResult` with Excel content.

- **`POST api/reports/students/unconfirmed-email`**: `GetUnconfirmedEmailStudents()`

  - Retrieves a list of students with unconfirmed emails.
  - Delegates to `_studentService.GetUnconfirmedEmailStudentsAsync()`.
  - Returns an `OkObjectResult` or `StatusCodeResult`.

- **`POST api/reports/students/unconfirmed-email/pdf`**: `GetUnconfirmedEmailStudentsPdfReport()`

  - Generates a PDF report of students with unconfirmed emails.
  - Delegates to `_studentService.GenerateUnconfirmedEmailStudentsPdfReportAsync()`.
  - Returns a `FileResult` with PDF content.

- **`POST api/reports/students/unconfirmed-email/excel`**: `GetUnconfirmedEmailStudentsExcelReport()`
  - Generates an Excel report of students with unconfirmed emails.
  - Delegates to `_studentService.GenerateUnconfirmedEmailStudentsExcelReportAsync()`.
  - Returns a `FileResult` with Excel content.

## 3. Service Logic (`MessageService.cs`, `StudentService.cs`)

The service logic resides in the `ETutoring-BE/ETutoring.DataAccess/Services/` directory, specifically in `MessageService.cs` (within `Messages/`) and `StudentService.cs` (within `Students/`).

### `MessageService.cs`

This service (`MessageService`) handles report generation related to messages and tutor performance.

- **`GetAverageMessagesPerTutorAsync()`**:

  - Retrieves all users with the "Tutor" role from the database.
  - Counts the number of messages sent by each tutor.
  - Calculates the average number of messages per tutor.
  - Returns an `ApiResponse<AverageMessagesResponse>` containing the average and a list of `TutorPerformanceResponse` objects.
  - Uses Entity Framework Core to query the `UserRoles`, `Roles`, `Users`, and `Messages` tables in the database context (`ApplicationDbContext`).

- **`GenerateTutorPerformancePdfReportAsync()`**:

  - Calls `GetAverageMessagesPerTutorAsync()` to get the data.
  - Uses `PdfSharp` library to create a PDF document.
  - Adds a title, average message count, and a table listing tutor names and their message counts to the PDF.
  - Returns the PDF document as a byte array.

- **`GenerateTutorPerformanceExcelReportAsync()`**:
  - Calls `GetAverageMessagesPerTutorAsync()` to get the data.
  - Uses `ClosedXML` library to create an Excel workbook.
  - Adds a title, average message count, and a table listing tutor names and their message counts to the Excel sheet.
  - Calculates the percentage of each tutor's message count compared to the average.
  - Returns the Excel workbook as a byte array.

### `StudentService.cs`

This service (`StudentService`) handles report generation related to student data.

- **`GetUnassignedStudentsAsync()`**:

  - Retrieves all users with the "Student" role who are not currently assigned to any tutor.
  - Returns an `ApiResponse<List<UnassignedStudentResponse>>` containing a list of `UnassignedStudentResponse` objects.
  - Uses Entity Framework Core to query the `UserRoles`, `Roles`, `StudentTutorManagements`, and `Users` tables.

- **`GenerateUnassignedStudentsPdfReportAsync()`**:

  - Calls `GetUnassignedStudentsAsync()` to get the data.
  - Uses `PdfSharp` library to create a PDF document.
  - Adds a title and a table listing unassigned student names and emails to the PDF.
  - Returns the PDF document as a byte array.

- **`GenerateUnassignedStudentsExcelReportAsync()`**:

  - Calls `GetUnassignedStudentsAsync()` to get the data.
  - Uses `ClosedXML` library to create an Excel workbook.
  - Adds a title and a table listing unassigned student names and emails to the Excel sheet.
  - Returns the Excel workbook as a byte array.

- **`GetInactiveStudentsAsync(int days)`**:

  - Retrieves all users with the "Student" role who have not logged in within the specified number of days.
  - Returns an `ApiResponse<List<InactiveStudentResponse>>` containing a list of `InactiveStudentResponse` objects.
  - Uses Entity Framework Core to query the `UserRoles`, `Roles`, and `Users` tables.

- **`GenerateInactiveStudentsPdfReportAsync(int days)`**:

  - Calls `GetInactiveStudentsAsync(int days)` to get the data.
  - Uses `PdfSharp` library to create a PDF document.
  - Adds a title, a table listing inactive student names, emails, last login time, and days inactive to the PDF.
  - Returns the PDF document as a byte array.

- **`GenerateInactiveStudentsExcelReportAsync(int days)`**:

  - Calls `GetInactiveStudentsAsync(int days)` to get the data.
  - Uses `ClosedXML` library to create an Excel workbook.
  - Adds a title, a table listing inactive student names, emails, last login time, and days inactive to the Excel sheet.
  - Returns the Excel workbook as a byte array.

- **`GetStudentsWithoutInteractionAsync(int days)`**:
  _ Retrieves students without interaction in the last specified days.
  _ Returns an `ApiResponse<List<StudentWithoutInteractionResponse>>` containing a list of `StudentWithoutInteractionResponse` objects. \* Uses Entity Framework Core to query the `UserRoles`, `Roles`, `Users`, and `Messages` tables.

- **`GenerateStudentsWithoutInteractionPdfReportAsync(int days)`**:
  _ Calls `GetStudentsWithoutInteractionAsync(int days)` to get the data.
  _ Uses `PdfSharp` library to create a PDF document.
  _ Adds a title, a table listing students without interaction, their emails, and last interaction time to the PDF.
  _ Returns the PDF document as a byte array.

- **`GenerateStudentsWithoutInteractionExcelReportAsync(int days)`**:
  _ Calls `GetStudentsWithoutInteractionAsync(int days)` to get the data.
  _ Uses `ClosedXML` library to create an Excel workbook.
  _ Adds a title, a table listing students without interaction, their emails, and last interaction time to the Excel sheet.
  _ Returns the Excel workbook as a byte array.

- **`GetUnconfirmedEmailStudentsAsync()`**:
  _ Retrieves a list of students with unconfirmed emails.
  _ Returns an `ApiResponse<List<UnconfirmedEmailStudentResponse>>` containing a list of `UnconfirmedEmailStudentResponse` objects. \* Uses Entity Framework Core to query the `Users` and `UserRoles` tables.

- **`GenerateUnconfirmedEmailStudentsPdfReportAsync()`**:
  _ Calls `GetUnconfirmedEmailStudentsAsync()` to get the data.
  _ Uses `PdfSharp` library to create a PDF document.
  _ Adds a title, a table listing students with unconfirmed emails and their emails to the PDF.
  _ Returns the PDF document as a byte array.

- **`GenerateUnconfirmedEmailStudentsExcelReportAsync()`**:
  _ Calls `GetUnconfirmedEmailStudentsAsync()` to get the data.
  _ Uses `ClosedXML` library to create an Excel workbook.
  _ Adds a title, a table listing students with unconfirmed emails and their emails to the Excel sheet.
  _ Returns the Excel workbook as a byte array.

## 4. Data Transfer Objects (DTOs)

The following DTOs are used in the report generation process:

- **`InactiveStudentsRequest`**:

  - Located in: `ETutoring-BE/ETutoring.Business/Dtos/Request/Moderator/InactiveStudentsRequest.cs`
  - Purpose: Represents the request body for the `GetInactiveStudents` endpoint.
  - Properties:
    - `Days` (int?): Optional. Specifies the number of days to consider for inactivity. Defaults to 7 if not provided.

- **`TutorPerformanceResponse`**:

  - Located in: `ETutoring-BE/ETutoring.Business/Dtos/Response/Message/TutorPerformanceResponse.cs`
  - Purpose: Represents the performance of a single tutor.
  - Properties:
    - `TutorId` (Guid): The unique identifier of the tutor.
    - `TutorName` (string): The full name of the tutor.
    - `MessageCount` (int): The number of messages sent by the tutor.

- **`AverageMessagesResponse`**:

  - Located in: `ETutoring-BE/ETutoring.Business/Dtos/Response/Message/TutorPerformanceResponse.cs`
  - Purpose: Represents the average message count across all tutors and a list of individual tutor performances.
  - Properties:
    - `AverageMessages` (double): The average number of messages per tutor.
    - `TutorPerformances` (List<TutorPerformanceResponse>): A list of `TutorPerformanceResponse` objects, each representing a tutor's performance.

- **`UnassignedStudentResponse`**:

  - Located in: `ETutoring-BE/ETutoring.Business/Dtos/Response/Students/UnassignedStudentResponse.cs`
  - Purpose: Represents a student who is not assigned to any tutor.
  - Properties:
    - `Id` (Guid): The unique identifier of the student.
    - `FullName` (string): The full name of the student.
    - `Email` (string): The email address of the student.

- **`InactiveStudentResponse`**:

  - Located in: `ETutoring-BE/ETutoring.Business/Dtos/Response/Students/InactiveStudentResponse.cs`
  - Purpose: Represents a student who has not logged in for a specified period.
  - Properties:
    - `Id` (Guid): The unique identifier of the student.
    - `FullName` (string): The full name of the student.
    - `Email` (string): The email address of the student.
    - `LastLoginTime` (DateTime?): The last login time of the student (nullable).
    - `DaysInactive` (int): The number of days since the student last logged in.

- **`StudentWithoutInteractionResponse`**:
  _ Located in: `ETutoring-BE/ETutoring.Business/Dtos/Response/Students/StudentWithoutInteractionResponse.cs`
  _ Purpose: Represents a student who has not interacted with any tutor in a specified period.
  _ Properties:
  _ `Id` (string): The unique identifier of the student.
  _ `FullName` (string): The full name of the student.
  _ `Email` (string): The email address of the student. \* `LastInteractionTime` (DateTime?): The last interaction time of the student (nullable).

- **`UnconfirmedEmailStudentResponse`**:
  _ Located in: `ETutoring-BE/ETutoring.Business/Dtos/Response/Students/UnconfirmedEmailStudentResponse.cs`
  _ Purpose: Represents a student with an unconfirmed email address.
  _ Properties:
  _ `Id` (string): The unique identifier of the student.
  _ `FullName` (string): The full name of the student.
  _ `Email` (string): The email address of the student.

## 5. Library Explanations

The report generation process utilizes the following external libraries:

- **`ClosedXML`**:

  - Purpose: Used for generating Excel (XLSX) files.
  - Usage: The service methods create an `XLWorkbook` object, add a worksheet, populate it with data, and then save the workbook to a memory stream. The stream is then returned as a byte array.
  - Example:

    ```csharp
    using var workbook = new XLWorkbook();
    var worksheet = workbook.Worksheets.Add("Tutor Performance");

    // Add data to worksheet
    worksheet.Cell(1, 1).Value = "Tutor Performance Report";

    using var stream = new MemoryStream();
    workbook.SaveAs(stream);
    return stream.ToArray();
    ```

    - `using var workbook = new XLWorkbook();`: This line creates a new `XLWorkbook` object, which represents the Excel workbook. The `using` statement ensures that the workbook is properly disposed of after it's used.
    - `var worksheet = workbook.Worksheets.Add("Tutor Performance");`: This line adds a new worksheet to the workbook with the name "Tutor Performance".
    - `worksheet.Cell(1, 1).Value = "Tutor Performance Report";`: This line sets the value of the cell at row 1, column 1 to "Tutor Performance Report".
    - `using var stream = new MemoryStream();`: This line creates a new `MemoryStream` object, which is used to store the Excel workbook in memory.
    - `workbook.SaveAs(stream);`: This line saves the Excel workbook to the memory stream.
    - `return stream.ToArray();`: This line returns the memory stream as a byte array.

- **`PdfSharp`**:

  - Purpose: Used for generating PDF documents.
  - Usage: The service methods create a `PdfDocument` object, add a page, create a graphics object, and then draw text and shapes onto the page. The document is then saved to a memory stream and returned as a byte array.
  - Example:

    ```csharp
    using var document = new PdfDocument();
    var page = document.AddPage();
    var gfx = XGraphics.FromPdfPage(page);
    var font = new XFont("Arial", 12);

    // Draw text on the page
    gfx.DrawString("Hello, PDF!", font, XBrushes.Black, new XPoint(100, 100));

    using var stream = new MemoryStream();
    document.Save(stream);
    return stream.ToArray();
    ```

    - `using var document = new PdfDocument();`: This line creates a new `PdfDocument` object, which represents the PDF document. The `using` statement ensures that the document is properly disposed of after it's used.
    - `var page = document.AddPage();`: This line adds a new page to the PDF document.
    - `var gfx = XGraphics.FromPdfPage(page);`: This line creates an `XGraphics` object from the page, which is used to draw text and shapes onto the page.
    - `var font = new XFont("Arial", 12);`: This line creates a new `XFont` object with the Arial font and a size of 12.
    - `gfx.DrawString("Hello, PDF!", font, XBrushes.Black, new XPoint(100, 100));`: This line draws the text "Hello, PDF!" onto the page.
    - `using var stream = new MemoryStream();`: This line creates a new `MemoryStream` object, which is used to store the PDF document in memory.
    - `document.Save(stream);`: This line saves the PDF document to the memory stream.
    - `return stream.ToArray();`: This line returns the memory stream as a byte array.

## 6. Code Examples

### 6.1. Normal Data Retrieval (`GetAverageMessagesPerTutorAsync`)

This example demonstrates a standard data retrieval endpoint that returns a JSON response.

```csharp
public async Task<ApiResponse<AverageMessagesResponse>> GetAverageMessagesPerTutorAsync()
{
    try
    {
        // Get all users with Tutor role
        var tutorIds = await _context.UserRoles
            .Join(_context.Roles,
                ur => ur.RoleId,
                r => r.Id,
                (ur, r) => new { ur.UserId, RoleName = r.Name })
            .Where(x => x.RoleName == "Tutor")
            .Select(x => x.UserId.ToString())
            .ToListAsync();

        // Get message counts for each tutor
        var tutorMessageCounts = await _context.Messages
            .Where(m => tutorIds.Contains(m.SenderId))
            .GroupBy(m => m.SenderId)
            .Select(g => new
            {
                TutorId = g.Key,
                MessageCount = g.Count()
            })
            .ToListAsync();

        // Get tutor names
        var tutorIdGuids = tutorMessageCounts.Select(t => Guid.Parse(t.TutorId)).ToList();
        var tutors = await _context.Users
            .Where(u => tutorIdGuids.Contains(u.Id))
            .Select(u => new { u.Id, u.FullName })
            .ToListAsync();

        // Calculate average
        double averageMessages = 0;
        if (tutorMessageCounts.Count > 0)
        {
            averageMessages = tutorMessageCounts.Average(t => t.MessageCount);
        }

        // Create response
        var tutorPerformances = tutorMessageCounts.Select(t => new TutorPerformanceResponse
        {
            TutorId = Guid.Parse(t.TutorId),
            TutorName = tutors.FirstOrDefault(u => u.Id == Guid.Parse(t.TutorId))?.FullName ?? "Unknown",
            MessageCount = t.MessageCount
        }).ToList();

        var response = new AverageMessagesResponse
        {
            AverageMessages = averageMessages,
            TutorPerformances = tutorPerformances
        };

        return ApiResponse<AverageMessagesResponse>.SuccessResponse(response);
    }
    catch (Exception ex)
    {
        return ApiResponse<AverageMessagesResponse>.FailureResponse($"An error occurred: {ex.Message}");
    }
}
```

Explanation:

- `public async Task<ApiResponse<AverageMessagesResponse>> GetAverageMessagesPerTutorAsync()`: This line defines the method signature. It's an asynchronous method that returns an `ApiResponse` containing an `AverageMessagesResponse` object.
- `try`: This block starts a try-catch block to handle potential exceptions.
- `var tutorIds = await _context.UserRoles ... ToListAsync();`: This code retrieves the IDs of all users with the "Tutor" role.
  - `_context.UserRoles`: Accesses the `UserRoles` table in the database.
  - `.Join(_context.Roles, ...)`: Joins the `UserRoles` table with the `Roles` table to filter by role name.
  - `.Where(x => x.RoleName == "Tutor")`: Filters the joined data to include only users with the "Tutor" role.
  - `.Select(x => x.UserId.ToString())`: Selects the `UserId` and converts it to a string.
  - `.ToListAsync()`: Asynchronously executes the query and returns the results as a list.
- `var tutorMessageCounts = await _context.Messages ... ToListAsync();`: This code retrieves the message counts for each tutor.
  - `_context.Messages`: Accesses the `Messages` table in the database.
  - `.Where(m => tutorIds.Contains(m.SenderId))`: Filters the messages to include only those sent by tutors.
  - `.GroupBy(m => m.SenderId)`: Groups the messages by the sender ID (tutor ID).
  - `.Select(g => new { TutorId = g.Key, MessageCount = g.Count() })`: Selects the tutor ID and the count of messages for each group.
  - `.ToListAsync()`: Asynchronously executes the query and returns the results as a list.
- `var tutorIdGuids = tutorMessageCounts.Select(t => Guid.Parse(t.TutorId)).ToList();`: This line converts the tutor IDs from strings to GUIDs.
- `var tutors = await _context.Users ... ToListAsync();`: This code retrieves the full names of the tutors.
  - `_context.Users`: Accesses the `Users` table in the database.
  - `.Where(u => tutorIdGuids.Contains(u.Id))`: Filters the users to include only those whose IDs are in the `tutorIdGuids` list.
  - `.Select(u => new { u.Id, u.FullName })`: Selects the user ID and full name.
  - `.ToListAsync()`: Asynchronously executes the query and returns the results as a list.
- `double averageMessages = 0;`: Initializes a variable to store the average message count.
- `if (tutorMessageCounts.Count > 0) { ... }`: This conditional statement checks if there are any tutors.
  - `averageMessages = tutorMessageCounts.Average(t => t.MessageCount);`: Calculates the average message count across all tutors.
- `var tutorPerformances = tutorMessageCounts.Select(t => new TutorPerformanceResponse ... ToList();`: This code creates a list of `TutorPerformanceResponse` objects.
  - `.Select(t => new TutorPerformanceResponse { ... })`: Selects the tutor ID, tutor name, and message count for each tutor.
  - `.ToList()`: Converts the selected data into a list of `TutorPerformanceResponse` objects.
- `var response = new AverageMessagesResponse { ... };`: This code creates an `AverageMessagesResponse` object containing the average message count and the list of tutor performances.
- `return ApiResponse<AverageMessagesResponse>.SuccessResponse(response);`: This line returns a success response with the `AverageMessagesResponse` object.
- `catch (Exception ex)`: This block catches any exceptions that occur during the process.
- `return ApiResponse<AverageMessagesResponse>.FailureResponse($"An error occurred: {ex.Message}");`: This line returns a failure response with an error message.

### 6.2. PDF Generation (`GenerateUnassignedStudentsPdfReportAsync`)

This example demonstrates how to generate a PDF report using the `PdfSharp` library.

```csharp
public async Task<byte[]> GenerateUnassignedStudentsPdfReportAsync()
{
    var unassignedStudentsResponse = await GetUnassignedStudentsAsync();
    if (!unassignedStudentsResponse.Success)
    {
        throw new Exception(unassignedStudentsResponse.Message);
    }

    var unassignedStudents = unassignedStudentsResponse.Data;

    using var document = new PdfDocument();
    var page = document.AddPage();
    var gfx = XGraphics.FromPdfPage(page);
    var font = new XFont("Arial", 12);
    var titleFont = new XFont("Arial", 16, XFontStyleEx.Bold);
    var headerFont = new XFont("Arial", 14, XFontStyleEx.Bold);

    gfx.DrawString("Unassigned Students Report", titleFont,
        XBrushes.Black, new XRect(50, 50, page.Width, 30), XStringFormats.TopLeft);

    gfx.DrawString("Student Name", headerFont, XBrushes.Black, 50, 100);
    gfx.DrawString("Email", headerFont, XBrushes.Black, 250, 100);
    // Removed Registration Date header

    var yPos = 130;
    foreach (var student in unassignedStudents)
    {
        gfx.DrawString(student.FullName, font, XBrushes.Black, 50, yPos);
        gfx.DrawString(student.Email, font, XBrushes.Black, 250, yPos);
        // Removed Registration Date data
        yPos += 20;

        if (yPos > page.Height - 50)
        {
            page = document.AddPage();
            gfx = XGraphics.FromPdfPage(page);
            yPos = 50;
        }
    }

    gfx.DrawString($"Generated on: {DateTime.Now}", font, XBrushes.Black, 50, page.Height - 50);

    using var stream = new MemoryStream();
    document.Save(stream);
    return stream.ToArray();
}
```

Explanation:

- `public async Task<byte[]> GenerateUnassignedStudentsPdfReportAsync()`: This line defines the method signature. It's an asynchronous method that returns a byte array representing the PDF document.
- `var unassignedStudentsResponse = await GetUnassignedStudentsAsync();`: This line calls the `GetUnassignedStudentsAsync()` method to retrieve the list of unassigned students.
- `if (!unassignedStudentsResponse.Success) { ... }`: This conditional statement checks if the response from `GetUnassignedStudentsAsync()` was successful.
  - `throw new Exception(unassignedStudentsResponse.Message);`: If the response was not successful, this line throws an exception with the error message.
- `var unassignedStudents = unassignedStudentsResponse.Data;`: This line retrieves the list of unassigned students from the response data.
- `using var document = new PdfDocument();`: This line creates a new `PdfDocument` object using the `using` statement, which ensures that the object is disposed of properly after it's used.
- `var page = document.AddPage();`: This line adds a new page to the PDF document.
- `var gfx = XGraphics.FromPdfPage(page);`: This line creates an `XGraphics` object from the page, which is used to draw text and shapes onto the page.
- `var font = new XFont("Arial", 12);`: This line creates a new `XFont` object with the Arial font and a size of 12.
- `var titleFont = new XFont("Arial", 16, XFontStyleEx.Bold);`: This line creates a new `XFont` object with the Arial font, a size of 16, and bold style.
- `var headerFont = new XFont("Arial", 14, XFontStyleEx.Bold);`: This line creates a new `XFont` object with the Arial font, a size of 14, and bold style.
- `gfx.DrawString("Unassigned Students Report", titleFont, ...);`: This line draws the title of the report onto the page.
  - `gfx.DrawString()`: Draws a string onto the page.
  - `"Unassigned Students Report"`: The string to draw.
  - `titleFont`: The font to use for the string.
  - `XBrushes.Black`: The color to use for the string.
  - `new XRect(50, 50, page.Width, 30)`: The rectangle to draw the string in.
  - `XStringFormats.TopLeft`: The string format to use.
- `gfx.DrawString("Student Name", headerFont, ...);`: This line draws the header for the student name column onto the page.
- `gfx.DrawString("Email", headerFont, ...);`: This line draws the header for the email column onto the page.
- `var yPos = 130;`: This line initializes a variable to store the Y position for the next row of data.
- `foreach (var student in unassignedStudents) { ... }`: This loop iterates over the list of unassigned students.
  - `gfx.DrawString(student.FullName, font, ...);`: This line draws the full name of the student onto the page.
  - `gfx.DrawString(student.Email, font, ...);`: This line draws the email address of the student onto the page.
  - `yPos += 20;`: This line increments the Y position for the next row of data.
  - `if (yPos > page.Height - 50) { ... }`: This conditional statement checks if the Y position is greater than the height of the page minus 50.
    - `page = document.AddPage();`: If the Y position is greater than the height of the page minus 50, this line adds a new page to the PDF document.
    - `gfx = XGraphics.FromPdfPage(page);`: This line creates a new `XGraphics` object from the new page.
    - `yPos = 50;`: This line resets the Y position to 50.
- `gfx.DrawString($"Generated on: {DateTime.Now}", font, ...);`: This line draws the generation date onto the page.
- `using var stream = new MemoryStream();`: This line creates a new `MemoryStream` object using the `using` statement.
- `document.Save(stream);`: This line saves the PDF document to the memory stream.
- `return stream.ToArray();`: This line returns the memory stream as a byte array.

### 6.3. Excel Generation (`GenerateInactiveStudentsExcelReportAsync`)

This example demonstrates how to generate an Excel report using the `ClosedXML` library.

```csharp
public async Task<byte[]> GenerateInactiveStudentsExcelReportAsync(int days)
{
    var inactiveStudentsResponse = await GetInactiveStudentsAsync(days);
    if (!inactiveStudentsResponse.Success)
    {
        throw new Exception(inactiveStudentsResponse.Message);
    }

    var inactiveStudents = inactiveStudentsResponse.Data;

    using var workbook = new XLWorkbook();
    var worksheet = workbook.Worksheets.Add("Inactive Students");

    worksheet.Cell(1, 1).Value = $"Inactive Students Report (Last {days} Days)";
    worksheet.Cell(1, 1).Style.Font.Bold = true;
    worksheet.Cell(1, 1).Style.Font.FontSize = 16;

    worksheet.Cell(3, 1).Value = "Student Name";
    worksheet.Cell(3, 2).Value = "Email";
    worksheet.Cell(3, 3).Value = "Last Login";
    worksheet.Cell(3, 4).Value = "Days Inactive";
    worksheet.Range(3, 1, 3, 4).Style.Font.Bold = true;

    var row = 4;
    foreach (var student in inactiveStudents)
    {
        worksheet.Cell(row, 1).Value = student.FullName;
        worksheet.Cell(row, 2).Value = student.Email;
        worksheet.Cell(row, 3).Value = student.LastLoginTime;
        if (student.LastLoginTime.HasValue)
        {
            worksheet.Cell(row, 3).Style.DateFormat.Format = "yyyy-MM-dd";
        }
        worksheet.Cell(row, 4).Value = student.DaysInactive;
        row++;
    }

    worksheet.Cell(row + 2, 1).Value = $"Generated on: {DateTime.Now}";

    worksheet.Columns().AdjustToContents();

    using var stream = new MemoryStream();
    workbook.SaveAs(stream);
    return stream.ToArray();
}
```

Explanation:

- `public async Task<byte[]> GenerateInactiveStudentsExcelReportAsync(int days)`: This line defines the method signature. It's an asynchronous method that takes an integer `days` as input and returns a byte array representing the Excel report.
- `var inactiveStudentsResponse = await GetInactiveStudentsAsync(days);`: This line calls the `GetInactiveStudentsAsync(days)` method to retrieve the list of inactive students.
- `if (!inactiveStudentsResponse.Success) { ... }`: This conditional statement checks if the response from `GetInactiveStudentsAsync(days)` was successful.
  - `throw new Exception(inactiveStudentsResponse.Message);`: If the response was not successful, this line throws an exception with the error message.
- `var inactiveStudents = inactiveStudentsResponse.Data;`: This line retrieves the list of inactive students from the response data.
- `using var workbook = new XLWorkbook();`: This line creates a new `XLWorkbook` object using the `using` statement, which ensures that the object is disposed of properly after it's used.
- `var worksheet = workbook.Worksheets.Add("Inactive Students");`: This line adds a new worksheet to the Excel workbook with the name "Inactive Students".
- `worksheet.Cell(1, 1).Value = $"Inactive Students Report (Last {days} Days)";`: This line sets the value of the cell at row 1, column 1 to the title of the report, which includes the number of days.
- `worksheet.Cell(1, 1).Style.Font.Bold = true;`: This line sets the font of the cell at row 1, column 1 to bold.
- `worksheet.Cell(1, 1).Style.Font.FontSize = 16;`: This line sets the font size of the cell at row 1, column 1 to 16.
- `worksheet.Cell(3, 1).Value = "Student Name";`: This line sets the value of the cell at row 3, column 1 to "Student Name".
- `worksheet.Cell(3, 2).Value = "Email";`: This line sets the value of the cell at row 3, column 2 to "Email".
- `worksheet.Cell(3, 3).Value = "Last Login";`: This line sets the value of the cell at row 3, column 3 to "Last Login".
- `worksheet.Cell(3, 4).Value = "Days Inactive";`: This line sets the value of the cell at row 3, column 4 to "Days Inactive".
- `worksheet.Range(3, 1, 3, 4).Style.Font.Bold = true;`: This line sets the font of the cells in the range from row 3, column 1 to row 3, column 4 to bold.
- `var row = 4;`: This line initializes a variable to store the current row number.
- `foreach (var student in inactiveStudents) { ... }`: This loop iterates over the list of inactive students.
  - `worksheet.Cell(row, 1).Value = student.FullName;`: This line sets the value of the cell at the current row and column 1 to the full name of the student.
  - `worksheet.Cell(row, 2).Value = student.Email;`: This line sets the value of the cell at the current row and column 2 to the email address of the student.
  - `worksheet.Cell(row, 3).Value = student.LastLoginTime;`: This line sets the value of the cell at the current row and column 3 to the last login time of the student.
  - `if (student.LastLoginTime.HasValue) { ... }`: This conditional statement checks if the student has a last login time.
    - `worksheet.Cell(row, 3).Style.DateFormat.Format = "yyyy-MM-dd";`: If the student has a last login time, this line sets the date format of the cell at the current row and column 3 to "yyyy-MM-dd".
  - `worksheet.Cell(row, 4).Value = student.DaysInactive;`: This line sets the value of the cell at the current row and column 4 to the number of days the student has been inactive.
  - `row++;`: This line increments the current row number.
- `worksheet.Cell(row + 2, 1).Value = $"Generated on: {DateTime.Now}";`: This line sets the value of the cell at row + 2, column 1 to the generation date.
- `worksheet.Columns().AdjustToContents();`: This line adjusts the width of the columns to fit the content.
- `using var stream = new MemoryStream();`: This line creates a new `MemoryStream` object using the `using` statement.
- `workbook.SaveAs(stream);`: This line saves the Excel workbook to the memory stream.
- `return stream.ToArray();`: This line returns the memory stream as a byte array.
