
import java.io.*;
import java.util.zip.ZipEntry;
import java.util.zip.ZipFile;
import java.util.zip.ZipOutputStream;

public void zipIt(String zipFile, String fromFolder) {

    byte[] buffer = new byte[1024];
    boolean hasAtLeastOneValidFile = false;

    try (FileOutputStream fos = new FileOutputStream(zipFile);
         ZipOutputStream zos = new ZipOutputStream(fos)) {

        for (String file : this.fileList) {

            File srcFile = new File(fromFolder + File.separator + file);

            // 1️⃣ Fichier invalide
            if (!srcFile.exists() || !srcFile.isFile()) {
                continue;
            }

            // 2️⃣ Fichier classique vide
            if (srcFile.length() == 0) {
                continue;
            }

            // 3️⃣ CAS CLÉ : ZIP vide mais size > 0
            if (file.toLowerCase().endsWith(".zip") && !zipContainsRealFile(srcFile)) {
                continue;
            }

            // ✅ À ce stade, on est sûr d’avoir un vrai fichier
            hasAtLeastOneValidFile = true;

            ZipEntry ze = new ZipEntry(file);
            zos.putNextEntry(ze);

            try (FileInputStream in = new FileInputStream(srcFile)) {
                int len;
                while ((len = in.read(buffer)) > 0) {
                    zos.write(buffer, 0, len);
                }
            }

            zos.closeEntry();
        }

    } catch (IOException ex) {
        ex.printStackTrace();
    }

    // 4️⃣ Sécurité finale : supprimer le ZIP vide
    if (!hasAtLeastOneValidFile) {
        new File(zipFile).delete();
    }
}


private boolean zipContainsRealFile(File zipFile) {
    try (ZipFile zip = new ZipFile(zipFile)) {
        return zip.stream()
                  .anyMatch(entry ->
                          !entry.isDirectory() && entry.getSize() > 0
                  );
    } catch (IOException e) {
        return false;
    }
}


private boolean zipContainsRealFile(File zipFile) {
    try (ZipFile zip = new ZipFile(zipFile)) {
        Enumeration<? extends ZipEntry> entries = zip.entries();

        while (entries.hasMoreElements()) {
            ZipEntry entry = entries.nextElement();
            if (!entry.isDirectory() && entry.getSize() > 0) {
                return true;
            }
        }
        return false;
    } catch (IOException e) {
        return false;
    }
}
