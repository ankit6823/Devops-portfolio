# Report Service Troubleshooting Summary — Preprod Environment

## Issue 1: `JRFontNotFoundException: Font "Arial" is not available to the JVM`

| Aspect | Details |
|---|---|
| **Error** | `net.sf.jasperreports.engine.util.JRFontNotFoundException: Font "Arial" is not available to the JVM` thrown during `generateDividendReport()` |
| **Root Cause** | JasperReports templates (`.jrxml`) referenced Microsoft fonts (Arial, Calibri) that were not installed on the Linux preprod server. Confirmed via `fc-list \| grep -i arial` returning nothing. Microsoft fonts are proprietary and not bundled with Linux distros by default. |
| **Diagnosis Steps** | 1. Checked `fc-list` for Arial — empty.<br>2. Checked if fontconfig itself had any fonts at all — it did (DejaVu, Nimbus, etc.), so only Arial specifically was missing.<br>3. Confirmed a licensed source existed: a Windows/Office machine (`admin1-ThinkCentre-M70s-Gen-3`) already had Arial (`msttcorefonts`) and Calibri installed. |
| **Fix Applied** | 1. Located Arial/Calibri `.ttf` files on the licensed Windows-fonts machine.<br>2. Copied (`scp`) the `.ttf` files to preprod (`/tmp` → `/usr/share/fonts/truetype/msttcorefonts/`).<br>3. Ran `fc-cache -f -v` to register the new fonts with fontconfig.<br>4. Verified with `fc-list \| grep -i arial` / `calibri`. |
| **For Missing Font Variants** | Calibri only had Regular/Bold available (no Italic/Bold-Italic). Installed **Carlito** (`fonts-crosextra-carlito`) — a metric-compatible open-source substitute for Calibri — via `apt-get install fonts-crosextra-carlito`, and used its italic/bold-italic files to fill the gap. |
| **JasperReports Config** | Created a `fonts.xml` + `jasperreports_extension.properties` on the app's classpath to explicitly map font family names ("Arial", "Calibri") to the actual `.ttf` file paths, using JasperReports' `SimpleFontExtensionsRegistryFactory`. Rebuilt and redeployed the app so the new resources were picked up. |
| **Key Learning** | Licensed/proprietary fonts must be sourced legitimately (from a licensed Windows/Office install, or Microsoft's official redistributable package via `ttf-mscorefonts-installer`) — they can't be downloaded from standard Linux repos. Open-source metric-compatible alternatives (Carlito for Calibri, Liberation Sans for Arial) are valid fallbacks when exact font files aren't available. |

---

## Issue 2: `dpkg was interrupted` blocking package installs

| Aspect | Details |
|---|---|
| **Error** | `E: dpkg was interrupted, you must manually run 'sudo dpkg --configure -a' to correct the problem` when running `apt-get install` |
| **Root Cause** | A previous package install/upgrade on the server was interrupted mid-process (e.g., terminated early), leaving dpkg's package database in an inconsistent state. |
| **Fix Applied** | Ran `sudo dpkg --configure -a` to let dpkg finish configuring the half-installed package(s) before retrying any `apt-get install` commands. |
| **Key Learning** | Always resolve dpkg state issues before troubleshooting "package not found" or install failures — an interrupted dpkg session blocks *all* subsequent package operations, not just the original one. |

---

## Issue 3: `FileNotFoundException` writing report PDF — `generateComprehensiveReport()`

| Aspect | Details |
|---|---|
| **Error** | `java.io.FileNotFoundException: /epsilonmoney/uploads/client/report/6575/report_1788346172378.pdf (No such file or directory)` |
| **Root Cause** | The target client-specific subdirectory (`6575`) did not exist, and the application's runtime user (`preprod`) did not have write permission on the parent `report/` directory to create it — the folder was owned by `root:root` with `755` permissions (no write access for other users). |
| **Diagnosis Steps** | 1. Confirmed `report/` folder existed but was empty.<br>2. Identified the actual service user by checking `systemctl show reporting.service -p User` → returned `preprod`.<br>3. Confirmed `ls -la` showed `root:root` ownership on the relevant directory. |
| **Fix Applied** | `sudo chown -R preprod:preprod /epsilonmoney/uploads/client/report/` and `chmod -R 755` to give the app's runtime user ownership/write access. Verified the fix by successfully running `mkdir` (no `sudo` required) as the `preprod` user afterward. |
| **Key Learning** | A service can only create files/folders within directories it has write permission on — read/execute access alone (via `755` for "other") isn't enough. Always match directory ownership to the actual user the service process runs as (found via `ps`, `systemctl show ... -p User`, or the systemd unit file). |

---

## Issue 4: Reporting service log file didn't match naming convention of other services

| Aspect | Details |
|---|---|
| **Symptom** | All other microservices produced logs named `EM<ServiceName>.log` (e.g. `EMLoginService.log`, `EMRefdataService.log`), but the reporting service produced `reporting.log` instead of `EMReportService.log` — inconsistent with the naming convention despite its config file being named `EMReportService_log4j.xml`. |
| **Root Cause** | The Log4j2 `RollingFile` appender inside `EMReportService_log4j.xml` had its `fileName` and `filePattern` hardcoded to `reporting.log` / `reporting-%d{yyyy-MM-dd}-%i.log.gz`, instead of following the `EM<ServiceName>` prefix convention used elsewhere. |
| **Diagnosis Steps** | 1. Found the reporting service is Log4j2-based (not Logback), via jar inspection (`unzip -l reporting.jar \| grep log4j`).<br>2. Found the active config location via `application.properties` → `logging.config=file:/epsilonmoney/logs2/EMReportService_log4j.xml`.<br>3. Compared file sizes/contents against other services' `*_log4j.xml` files. |
| **Fix Applied** | Edited `EMReportService_log4j.xml`, changing:<br>`fileName="/epsilonmoney/logs2/reporting.log"` → `fileName="/epsilonmoney/logs2/EMReportService.log"`<br>`filePattern="...reporting-%d{yyyy-MM-dd}-%i.log.gz"` → `filePattern="...EMReportService-%d{yyyy-MM-dd}-%i.log"`<br>Backed up the original config first, restarted `reporting.service`, and verified the new log file appeared correctly. |
| **Key Learning** | In Spring Boot + Log4j2 setups, `logging.config` in `application.properties` points to the actual active XML config — always check there first rather than assuming the config lives inside the jar or is named identically to the log output file. |

---

## Tools & Commands Used Throughout

| Purpose | Command |
|---|---|
| Check installed fonts | `fc-list \| grep -i <fontname>` |
| Refresh font cache | `sudo fc-cache -f -v` |
| Fix broken dpkg state | `sudo dpkg --configure -a` |
| Find running Java process/service | `ps -ef \| grep java`, `systemctl list-units --type=service` |
| Find which user runs a systemd service | `systemctl show <service> -p User -p Group` |
| Inspect jar contents without extracting | `unzip -l <file>.jar \| grep <keyword>` |
| Extract specific files from a jar | `unzip -o <file>.jar "path/inside/jar" -d <destination>` |
| Fix directory ownership/permissions | `sudo chown -R <user>:<group> <path>`, `chmod -R 755 <path>` |
| Restart a systemd service | `sudo systemctl restart <service>` |
| View a systemd unit's ExecStart/config | `systemctl cat <service>` |
