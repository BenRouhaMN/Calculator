import org.apache.poi.ss.usermodel.Workbook;
import org.apache.poi.ss.usermodel.WorkbookFactory;
import org.junit.Test;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;

public class TemplateDebugTest {

    @Test
    public void debugTemplateLoading() throws Exception {

        System.out.println("=== TEMPLATE DEBUG START ===");

        String[] templates = {
                "/templates/representativeness.xlsx",
                "/templates/risk.xlsx"
        };

        for (String path : templates) {

            System.out.println("\nChecking: " + path);

            InputStream is = getClass().getResourceAsStream(path);

            if (is == null) {
                System.out.println("❌ ERROR: getResourceAsStream returned null");
                System.out.println("➡ Meaning: File NOT FOUND in the classpath.");
                continue;
            }

            // Convert InputStream to byte[] (Java 8 way)
            ByteArrayOutputStream buffer = new ByteArrayOutputStream();
            byte[] data = new byte[4096];
            int nRead;
            while ((nRead = is.read(data, 0, data.length)) != -1) {
                buffer.write(data, 0, nRead);
            }
            buffer.flush();
            byte[] bytes = buffer.toByteArray();

            System.out.println("✔ File loaded, size = " + bytes.length + " bytes");

            if (bytes.length < 200) {
                System.out.println("⚠ WARNING: file is unusually small → PROBABLY CORRUPTED");
            }

            // Try opening with POI
            try (InputStream bis = new ByteArrayInputStream(bytes)) {
                Workbook workbook = WorkbookFactory.create(bis);
                System.out.println("✔ Apache POI can open workbook");
                System.out.println("→ Number of sheets: " + workbook.getNumberOfSheets());
            } catch (Exception e) {
                System.out.println("❌ POI FAILED TO OPEN WORKBOOK:");
                e.printStackTrace();
            }
        }

        System.out.println("\n=== TEMPLATE DEBUG END ===");
    }
}
