# Apache POI Excel Export Service Guide

## Complete Excel Export Architecture

### Maven Dependencies

```xml
<dependencies>
    <!-- Apache POI for Excel -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.5</version>
    </dependency>

    <!-- For streaming large files -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml-lite</artifactId>
        <version>5.2.5</version>
    </dependency>

    <!-- XML Beans (required by POI) -->
    <dependency>
        <groupId>org.apache.xmlbeans</groupId>
        <artifactId>xmlbeans</artifactId>
        <version>5.1.1</version>
    </dependency>
</dependencies>
```

## Generic Excel Export Service

### Base Export Service

```java
@Service
@Slf4j
public class ExcelExportService {

    private static final String CONTENT_TYPE = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";

    public <T> byte[] exportToExcel(
        List<T> data,
        ExcelConfig<T> config
    ) {
        try (Workbook workbook = new XSSFWorkbook();
             ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {

            Sheet sheet = workbook.createSheet(config.getSheetName());

            // Create styles
            CellStyle headerStyle = createHeaderStyle(workbook);
            CellStyle dateStyle = createDateStyle(workbook);
            CellStyle currencyStyle = createCurrencyStyle(workbook);
            CellStyle numberStyle = createNumberStyle(workbook);

            // Create header row
            createHeaderRow(sheet, config.getColumns(), headerStyle);

            // Create data rows
            createDataRows(sheet, data, config.getColumns(), dateStyle, currencyStyle, numberStyle);

            // Auto-size columns
            autoSizeColumns(sheet, config.getColumns().size());

            // Apply filters
            sheet.setAutoFilter(new CellRangeAddress(0, data.size(), 0, config.getColumns().size() - 1));

            workbook.write(outputStream);
            return outputStream.toByteArray();

        } catch (IOException e) {
            log.error("Error exporting to Excel", e);
            throw new ExcelExportException("Failed to export data to Excel", e);
        }
    }

    private CellStyle createHeaderStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();

        // Font
        Font font = workbook.createFont();
        font.setBold(true);
        font.setFontHeightInPoints((short) 12);
        font.setColor(IndexedColors.WHITE.getIndex());
        style.setFont(font);

        // Background
        style.setFillForegroundColor(IndexedColors.BLUE.getIndex());
        style.setFillPattern(FillPatternType.SOLID_FOREGROUND);

        // Borders
        style.setBorderBottom(BorderStyle.THIN);
        style.setBorderTop(BorderStyle.THIN);
        style.setBorderRight(BorderStyle.THIN);
        style.setBorderLeft(BorderStyle.THIN);

        // Alignment
        style.setAlignment(HorizontalAlignment.CENTER);
        style.setVerticalAlignment(VerticalAlignment.CENTER);

        return style;
    }

    private CellStyle createDateStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        CreationHelper createHelper = workbook.getCreationHelper();
        style.setDataFormat(createHelper.createDataFormat().getFormat("dd/mm/yyyy"));
        return style;
    }

    private CellStyle createCurrencyStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        CreationHelper createHelper = workbook.getCreationHelper();
        style.setDataFormat(createHelper.createDataFormat().getFormat("€#,##0.00"));
        return style;
    }

    private CellStyle createNumberStyle(Workbook workbook) {
        CellStyle style = workbook.createCellStyle();
        CreationHelper createHelper = workbook.getCreationHelper();
        style.setDataFormat(createHelper.createDataFormat().getFormat("#,##0.00"));
        return style;
    }

    private <T> void createHeaderRow(Sheet sheet, List<ColumnConfig<T>> columns, CellStyle headerStyle) {
        Row headerRow = sheet.createRow(0);
        headerRow.setHeightInPoints(25);

        for (int i = 0; i < columns.size(); i++) {
            Cell cell = headerRow.createCell(i);
            cell.setCellValue(columns.get(i).getHeader());
            cell.setCellStyle(headerStyle);
        }
    }

    private <T> void createDataRows(
        Sheet sheet,
        List<T> data,
        List<ColumnConfig<T>> columns,
        CellStyle dateStyle,
        CellStyle currencyStyle,
        CellStyle numberStyle
    ) {
        int rowNum = 1;
        for (T item : data) {
            Row row = sheet.createRow(rowNum++);
            for (int i = 0; i < columns.size(); i++) {
                Cell cell = row.createCell(i);
                ColumnConfig<T> column = columns.get(i);
                Object value = column.getValueExtractor().apply(item);

                if (value == null) {
                    cell.setBlank();
                } else if (value instanceof String) {
                    cell.setCellValue((String) value);
                } else if (value instanceof Number) {
                    cell.setCellValue(((Number) value).doubleValue());
                    if (column.getType() == ColumnType.CURRENCY) {
                        cell.setCellStyle(currencyStyle);
                    } else if (column.getType() == ColumnType.NUMBER) {
                        cell.setCellStyle(numberStyle);
                    }
                } else if (value instanceof LocalDate) {
                    cell.setCellValue(Date.from(((LocalDate) value)
                        .atStartOfDay(ZoneId.systemDefault()).toInstant()));
                    cell.setCellStyle(dateStyle);
                } else if (value instanceof LocalDateTime) {
                    cell.setCellValue(Date.from(((LocalDateTime) value)
                        .atZone(ZoneId.systemDefault()).toInstant()));
                    cell.setCellStyle(dateStyle);
                } else if (value instanceof Boolean) {
                    cell.setCellValue((Boolean) value);
                } else {
                    cell.setCellValue(value.toString());
                }
            }
        }
    }

    private void autoSizeColumns(Sheet sheet, int columnCount) {
        for (int i = 0; i < columnCount; i++) {
            sheet.autoSizeColumn(i);
            // Add some padding
            int currentWidth = sheet.getColumnWidth(i);
            sheet.setColumnWidth(i, currentWidth + 1000);
        }
    }
}
```

### Configuration Classes

```java
@Data
@Builder
public class ExcelConfig<T> {
    private String sheetName;
    private List<ColumnConfig<T>> columns;
}

@Data
@Builder
public class ColumnConfig<T> {
    private String header;
    private Function<T, Object> valueExtractor;
    private ColumnType type;
}

public enum ColumnType {
    STRING,
    NUMBER,
    CURRENCY,
    DATE,
    BOOLEAN
}
```

## Domain-Specific Export Services

### User Export Service

```java
@Service
@RequiredArgsConstructor
public class UserExportService {

    private final ExcelExportService excelExportService;
    private final UserRepository userRepository;

    public byte[] exportUsers() {
        List<User> users = userRepository.findAll();

        ExcelConfig<User> config = ExcelConfig.<User>builder()
            .sheetName("Users")
            .columns(Arrays.asList(
                ColumnConfig.<User>builder()
                    .header("ID")
                    .valueExtractor(User::getId)
                    .type(ColumnType.NUMBER)
                    .build(),
                ColumnConfig.<User>builder()
                    .header("Name")
                    .valueExtractor(User::getName)
                    .type(ColumnType.STRING)
                    .build(),
                ColumnConfig.<User>builder()
                    .header("Email")
                    .valueExtractor(User::getEmail)
                    .type(ColumnType.STRING)
                    .build(),
                ColumnConfig.<User>builder()
                    .header("Role")
                    .valueExtractor(user -> user.getRole().name())
                    .type(ColumnType.STRING)
                    .build(),
                ColumnConfig.<User>builder()
                    .header("Registration Date")
                    .valueExtractor(User::getCreatedAt)
                    .type(ColumnType.DATE)
                    .build(),
                ColumnConfig.<User>builder()
                    .header("Active")
                    .valueExtractor(User::isActive)
                    .type(ColumnType.BOOLEAN)
                    .build()
            ))
            .build();

        return excelExportService.exportToExcel(users, config);
    }

    public byte[] exportUsersWithFilters(UserFilter filter) {
        List<User> users = userRepository.findByFilter(filter);
        return exportUsersToExcel(users);
    }

    private byte[] exportUsersToExcel(List<User> users) {
        ExcelConfig<User> config = ExcelConfig.<User>builder()
            .sheetName("Users Export " + LocalDate.now())
            .columns(getUserColumns())
            .build();

        return excelExportService.exportToExcel(users, config);
    }

    private List<ColumnConfig<User>> getUserColumns() {
        return Arrays.asList(
            ColumnConfig.<User>builder()
                .header("ID")
                .valueExtractor(User::getId)
                .type(ColumnType.NUMBER)
                .build(),
            ColumnConfig.<User>builder()
                .header("Full Name")
                .valueExtractor(User::getName)
                .type(ColumnType.STRING)
                .build(),
            ColumnConfig.<User>builder()
                .header("Email")
                .valueExtractor(User::getEmail)
                .type(ColumnType.STRING)
                .build(),
            ColumnConfig.<User>builder()
                .header("Phone")
                .valueExtractor(User::getPhone)
                .type(ColumnType.STRING)
                .build(),
            ColumnConfig.<User>builder()
                .header("Role")
                .valueExtractor(u -> u.getRole().getDisplayName())
                .type(ColumnType.STRING)
                .build(),
            ColumnConfig.<User>builder()
                .header("Status")
                .valueExtractor(u -> u.isActive() ? "Active" : "Inactive")
                .type(ColumnType.STRING)
                .build(),
            ColumnConfig.<User>builder()
                .header("Registration Date")
                .valueExtractor(User::getCreatedAt)
                .type(ColumnType.DATE)
                .build(),
            ColumnConfig.<User>builder()
                .header("Last Login")
                .valueExtractor(User::getLastLoginAt)
                .type(ColumnType.DATE)
                .build()
        );
    }
}
```

### Order Export Service

```java
@Service
@RequiredArgsConstructor
public class OrderExportService {

    private final ExcelExportService excelExportService;
    private final OrderRepository orderRepository;

    public byte[] exportOrders(LocalDate startDate, LocalDate endDate) {
        List<Order> orders = orderRepository.findByDateRange(startDate, endDate);

        ExcelConfig<Order> config = ExcelConfig.<Order>builder()
            .sheetName("Orders")
            .columns(Arrays.asList(
                ColumnConfig.<Order>builder()
                    .header("Order Number")
                    .valueExtractor(Order::getOrderNumber)
                    .type(ColumnType.STRING)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Customer")
                    .valueExtractor(order -> order.getCustomer().getName())
                    .type(ColumnType.STRING)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Customer Email")
                    .valueExtractor(order -> order.getCustomer().getEmail())
                    .type(ColumnType.STRING)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Order Date")
                    .valueExtractor(Order::getOrderDate)
                    .type(ColumnType.DATE)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Status")
                    .valueExtractor(order -> order.getStatus().name())
                    .type(ColumnType.STRING)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Items Count")
                    .valueExtractor(order -> order.getItems().size())
                    .type(ColumnType.NUMBER)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Subtotal")
                    .valueExtractor(Order::getSubtotal)
                    .type(ColumnType.CURRENCY)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Shipping")
                    .valueExtractor(Order::getShippingCost)
                    .type(ColumnType.CURRENCY)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Discount")
                    .valueExtractor(Order::getDiscount)
                    .type(ColumnType.CURRENCY)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Total")
                    .valueExtractor(Order::getTotal)
                    .type(ColumnType.CURRENCY)
                    .build(),
                ColumnConfig.<Order>builder()
                    .header("Payment Method")
                    .valueExtractor(order -> order.getPaymentMethod().name())
                    .type(ColumnType.STRING)
                    .build()
            ))
            .build();

        return excelExportService.exportToExcel(orders, config);
    }
}
```

## Advanced Features

### Multi-Sheet Export

```java
@Service
@RequiredArgsConstructor
public class ReportExportService {

    public byte[] exportSalesReport(int year, int month) {
        try (Workbook workbook = new XSSFWorkbook();
             ByteArrayOutputStream outputStream = new ByteArrayOutputStream()) {

            // Sheet 1: Summary
            createSummarySheet(workbook, year, month);

            // Sheet 2: Detailed Orders
            createOrdersSheet(workbook, year, month);

            // Sheet 3: Top Products
            createTopProductsSheet(workbook, year, month);

            // Sheet 4: Customer Analysis
            createCustomerAnalysisSheet(workbook, year, month);

            workbook.write(outputStream);
            return outputStream.toByteArray();

        } catch (IOException e) {
            throw new ExcelExportException("Failed to create sales report", e);
        }
    }

    private void createSummarySheet(Workbook workbook, int year, int month) {
        Sheet sheet = workbook.createSheet("Summary");

        CellStyle titleStyle = createTitleStyle(workbook);
        CellStyle labelStyle = createLabelStyle(workbook);
        CellStyle valueStyle = createValueStyle(workbook);

        // Title
        Row titleRow = sheet.createRow(0);
        Cell titleCell = titleRow.createCell(0);
        titleCell.setCellValue("Sales Report - " + month + "/" + year);
        titleCell.setCellStyle(titleStyle);
        sheet.addMergedRegion(new CellRangeAddress(0, 0, 0, 3));

        // Metrics
        int rowNum = 2;
        addMetric(sheet, rowNum++, "Total Orders:", getTotalOrders(year, month), labelStyle, valueStyle);
        addMetric(sheet, rowNum++, "Total Revenue:", getTotalRevenue(year, month), labelStyle, valueStyle);
        addMetric(sheet, rowNum++, "Average Order Value:", getAverageOrderValue(year, month), labelStyle, valueStyle);
        addMetric(sheet, rowNum++, "New Customers:", getNewCustomers(year, month), labelStyle, valueStyle);

        sheet.autoSizeColumn(0);
        sheet.autoSizeColumn(1);
    }

    private void addMetric(Sheet sheet, int rowNum, String label, Object value, CellStyle labelStyle, CellStyle valueStyle) {
        Row row = sheet.createRow(rowNum);

        Cell labelCell = row.createCell(0);
        labelCell.setCellValue(label);
        labelCell.setCellStyle(labelStyle);

        Cell valueCell = row.createCell(1);
        if (value instanceof Number) {
            valueCell.setCellValue(((Number) value).doubleValue());
        } else {
            valueCell.setCellValue(value.toString());
        }
        valueCell.setCellStyle(valueStyle);
    }
}
```

### Excel with Charts

```java
public void createChartSheet(Workbook workbook, List<SalesData> data) {
    Sheet sheet = workbook.createSheet("Sales Chart");

    // Add data
    int rowNum = 0;
    Row headerRow = sheet.createRow(rowNum++);
    headerRow.createCell(0).setCellValue("Month");
    headerRow.createCell(1).setCellValue("Sales");

    for (SalesData salesData : data) {
        Row row = sheet.createRow(rowNum++);
        row.createCell(0).setCellValue(salesData.getMonth());
        row.createCell(1).setCellValue(salesData.getAmount());
    }

    // Create chart
    Drawing<?> drawing = sheet.createDrawingPatriarch();
    ClientAnchor anchor = drawing.createAnchor(0, 0, 0, 0, 0, 5, 10, 20);

    Chart chart = drawing.createChart(anchor);
    ChartLegend legend = chart.getOrCreateLegend();
    legend.setPosition(LegendPosition.TOP_RIGHT);

    LineChartData chartData = chart.getChartDataFactory().createLineChartData();

    ChartAxis bottomAxis = chart.getChartAxisFactory().createCategoryAxis(AxisPosition.BOTTOM);
    ValueAxis leftAxis = chart.getChartAxisFactory().createValueAxis(AxisPosition.LEFT);
    leftAxis.setCrosses(AxisCrosses.AUTO_ZERO);

    ChartDataSource<Number> xs = DataSources.fromNumericCellRange(sheet,
        new CellRangeAddress(1, data.size(), 0, 0));
    ChartDataSource<Number> ys = DataSources.fromNumericCellRange(sheet,
        new CellRangeAddress(1, data.size(), 1, 1));

    LineChartSeries series = chartData.addSeries(xs, ys);
    series.setTitle("Monthly Sales");

    chart.plot(chartData, bottomAxis, leftAxis);
}
```

### Streaming Export for Large Datasets

```java
@Service
public class StreamingExcelExportService {

    public void exportLargeDataset(List<User> users, OutputStream outputStream) throws IOException {
        try (SXSSFWorkbook workbook = new SXSSFWorkbook(100)) { // Keep 100 rows in memory

            Sheet sheet = workbook.createSheet("Users");

            CellStyle headerStyle = createHeaderStyle(workbook);
            CellStyle dateStyle = createDateStyle(workbook);

            // Header
            Row headerRow = sheet.createRow(0);
            String[] headers = {"ID", "Name", "Email", "Registration Date"};
            for (int i = 0; i < headers.length; i++) {
                Cell cell = headerRow.createCell(i);
                cell.setCellValue(headers[i]);
                cell.setCellStyle(headerStyle);
            }

            // Data rows (streaming)
            int rowNum = 1;
            for (User user : users) {
                Row row = sheet.createRow(rowNum++);
                row.createCell(0).setCellValue(user.getId());
                row.createCell(1).setCellValue(user.getName());
                row.createCell(2).setCellValue(user.getEmail());

                Cell dateCell = row.createCell(3);
                dateCell.setCellValue(Date.from(user.getCreatedAt()
                    .atZone(ZoneId.systemDefault()).toInstant()));
                dateCell.setCellStyle(dateStyle);

                // Flush rows to disk when window is full
                if (rowNum % 100 == 0) {
                    ((SXSSFSheet) sheet).flushRows();
                }
            }

            workbook.write(outputStream);
            workbook.dispose(); // Clean up temporary files
        }
    }
}
```

## REST Controller Integration

```java
@RestController
@RequestMapping("/api/exports")
@RequiredArgsConstructor
public class ExportController {

    private final UserExportService userExportService;
    private final OrderExportService orderExportService;

    @GetMapping("/users")
    public ResponseEntity<byte[]> exportUsers() {
        byte[] excelData = userExportService.exportUsers();

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.parseMediaType(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"));
        headers.setContentDispositionFormData("attachment",
            "users-export-" + LocalDate.now() + ".xlsx");

        return ResponseEntity.ok()
            .headers(headers)
            .body(excelData);
    }

    @GetMapping("/orders")
    public ResponseEntity<byte[]> exportOrders(
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate startDate,
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate endDate
    ) {
        byte[] excelData = orderExportService.exportOrders(startDate, endDate);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.parseMediaType(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"));
        headers.setContentDispositionFormData("attachment",
            String.format("orders-%s-to-%s.xlsx", startDate, endDate));

        return ResponseEntity.ok()
            .headers(headers)
            .body(excelData);
    }

    @PostMapping("/users/filtered")
    public ResponseEntity<byte[]> exportFilteredUsers(@RequestBody UserFilter filter) {
        byte[] excelData = userExportService.exportUsersWithFilters(filter);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.parseMediaType(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"));
        headers.setContentDispositionFormData("attachment",
            "users-filtered-export.xlsx");

        return ResponseEntity.ok()
            .headers(headers)
            .body(excelData);
    }

    @GetMapping("/sales/report")
    public ResponseEntity<byte[]> exportSalesReport(
        @RequestParam int year,
        @RequestParam int month
    ) {
        byte[] excelData = reportExportService.exportSalesReport(year, month);

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.parseMediaType(
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"));
        headers.setContentDispositionFormData("attachment",
            String.format("sales-report-%d-%02d.xlsx", year, month));

        return ResponseEntity.ok()
            .headers(headers)
            .body(excelData);
    }
}
```

## Excel Import Service

```java
@Service
@Slf4j
public class ExcelImportService {

    public <T> List<T> importFromExcel(
        InputStream inputStream,
        ExcelImportConfig<T> config
    ) throws IOException {
        List<T> results = new ArrayList<>();

        try (Workbook workbook = new XSSFWorkbook(inputStream)) {
            Sheet sheet = workbook.getSheetAt(0);

            // Skip header row
            for (int i = 1; i <= sheet.getLastRowNum(); i++) {
                Row row = sheet.getRow(i);
                if (row == null) continue;

                try {
                    T entity = config.getRowMapper().apply(row);
                    results.add(entity);
                } catch (Exception e) {
                    log.warn("Failed to parse row {}: {}", i, e.getMessage());
                }
            }
        }

        return results;
    }

    public List<User> importUsers(InputStream inputStream) throws IOException {
        ExcelImportConfig<User> config = ExcelImportConfig.<User>builder()
            .rowMapper(row -> User.builder()
                .name(getCellValueAsString(row.getCell(0)))
                .email(getCellValueAsString(row.getCell(1)))
                .phone(getCellValueAsString(row.getCell(2)))
                .build())
            .build();

        return importFromExcel(inputStream, config);
    }

    private String getCellValueAsString(Cell cell) {
        if (cell == null) return null;

        return switch (cell.getCellType()) {
            case STRING -> cell.getStringCellValue();
            case NUMERIC -> String.valueOf((long) cell.getNumericCellValue());
            case BOOLEAN -> String.valueOf(cell.getBooleanCellValue());
            default -> null;
        };
    }
}
```

## Testing

```java
@SpringBootTest
class ExcelExportServiceTest {

    @Autowired
    private ExcelExportService excelExportService;

    @Test
    @DisplayName("Should export users to Excel successfully")
    void shouldExportUsersToExcel() throws IOException {
        // Given
        List<User> users = Arrays.asList(
            User.builder().id(1L).name("John Doe").email("john@example.com").build(),
            User.builder().id(2L).name("Jane Smith").email("jane@example.com").build()
        );

        ExcelConfig<User> config = ExcelConfig.<User>builder()
            .sheetName("Users")
            .columns(Arrays.asList(
                ColumnConfig.<User>builder()
                    .header("ID")
                    .valueExtractor(User::getId)
                    .type(ColumnType.NUMBER)
                    .build(),
                ColumnConfig.<User>builder()
                    .header("Name")
                    .valueExtractor(User::getName)
                    .type(ColumnType.STRING)
                    .build()
            ))
            .build();

        // When
        byte[] excelData = excelExportService.exportToExcel(users, config);

        // Then
        assertThat(excelData).isNotEmpty();

        // Verify content
        try (Workbook workbook = new XSSFWorkbook(new ByteArrayInputStream(excelData))) {
            Sheet sheet = workbook.getSheet("Users");
            assertThat(sheet).isNotNull();
            assertThat(sheet.getLastRowNum()).isEqualTo(2); // Header + 2 data rows

            Row headerRow = sheet.getRow(0);
            assertThat(headerRow.getCell(0).getStringCellValue()).isEqualTo("ID");
            assertThat(headerRow.getCell(1).getStringCellValue()).isEqualTo("Name");

            Row dataRow = sheet.getRow(1);
            assertThat(dataRow.getCell(0).getNumericCellValue()).isEqualTo(1.0);
            assertThat(dataRow.getCell(1).getStringCellValue()).isEqualTo("John Doe");
        }
    }
}
```

## Best Practices

1. ✅ Use `SXSSFWorkbook` for large datasets (> 10,000 rows)
2. ✅ Always close workbooks and streams in try-with-resources
3. ✅ Apply auto-sizing after all data is written
4. ✅ Use cell styles efficiently (reuse style objects)
5. ✅ Add data validation for input cells
6. ✅ Include filters on header rows for better UX
7. ✅ Use proper date formatting based on locale
8. ✅ Handle null values gracefully
9. ✅ Add metadata (creation date, author) to workbooks
10. ✅ Test exports with real data samples

## References

- [Apache POI Documentation](https://poi.apache.org/components/spreadsheet/)
- [Apache POI Quick Guide](https://poi.apache.org/components/spreadsheet/quick-guide.html)
- [Excel Best Practices](https://support.microsoft.com/en-us/office/excel-specifications-and-limits-1672b34d-7043-467e-8e27-269d656771c3)
