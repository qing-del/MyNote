This is a very common error when working with Apache POI!

### Understanding the Error

The `java.lang.IllegalStateException: Cannot get a STRING value from a NUMERIC cell` happens because POI is strictly typed when reading cells.
In your `getAllEmployees()` method, you are fetching the `id` as a `Long` (`rs.getLong("id")`). When `testWrite()` writes this into the Excel file, it saves it as a **Numeric** cell. However, in your `testRead()` method on line 73, you are telling POI to read it as a **String**:
`String id = row.getCell(1).getStringCellValue();`

### How to Fix It

You have two great ways to solve this. You can either use a `DataFormatter` (which is the safest and most flexible method), or you can explicitly read it as a number and cast it to a String.

#### Option 1: Use `DataFormatter` (Best Practice)
`DataFormatter` is a handy utility in Apache POI that automatically converts whatever is in the cell into a String, exactly as it appears in Excel.
Modify your `testRead()` method like this:

```Java
import org.apache.poi.ss.usermodel.DataFormatter; // Add this import

// ... inside your testRead() method ...

    for (int i = 1; i <= lastRowNum; i++) {
        // 获取某一行
        XSSFRow row = sheet.getRow(i);
        
        // Use DataFormatter to safely extract any cell type as a String
        DataFormatter formatter = new DataFormatter();
        String id = formatter.formatCellValue(row.getCell(1));
        
        // Name is already a string, but you can use formatter here too if you prefer!
        String name = row.getCell(2).getStringCellValue(); 
        
        log.info("id: {}, name: {}", id, name);
    }
```

#### Option 2: Read as Numeric

Since you know the `id` column is a number, you can specifically use `getNumericCellValue()`. Because POI reads numbers as `double` by default, you will want to cast it to a `long` to remove any decimal points before converting it to a String.

```Java
    for (int i = 1; i <= lastRowNum; i++) {
        // 获取某一行
        XSSFRow row = sheet.getRow(i);
        
        // Read as a number, cast to long to remove decimals, then convert to String
        long numericId = (long) row.getCell(1).getNumericCellValue();
        String id = String.valueOf(numericId);
        
        String name = row.getCell(2).getStringCellValue();
        log.info("id: {}, name: {}", id, name);
    }
```

---

**A quick tip for `testWrite()`:** Since you commented out `@Test` on `testWrite()`, make sure you've actually run it recently to generate the `info.xlsx` file with the correct database data before running `testRead()`.