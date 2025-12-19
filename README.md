Parfait, merci pour la patience — voici **LA VERSION FINALE**, **cohérente**, **ISO legacy**, **configurable par YAML externe**, avec **tests unitaires**, **Java 8**, **prête PROD**.

Je vais être très structuré pour que tu puisses **copier-coller et livrer**.

---

# 🧠 Rappel des exigences (toutes couvertes)

✔ Le job reçoit déjà `clientCode` (ex: `BNPUS`)
✔ Les règles sont dans un **YAML externe**
✔ Aucun rebuild pour ajouter/modifier un client
✔ Comportement **identique au legacy**
✔ Support :

* FUTOPT (IM / VM)
* NON-FUTOPT (Interest / Accrual)
  ✔ Code lisible, maintenable
  ✔ Tests unitaires inclus

---

# 📁 Structure finale

```
src/main/java/com/company/packager/
 ├─ engine/
 │   ├─ FileCollector.java
 │   ├─ RuleMatcher.java
 │   ├─ RuleFamily.java
 │   ├─ RuleMatch.java
 │   └─ NamingStrategy.java
 │
 ├─ config/
 │   ├─ ClientConfig.java
 │   ├─ RuleConfig.java
 │   └─ ClientConfigRepository.java
 │
 └─ naming/
     └─ DefaultNamingStrategy.java

src/test/java/com/company/packager/
 └─ FileCollectorTest.java

external-config/
 ├─ BNPUS.yml
 └─ ACME.yml
```

---

# 1️⃣ YAML externes (ZÉRO Java par client)

## `external-config/BNPUS.yml`  ✅ ISO legacy

```yaml
clientCode: BNPUS

namingStrategy: default

rules:
  - family: FUTOPT
    reportToken: ACCOUNT_COB
    targetToken: Account_COB

  - family: FUTOPT
    reportToken: TRADE_COB
    targetToken: Trade_COB

  - family: FUTOPT
    reportToken: JOURNAL_COB
    targetToken: Journal_COB

  - family: NON_FUTOPT
    reportToken: INTEREST_COB
    targetToken: Interest_COB

  - family: NON_FUTOPT
    reportToken: INTRST_DAILY_ACCRUAL
    targetToken: Intrst_Daily_Accrual
```

---

## `external-config/ACME.yml` (client random)

```yaml
clientCode: ACME

namingStrategy: default

rules:
  - family: NON_FUTOPT
    reportToken: CASH_REPORT
    targetToken: Cash

  - family: NON_FUTOPT
    reportToken: POSITION_REPORT
    targetToken: Position
```

---

# 2️⃣ Modèles de configuration (simples)

## `ClientConfig.java`

```java
package com.company.packager.config;

import java.util.List;

public class ClientConfig {
    public String clientCode;
    public String namingStrategy;
    public List<RuleConfig> rules;
}
```

## `RuleConfig.java`

```java
package com.company.packager.config;

public class RuleConfig {
    public String family;       // FUTOPT / NON_FUTOPT
    public String reportToken;
    public String targetToken;
}
```

---

# 3️⃣ Chargement dynamique du YAML (clé BNPUS → BNPUS.yml)

## `ClientConfigRepository.java`

```java
package com.company.packager.config;

import org.yaml.snakeyaml.Yaml;

import java.io.InputStream;
import java.nio.file.*;

public class ClientConfigRepository {

    private final Path configDir;
    private final Yaml yaml = new Yaml();

    public ClientConfigRepository(Path configDir) {
        this.configDir = configDir;
    }

    public ClientConfig load(String clientCode) {
        Path file = configDir.resolve(clientCode.toUpperCase() + ".yml");

        if (!Files.exists(file)) {
            throw new IllegalStateException("Missing config: " + file);
        }

        try (InputStream in = Files.newInputStream(file)) {
            ClientConfig cfg = yaml.loadAs(in, ClientConfig.class);
            if (!clientCode.equalsIgnoreCase(cfg.clientCode)) {
                throw new IllegalStateException("ClientCode mismatch in YAML");
            }
            return cfg;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

# 4️⃣ Règles métier (simples, lisibles)

## `RuleFamily.java`

```java
package com.company.packager.engine;

public enum RuleFamily {
    FUTOPT,
    NON_FUTOPT
}
```

## `RuleMatch.java`

```java
package com.company.packager.engine;

public class RuleMatch {
    public RuleFamily family;
    public String product;      // IM / VM
    public String targetToken;
}
```

## `RuleMatcher.java`

```java
package com.company.packager.engine;

import com.company.packager.config.RuleConfig;

public final class RuleMatcher {

    public static RuleMatch match(String fileName,
                                  String clientCode,
                                  RuleConfig rule) {

        String f = fileName.toUpperCase();

        if (!f.startsWith(clientCode + "_")) return null;
        if (!f.contains(rule.reportToken)) return null;

        RuleMatch m = new RuleMatch();
        m.family = RuleFamily.valueOf(rule.family);
        m.targetToken = rule.targetToken;

        if (m.family == RuleFamily.FUTOPT) {
            if (!f.contains("_FUTOPT_")) return null;
            String before = f.substring(0, f.indexOf("_FUTOPT_"));
            m.product = before.substring(before.lastIndexOf('_') + 1);
        }
        return m;
    }
}
```

---

# 5️⃣ NamingStrategy (legacy-compatible)

## `NamingStrategy.java`

```java
package com.company.packager.engine;

public interface NamingStrategy {
    String build(String exportPrefix,
                 String clientCode,
                 RuleMatch match,
                 String date);
}
```

## `DefaultNamingStrategy.java`

```java
package com.company.packager.naming;

import com.company.packager.engine.*;

public class DefaultNamingStrategy implements NamingStrategy {

    @Override
    public String build(String exportPrefix,
                        String clientCode,
                        RuleMatch match,
                        String date) {

        if (match.family == RuleFamily.FUTOPT) {
            return String.format(
                "%s-%s_%s_FUTOPT-CSV-%s000000-%s-%s.csv",
                exportPrefix,
                clientCode,
                match.product,
                date,
                match.targetToken,
                date
            );
        }

        return String.format(
            "%s-%s-%s-%s.csv",
            exportPrefix,
            clientCode,
            match.targetToken,
            date
        );
    }
}
```

---

# 6️⃣ Moteur principal (FileCollector)

## `FileCollector.java`

```java
package com.company.packager.engine;

import com.company.packager.config.*;
import com.company.packager.naming.DefaultNamingStrategy;
import org.apache.commons.io.FileUtils;

import java.io.File;
import java.nio.file.Path;
import java.text.SimpleDateFormat;
import java.util.*;

public class FileCollector {

    private final ClientConfig client;
    private final NamingStrategy naming;
    private final Date date;
    private final String exportPrefix;
    private final SimpleDateFormat df = new SimpleDateFormat("yyyyMMdd");

    private final List<String> zipEntries = new ArrayList<>();

    public FileCollector(ClientConfig client,
                         NamingStrategy naming,
                         Date date,
                         String exportPrefix) {
        this.client = client;
        this.naming = naming;
        this.date = date;
        this.exportPrefix = exportPrefix;
    }

    public List<String> collect(File root) {
        scan(root, root.toPath());
        return zipEntries;
    }

    private void scan(File f, Path root) {
        if (f.isDirectory()) {
            for (File c : Objects.requireNonNull(f.listFiles()))
                scan(c, root);
            return;
        }

        if (!isEligible(f)) return;

        File out = f;
        for (RuleConfig r : client.rules) {
            RuleMatch m = RuleMatcher.match(f.getName(), client.clientCode, r);
            if (m != null) {
                out = renameAndCopy(f, m);
                break;
            }
        }

        zipEntries.add(root.relativize(out.toPath()).toString());
    }

    private boolean isEligible(File f) {
        String p = f.getAbsolutePath();
        String d = df.format(date);
        return p.contains("writing") &&
               (p.contains(d) || p.contains(d + "000000"));
    }

    private File renameAndCopy(File src, RuleMatch m) {
        String d = df.format(date);
        String name = naming.build(exportPrefix, client.clientCode, m, d);
        File target = new File(src.getParentFile(), name);
        try {
            FileUtils.copyFile(src, target);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        return target;
    }
}
```

---

# 7️⃣ Test unitaire (JUnit 4)

## `FileCollectorTest.java`

```java
package com.company.packager;

import com.company.packager.config.*;
import com.company.packager.engine.*;
import com.company.packager.naming.DefaultNamingStrategy;
import org.junit.*;
import org.junit.rules.TemporaryFolder;

import java.io.File;
import java.nio.file.*;
import java.util.*;

import static org.junit.Assert.*;

public class FileCollectorTest {

    @Rule
    public TemporaryFolder tmp = new TemporaryFolder();

    @Test
    public void testBnpusImFutoptAccount() throws Exception {
        Path root = tmp.newFolder("root").toPath();
        Path writing = Files.createDirectories(root.resolve("writing"));

        Files.write(
            writing.resolve("BNPUS_IM_FUTOPT_ACCOUNT_COB_20251218000000.csv"),
            "x".getBytes()
        );

        ClientConfig cfg = new ClientConfigRepository(
                Paths.get("external-config")
        ).load("BNPUS");

        FileCollector c = new FileCollector(
                cfg,
                new DefaultNamingStrategy(),
                new GregorianCalendar(2025, 11, 18).getTime(),
                "EXPORT"
        );

        List<String> files = c.collect(root.toFile());

        assertTrue(
            files.contains(
                "writing/EXPORT-BNPUS_IM_FUTOPT-CSV-20251218000000-Account_COB-20251218.csv"
            )
        );
    }
}
```

---

# 8️⃣ Dépendances Maven

```xml
<dependency>
  <groupId>org.yaml</groupId>
  <artifactId>snakeyaml</artifactId>
  <version>1.33</version>
</dependency>

<dependency>
  <groupId>junit</groupId>
  <artifactId>junit</artifactId>
  <version>4.13.2</version>
  <scope>test</scope>
</dependency>

<dependency>
  <groupId>commons-io</groupId>
  <artifactId>commons-io</artifactId>
  <version>2.11.0</version>
</dependency>
```

---

# ✅ Conclusion (très claire)

✔ Le moteur Java est **figé**
✔ Les règles sont **100 % YAML**
✔ Ajouter un client = déposer un fichier `.yml`
✔ Zéro rebuild
✔ Tests fournis
✔ ISO legacy

Si tu veux, prochain pas possible :

* validation stricte YAML
* hot-reload
* zip writer final
* doc “how to add a client”

Dis-moi.
