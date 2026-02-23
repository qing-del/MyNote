---
title: Apache POI
tags:
 - Apache
 - Java
 - Office
create_time: 2026-02-20
---

# 目录

---

## 介绍

---

## 入门案例

### Maven 依赖
```xml
<dependency>  
    <groupId>org.apache.poi</groupId>  
    <artifactId>poi</artifactId>
    <version>3.16<version>
</dependency>  
  
<dependency>  
    <groupId>org.apache.poi</groupId>  
    <artifactId>poi-ooxml</artifactId>
    <version>3.16<version>  
</dependency>
```

```Java
// POITest.java
package com.sky.test;  
  
import com.sky.entity.Employee;  
import lombok.extern.slf4j.Slf4j;  
import org.apache.poi.xssf.usermodel.XSSFRow;  
import org.apache.poi.xssf.usermodel.XSSFSheet;  
import org.apache.poi.xssf.usermodel.XSSFWorkbook;  
import org.junit.jupiter.api.Test;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.boot.test.context.SpringBootTest;  
import org.springframework.boot.test.mock.mockito.MockBean;  
import org.springframework.web.socket.server.standard.ServerEndpointExporter;  
  
import javax.sql.DataSource;  
import java.io.FileInputStream;  
import java.io.FileOutputStream;  
import java.io.InputStream;  
import java.sql.*;  
import java.util.ArrayList;  
  
@Slf4j  
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)  
public class POITest {  
    @Autowired private DataSource dataSource;  
	// 防止因为WebSocket没有Web环境导致的测试错误
    @MockBean private ServerEndpointExporter serverEndpointExporter;  
  
    @Test  
    public void testWrite() throws Exception {  
        // 创建一个Excel文件  
        XSSFWorkbook workbook = new XSSFWorkbook();  
        // 在Excel中创建个一个Sheet页  
        XSSFSheet sheet = workbook.createSheet("info");  
        // 在Sheet页中创建行，下标是从0开始的  
        XSSFRow row = sheet.createRow(0);  
        row.createCell(1).setCellValue("id");  
        row.createCell(2).setCellValue("name");  
  
        // 获取数据库中的数据  
        ArrayList<Employee> list = getAllEmployees();  
        for (Employee employee: list) {  
            XSSFRow row1 = sheet.createRow(sheet.getLastRowNum() + 1);  
            row1.createCell(1).setCellValue(employee.getId().toString());  
            row1.createCell(2).setCellValue(employee.getName());  
        }  
  
        // 输出Excel文件  
        FileOutputStream outputStream = new FileOutputStream("D:/info.xlsx");  
        workbook.write(outputStream);  
  
        // 关闭流  
        outputStream.close();  
        workbook.close();  
    }  
  
    @Test  
    public void testRead() throws Exception {  
        InputStream in = null;  
        XSSFWorkbook excel = null;  
        try {  
            in = new FileInputStream("D:/info.xlsx");  
            excel = new XSSFWorkbook(in);  
            XSSFSheet sheet = excel.getSheetAt(0);  
            int lastRowNum = sheet.getLastRowNum();  
  
            for (int i = 1; i <= lastRowNum; i++) {  
                // 获取某一行  
                XSSFRow row = sheet.getRow(i);  
                // 获得单元格对象  
                String id = row.getCell(1).getStringCellValue();  
                String name = row.getCell(2).getStringCellValue();  
                log.info("id: {}, name: {}", id, name);  
            }  
        } finally {  
            if (in != null) in.close();  
            if (excel != null) excel.close();  
        }  
    }  
  
    /**  
     * 获取数据库中的所有员工数据  
     * @return  
     */  
    private ArrayList<Employee> getAllEmployees() throws Exception {  
        // 使用 try-with-resources 语法，自动关闭流，防止内存泄漏  
        try (Connection conn = dataSource.getConnection();  
             PreparedStatement pstmt = conn.prepareStatement("select id, name from employee");  
             ResultSet rs = pstmt.executeQuery()) {  
  
            ArrayList<Employee> list = new ArrayList<>();  
            while (rs.next()) {  
                Employee employee = new Employee();  
                employee.setId(rs.getLong("id"));  
                employee.setName(rs.getString("name"));  
                list.add(employee);  
            }  
            return list;  
        }  
    }  
}
```