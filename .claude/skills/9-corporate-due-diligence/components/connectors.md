# Εξωτερικά συστήματα & connectors

> Ορίζει πώς το `/9-corporate-due-diligence` συνδέεται με κάθε εξωτερικό σύστημα του workflow. Τα στοιχεία σύνδεσης (endpoints, IDs) μπαίνουν στο `connectors.example.json` → αντίγραφο `connectors.json` **εκτός repository**. **Κλειδιά, tokens και κωδικοί δεν αποθηκεύονται ποτέ σε αρχεία του repository.** Δίνονται μέσω ασφαλούς φόρμας διαπιστευτηρίων του περιβάλλοντος ή μέσω OAuth του αντίστοιχου MCP server.

## Καταστάσεις διαθεσιμότητας

| Κατάσταση | Σημασία |
|---|---|
| `live` | Ο connector απαντά και επιστρέφει δεδομένα για την εταιρεία-στόχο. |
| `fallback` | Δεν υπάρχει απευθείας σύνδεση, αλλά υπάρχει εναλλακτική πηγή (PDF, CSV, αντίγραφα). |
| `missing` | Ούτε σύνδεση ούτε εναλλακτική πηγή. Οι αντίστοιχοι έλεγχοι γίνονται «Μ.Ε.» και καταγράφονται ως περιορισμός εύρους. |

## Πίνακας connectors

| Κλειδί | Σύστημα / Κόμβος | Τρόπος σύνδεσης (προτεινόμενος) | Δικαιώματα | Κατάσταση σήμερα | Fallback |
|---|---|---|---|---|---|
| `erp` | SAP S/4HANA ή Oracle (`DB1`) | MCP server **PLACEHOLDER** `mcp-erp` πάνω σε SAP OData v4 ή Oracle Fusion REST API, με τεχνικό χρήστη μόνο για ανάγνωση | read-only | **Δεν υπάρχει** | PDF Γενικού & Αναλυτικού Καθολικού |
| `crm` | Salesforce (`DB2`) | MCP server **PLACEHOLDER** `mcp-salesforce` (REST/SOQL, Connected App με OAuth) | read-only | **Δεν υπάρχει** | CSV export από την εταιρεία-στόχο |
| `bank` | Τράπεζες (`API1`) | **PLACEHOLDER** Open Banking / PSD2 AIS API (μέσω αδειοδοτημένου παρόχου AISP, με συναίνεση της εταιρείας-στόχου) | read-only | **Δεν υπάρχει** | Αντίγραφα κίνησης PDF/CSV, βεβαιώσεις υπολοίπων |
| `pdf` | PDF οικονομικών καταστάσεων (`DOC1`) | Τοπικά αρχεία στον κεντρικό φάκελο + skill `anthropic-skills:pdf` | read | **Διαθέσιμο** (EUROSOS) | — |
| `data_room` | Συμβάσεις & έγγραφα (`DOC1`) | **PLACEHOLDER**: (α) φάκελος repository `data-room/[εταιρεία]/` ή (β) υποφάκελος Drive `Isologismos/data-room/[εταιρεία]/` (connector Google Drive) | read | **Δεν υπάρχει** | Ζήτηση εγγράφων στο `RECON_FIX` |
| `drive` | Google Drive «Isologismos» (`REVIEW_DEC`, `FINAL_REPORT`) | Connector Google Drive / Google Docs του λογαριασμού | read/write (μόνο νέα αρχεία) | **Διαθέσιμο** | Κανένα: υποχρεωτικό |
| `gmail` | Επικοινωνία με πελάτη (`RECON_FIX`) | Connector Gmail: **μόνο** `create_draft` | draft only | **Διαθέσιμο** | Κείμενο email στη συνομιλία για αντιγραφή |
| `github` | Αποθετήριο `videobots/isologismos` (`FINAL_REPORT`) | GitHub MCP / git | write (μόνο `outputs/`) | **Διαθέσιμο** | Κανένα: υποχρεωτικό |

## Health check (προέλεγχος, §0.4 του SKILL.md)

Για κάθε connector:
1. Υπάρχει ο MCP server / εργαλείο στη συνεδρία; (ToolSearch / ListConnectors). Αν όχι → δοκιμάζεις fallback.
2. Μία ελαφριά κλήση ανάγνωσης (π.χ. λίστα εταιρικών κωδικών, `SELECT Id FROM Account LIMIT 1`, λίστα λογαριασμών).
3. Καταγραφή στο `state.json → connectors.[κλειδί] = {status, mode, checked_at, note}`.

## Πώς θα ενεργοποιηθεί ένας placeholder (για μελλοντική υλοποίηση)

1. Ο χρήστης εξασφαλίζει από την εταιρεία-στόχο πρόσβαση μόνο για ανάγνωση (τεχνικός χρήστης / OAuth app / συναίνεση PSD2).
2. Προστίθεται ο MCP server στη ρύθμιση του περιβάλλοντος και το host του στη λίστα επιτρεπόμενων δικτύων. Τα κλειδιά δίνονται μέσω της ασφαλούς φόρμας διαπιστευτηρίων, **όχι** στη συνομιλία.
3. Συμπληρώνεται το αντίστοιχο block του `connectors.json` (από το `connectors.example.json`).
4. Ενημερώνεται ο πίνακας «Κατάσταση σήμερα» σε αυτό το αρχείο (αλλαγή αρχείου του Agent: χρειάζεται έγκριση του χρήστη).
