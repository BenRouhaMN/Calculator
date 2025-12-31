private void zipInnerZipContent(File zipFile, ZipOutputStream zos) {

    byte[] buffer = new byte[4096];

    try (ZipFile zip = new ZipFile(zipFile)) {

        Enumeration<? extends ZipEntry> entries = zip.entries();

        while (entries.hasMoreElements()) {
            ZipEntry entry = entries.nextElement();

            // on ignore les dossiers
            if (entry.isDirectory()) {
                continue;
            }

            try (InputStream is = zip.getInputStream(entry)) {

                ZipEntry newEntry = new ZipEntry(entry.getName());
                zos.putNextEntry(newEntry);

                int len;
                while ((len = is.read(buffer)) > 0) {
                    zos.write(buffer, 0, len);
                }
                zos.closeEntry();
            }
        }

    } catch (IOException e) {
        // ZIP interne illisible → ignoré volontairement
        e.printStackTrace();
    }
}
