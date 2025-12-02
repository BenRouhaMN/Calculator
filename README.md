Super — je te fournis le code final prêt à copier/coller.
C’est un petit projet Spring Boot (Maven) complet qui implémente :

les 2 templates (Representativeness & RiskActivities) reproduisant la structure des screenshots (layout / merges / styles simples verts),

un template fallback DefaultGenericTemplate qui génère automatiquement toutes les colonnes d’un DTO,

un contrôleur REST pour télécharger le fichier .xlsx.



---

Structure du projet (packages)

com.example.report
 ├─ ReportApplication.java
 ├─ controller
 │    └─ ReportController.java
 ├─ domain
 │    └─ TradeRecord.java
 ├─ dto
 │    └─ ExcelRowDTO.java
 ├─ mapper
 │    └─ TradeExcelMapper.java
 ├─ repository
 │    └─ TradeRepository.java   (exemple Spring Data ; optionnel)
 ├─ service
 │    └─ ReportingService.java
 └─ excel
      ├─ template
      │    ├─ ExcelTemplate.java
      │    ├─ RepresentativenessTemplate.java
      │    ├─ RiskActivitiesTemplate.java
      │    └─ DefaultGenericTemplate.java
      ├─ ExcelReportGenerator.java
      ├─ ExcelStyleFactory.java
      └─ ExcelSectionBuilder.java

Je mets les fichiers essentiels ci-dessous. Tu peux t’en inspirer, les coller dans ton IDE, lancer mvn spring-boot:run et appeler l’endpoint pour tester.


---

pom.xml

<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>excel-report</artifactId>
  <version>1.0.0</version>
  <properties>
    <java.version>17</java.version>
    <spring.boot.version>3.2.0</spring.boot.version>
    <poi.version>5.2.3</poi.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <version>${spring.boot.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi-ooxml</artifactId>
      <version>${poi.version}</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-jpa</artifactId>
      <version>${spring.boot.version}</version>
    </dependency>
    <!-- H2 pour tests rapides -->
    <dependency>
      <groupId>com.h2database</groupId>
      <artifactId>h2</artifactId>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>${spring.boot.version}</version>
      </plugin>
    </plugins>
  </build>
</project>


---

ReportApplication.java

package com.example.report;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ReportApplication {
    public static void main(String[] args) {
        SpringApplication.run(ReportApplication.class, args);
    }
}


---

Domain / DTO / Mapper

domain/TradeRecord.java

package com.example.report.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDate;

@Entity
@Table(name = "trade_record")
public class TradeRecord {
    @Id @GeneratedValue
    private Long id;
    private String productType;   // ex: "EUR OTC IRD", "PLN OTC IRD", "EUR STIR"
    private String ccpName;       // ex: "CCP1"
    private String category;      // ex: "25(2)(c)" or "14"
    private String maturityBucket; // ex: "[0-6M]"
    private String tradeSizeBucket;// ex: "Any trade size"
    private BigDecimal notional;
    private LocalDate tradeDate;

    // getters & setters omitted for brevity (générer via IDE)
}

dto/ExcelRowDTO.java

package com.example.report.dto;

import java.math.BigDecimal;

public class ExcelRowDTO {
    private String productType;
    private String ccpName;
    private String category;
    private String maturityBucket;
    private String tradeSizeBucket;
    private BigDecimal notional;
    // getters/setters
    // constructeur utile
}

mapper/TradeExcelMapper.java

package com.example.report.mapper;

import com.example.report.domain.TradeRecord;
import com.example.report.dto.ExcelRowDTO;

public class TradeExcelMapper {
    public static ExcelRowDTO toDto(TradeRecord r) {
        ExcelRowDTO dto = new ExcelRowDTO();
        dto.setProductType(r.getProductType());
        dto.setCcpName(r.getCcpName());
        dto.setCategory(r.getCategory());
        dto.setMaturityBucket(r.getMaturityBucket());
        dto.setTradeSizeBucket(r.getTradeSizeBucket());
        dto.setNotional(r.getNotional());
        return dto;
    }
}


---

Repository (optionnel) — pour tests

repository/TradeRepository.java

package com.example.report.repository;

import com.example.report.domain.TradeRecord;
import org.springframework.data.jpa.repository.JpaRepository;
public interface TradeRepository extends JpaRepository<TradeRecord, Long> {}


---

Excel helpers

excel/ExcelStyleFactory.java

package com.example.report.excel;

import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;

public class ExcelStyleFactory {

    public static CellStyle topHeaderStyle(Workbook wb) {
        CellStyle s = wb.createCellStyle();
        Font f = wb.createFont();
        f.setBold(true);
        f.setColor(IndexedColors.WHITE.getIndex());
        f.setFontHeightInPoints((short)12);
        s.setFont(f);
        s.setAlignment(HorizontalAlignment.CENTER);
        s.setVerticalAlignment(VerticalAlignment.CENTER);
        s.setFillForegroundColor(IndexedColors.DARK_GREEN.getIndex());
        s.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        return s;
    }

    public static CellStyle sectionHeaderStyle(Workbook wb) {
        CellStyle s = wb.createCellStyle();
        Font f = wb.createFont();
        f.setBold(true);
        f.setColor(IndexedColors.WHITE.getIndex());
        s.setFont(f);
        s.setAlignment(HorizontalAlignment.CENTER);
        s.setFillForegroundColor(IndexedColors.GREEN.getIndex());
        s.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        return s;
    }

    public static CellStyle lightCellStyle(Workbook wb) {
        CellStyle s = wb.createCellStyle();
        Font f = wb.createFont();
        f.setFontHeightInPoints((short)10);
        s.setFont(f);
        s.setFillForegroundColor(IndexedColors.LIGHT_GREEN.getIndex());
        s.setFillPattern(FillPatternType.SOLID_FOREGROUND);
        s.setAlignment(HorizontalAlignment.CENTER);
        s.setVerticalAlignment(VerticalAlignment.CENTER);
        s.setWrapText(true);
        return s;
    }

    public static CellStyle normalStyle(Workbook wb) {
        CellStyle s = wb.createCellStyle();
        s.setAlignment(HorizontalAlignment.LEFT);
        s.setVerticalAlignment(VerticalAlignment.CENTER);
        return s;
    }
}

excel/ExcelSectionBuilder.java

package com.example.report.excel;

import org.apache.poi.ss.usermodel.*;
import org.apache.poi.ss.util.CellRangeAddress;
import java.util.List;
import com.example.report.dto.ExcelRowDTO;

public class ExcelSectionBuilder {

    // Crée un titre fusionné (rowIndex, colFirst..colLast)
    public static void createTitle(Sheet sheet, int rowIndex, int colFirst, int colLast, String title, CellStyle style, int height) {
        Row r = getOrCreateRow(sheet, rowIndex);
        r.setHeightInPoints(height);
        Cell c = r.createCell(colFirst);
        c.setCellValue(title);
        c.setCellStyle(style);
        sheet.addMergedRegion(new CellRangeAddress(rowIndex, rowIndex, colFirst, colLast));
        // appliquer style aux autres cellules fusionnées
        for (int i = colFirst; i <= colLast; i++) {
            Cell cc = r.getCell(i);
            if (cc == null) cc = r.createCell(i);
            cc.setCellStyle(style);
        }
    }

    public static void createSimpleHeader(Sheet sheet, int rowIndex, String[] headers, CellStyle style) {
        Row r = getOrCreateRow(sheet, rowIndex);
        for (int i = 0; i < headers.length; i++) {
            Cell c = r.createCell(i);
            c.setCellValue(headers[i]);
            c.setCellStyle(style);
        }
    }

    public static int writeRows(Sheet sheet, int startRow, List<ExcelRowDTO> rows, CellStyle style) {
        int rIdx = startRow;
        for (ExcelRowDTO dto : rows) {
            Row r = getOrCreateRow(sheet, rIdx++);
            r.createCell(0).setCellValue(dto.getProductType() == null ? "" : dto.getProductType());
            r.createCell(1).setCellValue(dto.getCcpName() == null ? "" : dto.getCcpName());
            r.createCell(2).setCellValue(dto.getCategory() == null ? "" : dto.getCategory());
            r.createCell(3).setCellValue(dto.getMaturityBucket() == null ? "" : dto.getMaturityBucket());
            r.createCell(4).setCellValue(dto.getTradeSizeBucket() == null ? "" : dto.getTradeSizeBucket());
            r.createCell(5).setCellValue(dto.getNotional() == null ? "" : dto.getNotional().toPlainString());
            for (int c = 0; c <= 5; c++) r.getCell(c).setCellStyle(style);
        }
        return rIdx;
    }

    private static Row getOrCreateRow(Sheet s, int rowIdx) {
        Row r = s.getRow(rowIdx);
        if (r == null) r = s.createRow(rowIdx);
        return r;
    }
}


---

Templates

excel/template/ExcelTemplate.java

package com.example.report.excel.template;

import java.util.List;
import com.example.report.domain.TradeRecord;
import org.apache.poi.ss.usermodel.Workbook;
import org.apache.poi.ss.usermodel.Sheet;

public interface ExcelTemplate {
    void prepareData(List<TradeRecord> records);
    void buildExcel(Workbook wb, Sheet sheet);
}

excel/template/RepresentativenessTemplate.java

package com.example.report.excel.template;

import com.example.report.dto.ExcelRowDTO;
import com.example.report.mapper.TradeExcelMapper;
import com.example.report.domain.TradeRecord;
import com.example.report.excel.ExcelStyleFactory;
import com.example.report.excel.ExcelSectionBuilder;
import org.apache.poi.ss.usermodel.*;
import java.util.List;
import java.util.stream.Collectors;

public class RepresentativenessTemplate implements ExcelTemplate {

    private List<ExcelRowDTO> leftBlock;
    private List<ExcelRowDTO> rightBlock;

    @Override
    public void prepareData(List<TradeRecord> records) {
        List<ExcelRowDTO> rows = records.stream().map(TradeExcelMapper::toDto).collect(Collectors.toList());
        leftBlock = rows.stream().filter(r -> "25(2)(c)".equals(r.getCategory()) || "EUR STIR ESTR".equals(r.getProductType())).collect(Collectors.toList());
        rightBlock = rows.stream().filter(r -> "14".equals(r.getCategory()) || "EUR STIR ESTR".equals(r.getProductType())).collect(Collectors.toList());
    }

    @Override
    public void buildExcel(Workbook wb, Sheet sheet) {
        // styles
        CellStyle top = ExcelStyleFactory.topHeaderStyle(wb);
        CellStyle section = ExcelStyleFactory.sectionHeaderStyle(wb);
        CellStyle light = ExcelStyleFactory.lightCellStyle(wb);
        CellStyle normal = ExcelStyleFactory.normalStyle(wb);

        // col widths
        for (int i=0;i<12;i++) sheet.setColumnWidth(i, 18*256);

        int row = 0;
        ExcelSectionBuilder.createTitle(sheet, row++, 0, 11, "Reference period", top, 18);

        // leave one empty row
        row++;

        // Create left block title (columns 0..5) and right block (6..11)
        ExcelSectionBuilder.createTitle(sheet, row, 0, 5, "Clearing service of substantial systemic importance under Article 25(2c)", section, 14);
        ExcelSectionBuilder.createTitle(sheet, row, 6, 11, "Authorised CCP under Article 14", section, 14);
        row++;

        // sub-headers
        ExcelSectionBuilder.createSimpleHeader(sheet, row++, new String[]{"EUR STIR ESTR","Trade size (in EUR million)","","","EUR STIR ESTR","Trade size (in EUR million)",""}, light);

        // example: write the rows under left and right blocks side by side
        int dataStart = row;
        int max = Math.max(leftBlock.size(), rightBlock.size());
        for (int i = 0; i < max; i++) {
            Row r = sheet.createRow(dataStart + i);
            if (i < leftBlock.size()) {
                ExcelRowDTO l = leftBlock.get(i);
                r.createCell(0).setCellValue(l.getMaturityBucket() == null ? "" : l.getMaturityBucket());
                r.createCell(1).setCellValue(l.getTradeSizeBucket() == null ? "" : l.getTradeSizeBucket());
                r.createCell(2).setCellValue(l.getCcpName() == null ? "" : l.getCcpName());
            }
            if (i < rightBlock.size()) {
                ExcelRowDTO rr = rightBlock.get(i);
                r.createCell(6).setCellValue(rr.getMaturityBucket() == null ? "" : rr.getMaturityBucket());
                r.createCell(7).setCellValue(rr.getTradeSizeBucket() == null ? "" : rr.getTradeSizeBucket());
                r.createCell(8).setCellValue(rr.getCcpName() == null ? "" : rr.getCcpName());
            }
            // apply style
            for (int c=0;c<=11;c++) {
                Cell cell = r.getCell(c);
                if (cell != null) cell.setCellStyle(normal);
            }
        }
    }
}

> Remarque : ce template fabrique la structure générale visible sur le screenshot. Tu peux affiner les merges, hauteurs, et colonnes exactes selon tes besoins — le pattern est là.



excel/template/RiskActivitiesTemplate.java

package com.example.report.excel.template;

import com.example.report.dto.ExcelRowDTO;
import com.example.report.mapper.TradeExcelMapper;
import com.example.report.domain.TradeRecord;
import com.example.report.excel.ExcelStyleFactory;
import com.example.report.excel.ExcelSectionBuilder;
import org.apache.poi.ss.usermodel.*;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public class RiskActivitiesTemplate implements ExcelTemplate {

    private List<ExcelRowDTO> allRows;
    private Map<String, List<ExcelRowDTO>> byCategory;
    private Map<String, Map<String, List<ExcelRowDTO>>> byCategoryByCcp;

    @Override
    public void prepareData(List<TradeRecord> records) {
        allRows = records.stream().map(TradeExcelMapper::toDto).collect(Collectors.toList());
        byCategory = allRows.stream().collect(Collectors.groupingBy(ExcelRowDTO::getProductType));
        byCategoryByCcp = allRows.stream()
                .collect(Collectors.groupingBy(ExcelRowDTO::getProductType,
                    Collectors.groupingBy(ExcelRowDTO::getCcpName)));
    }

    @Override
    public void buildExcel(Workbook wb, Sheet sheet) {
        CellStyle top = ExcelStyleFactory.topHeaderStyle(wb);
        CellStyle section = ExcelStyleFactory.sectionHeaderStyle(wb);
        CellStyle light = ExcelStyleFactory.lightCellStyle(wb);
        CellStyle normal = ExcelStyleFactory.normalStyle(wb);

        for (int i=0;i<16;i++) sheet.setColumnWidth(i, 18*256);

        int row = 0;
        ExcelSectionBuilder.createTitle(sheet, row++, 0, 15, "Entity: [YourEntity]   Total", top, 20);
        row++;

        // Dimension 1 header
        ExcelSectionBuilder.createTitle(sheet, row, 0, 4, "Dimension 1 – Break down by category of derivative", section, 14);
        row++;

        // For each category, create a small header area
        int startCol = 0;
        for (String category : byCategory.keySet()) {
            ExcelSectionBuilder.createTitle(sheet, row, startCol, startCol + 4, category, light, 14);
            startCol += 5;
        }
        row += 2;

        // Dimension 2 – CCP breakdown
        ExcelSectionBuilder.createTitle(sheet, row++, 0, 15, "Dimension 2 – Breakdown by CCP (reporting at CCP LEI level)", section, 14);
        // CCP labels row
        Row ccprow = sheet.createRow(row++);
        int c = 0;
        for (String category : byCategory.keySet()) {
            Map<String, List<ExcelRowDTO>> map = byCategoryByCcp.get(category);
            if (map != null) {
                int localCol = c;
                for (String ccp : map.keySet()) {
                    Cell cell = ccprow.createCell(localCol++);
                    cell.setCellValue(ccp);
                    cell.setCellStyle(light);
                }
            }
            c += 5;
        }

        // leave rows for data (empty for now)
    }
}

excel/template/DefaultGenericTemplate.java

package com.example.report.excel.template;

import com.example.report.domain.TradeRecord;
import com.example.report.dto.ExcelRowDTO;
import com.example.report.mapper.TradeExcelMapper;
import org.apache.poi.ss.usermodel.*;
import java.util.List;
import java.lang.reflect.Field;

public class DefaultGenericTemplate implements ExcelTemplate {

    private List<ExcelRowDTO> rows;

    @Override
    public void prepareData(List<TradeRecord> records) {
        rows = records.stream().map(TradeExcelMapper::toDto).toList();
    }

    @Override
    public void buildExcel(Workbook wb, Sheet sheet) {
        CellStyle header = wb.createCellStyle();
        Font f = wb.createFont(); f.setBold(true);
        header.setFont(f);

        if (rows.isEmpty()) return;

        // dynamicaly use fields of ExcelRowDTO
        Class<?> clazz = rows.get(0).getClass();
        Field[] fields = clazz.getDeclaredFields();

        Row h = sheet.createRow(0);
        for (int i = 0; i < fields.length; i++) {
            sheet.setColumnWidth(i, 20*256);
            Cell c = h.createCell(i);
            c.setCellValue(fields[i].getName());
            c.setCellStyle(header);
        }

        int rIdx = 1;
        for (Object obj : rows) {
            Row row = sheet.createRow(rIdx++);
            for (int i = 0; i < fields.length; i++) {
                Field fld = fields[i];
                fld.setAccessible(true);
                try {
                    Object val = fld.get(obj);
                    row.createCell(i).setCellValue(val == null ? "" : val.toString());
                } catch (IllegalAccessException e) {
                    row.createCell(i).setCellValue("");
                }
            }
        }
    }
}


---

Générateur central

excel/ExcelReportGenerator.java

package com.example.report.excel;

import com.example.report.domain.TradeRecord;
import com.example.report.excel.template.*;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import java.io.ByteArrayOutputStream;
import java.util.List;

public class ExcelReportGenerator {

    public static byte[] generate(List<TradeRecord> records, String templateName) {
        ExcelTemplate tmpl = selectTemplate(templateName);
        tmpl.prepareData(records);

        try (Workbook wb = new XSSFWorkbook(); ByteArrayOutputStream out = new ByteArrayOutputStream()) {
            Sheet sheet = wb.createSheet("Report");
            tmpl.buildExcel(wb, sheet);
            wb.write(out);
            return out.toByteArray();
        } catch (Exception e) {
            throw new RuntimeException("Error generating Excel", e);
        }
    }

    private static ExcelTemplate selectTemplate(String name) {
        if (name == null) return new DefaultGenericTemplate();
        return switch (name.toUpperCase()) {
            case "REPRESENTATIVENESS", "REP" -> new RepresentativenessTemplate();
            case "RISK", "RISK_ACTIVITIES" -> new RiskActivitiesTemplate();
            default -> new DefaultGenericTemplate();
        };
    }
}


---

Service + Controller

service/ReportingService.java

package com.example.report.service;

import com.example.report.domain.TradeRecord;
import com.example.report.excel.ExcelReportGenerator;
import com.example.report.repository.TradeRepository;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class ReportingService {

    private final TradeRepository repo;
    public ReportingService(TradeRepository repo) {
        this.repo = repo;
    }

    public byte[] generate(String templateName) {
        List<TradeRecord> records = repo.findAll();
        return ExcelReportGenerator.generate(records, templateName);
    }
}

controller/ReportController.java

package com.example.report.controller;

import com.example.report.service.ReportingService;
import org.springframework.core.io.ByteArrayResource;
import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/report")
public class ReportController {

    private final ReportingService reportingService;
    public ReportController(ReportingService reportingService) {
        this.reportingService = reportingService;
    }

    @GetMapping("/download")
    public ResponseEntity<ByteArrayResource> download(@RequestParam(required = false) String template) {
        byte[] bytes = reportingService.generate(template);
        ByteArrayResource resource = new ByteArrayResource(bytes);
        String filename = (template == null ? "generic" : template.toLowerCase()) + "-report.xlsx";

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=" + filename)
            .contentType(MediaType.parseMediaType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"))
            .contentLength(bytes.length)
            .body(resource);
    }
}


---

Instructions pour tester rapidement

1. Copier/coller les fichiers dans un projet Maven (respecter les packages).


2. Lancer mvn spring-boot:run.


3. Ajouter des données de test dans trade_record (H2 console ou via un data.sql) ou adapter ReportingService pour renvoyer des données simulées.


4. Appeler :

GET http://localhost:8080/api/report/download?template=REPRESENTATIVENESS

GET http://localhost:8080/api/report/download?template=RISK

GET http://localhost:8080/api/report/download (fallback générique)





---

Remarques & améliorations possibles (production)

Pour très gros volumes : remplacer XSSFWorkbook par SXSSFWorkbook (streaming) et gérer dispose() des fichiers temporaires.

Pour layout très exact : mieux partir d’un template Excel (.xlsx) et ne remplir que les cellules (méthode plus simple pour designers métier).

Réutiliser au maximum les CellStyle (déjà fait via ExcelStyleFactory) — éviter d’en créer trop (limite POI).

Ajouter des tests unitaires : générer le workbook, relire son contenu et assert valeurs & merges.

Si export long : implémenter job asynchrone et stocker le fichier (S3, disque) puis renvoyer URL.



---

Si tu veux, je peux maintenant :

1. Te fournir le fichier data.sql d’exemple pour peupler trade_record et tester (valeurs de maturities, ccps, catégories) ;



2. Ajuster les templates pour reproduire à l’identique les merges/largeurs/hauteurs/nuances vertes de tes captures ;



3. Te donner une version template-file (lire un .xlsx existant et ne remplacer que les cellules) — très maintenable quand les métiers changent la mise en page.




Laquelle préfères-tu en priorité ?
