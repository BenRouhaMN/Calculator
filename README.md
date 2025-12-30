private boolean zipContainsReadableFile(File zipFile) {
    try (java.util.zip.ZipFile zip = new java.util.zip.ZipFile(zipFile)) {

        Enumeration<? extends ZipEntry> entries = zip.entries();

        while (entries.hasMoreElements()) {
            ZipEntry entry = entries.nextElement();

            // ignorer les dossiers
            if (entry.isDirectory()) {
                continue;
            }

            // tenter de lire 1 octet réel
            try (InputStream is = zip.getInputStream(entry)) {
                if (is.read() != -1) {
                    return true; // ✅ contenu réel détecté
                }
            }
        }
        return false;

    } catch (IOException e) {
        // ZIP corrompu ou illisible
        return false;
    }
}
