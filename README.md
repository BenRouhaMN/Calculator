
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

        // Check size
        byte[] bytes = is.readAllBytes();
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
