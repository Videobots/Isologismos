# Sub-skills & subagents — συμβάσεις (contracts)

> Κάθε κόμβος του workflow εκτελείται από μία «μονάδα» με σταθερή σύμβαση: **είσοδοι → εργασία → έξοδοι**. Όσες σημειώνονται **PLACEHOLDER** δεν υπάρχουν ακόμη ως αυτόνομα skills. Μέχρι να δημιουργηθούν (μόνο με έγκριση του χρήστη, βλ. `CLAUDE.md`), το master skill τις εκτελεί:
> - με **subagent** (εργαλείο Agent, `general-purpose`), με prompt την ενότητα της σύμβασης + τα `CLAUDE.md` κανόνες 1–4, ή
> - **inline**, όταν η εργασία είναι μικρή ή χρειάζεται διάλογο με τον χρήστη.
>
> Όταν κάποτε δημιουργηθεί πραγματικό skill, θα ζει στο `.claude/skills/[όνομα]/SKILL.md` και ο αντίστοιχος πίνακας του master SKILL.md θα αλλάξει από «Placeholder» σε «Υπάρχον skill».
>
> **Κοινός κανόνας εξόδου:** κάθε μονάδα επιστρέφει (α) αρχείο Markdown στο `nodes/` ή `raw/`, (β) σύνοψη ≤10 γραμμών, (γ) ένα block ` ```json ` με τα structured outputs που ορίζονται παρακάτω.

---

## ΦΑΣΗ 1

### `dd-erp-extractor` — PLACEHOLDER (κόμβος `DB1`)
- **Είσοδοι:** connector `erp` (SAP OData / Oracle Fusion REST), περίοδοι χρήσεων, εταιρικός κωδικός.
- **Εργασία:** ανάγνωση (read-only) ισοζυγίου, κινήσεων καθολικού, πελατών/προμηθευτών (ανοιχτά υπόλοιπα, ενηλικίωση), παγίων, αποθεμάτων, πωλήσεων ανά πελάτη/προϊόν.
- **Fallback:** εξαγωγή από τα PDF Γενικού & Αναλυτικού Καθολικού (μέσω `dd-document-extractor`). Σημείωση `mode: fallback`.
- **Έξοδος:** `raw/erp.json`: `{status, mode, period[], trial_balance[], ledger_entries[], ar_aging[], ap_aging[], fixed_assets[], inventory[], sales_by_customer[]}`.

### `dd-crm-extractor` — PLACEHOLDER (κόμβος `DB2`)
- **Είσοδοι:** connector `crm` (Salesforce REST/SOQL).
- **Εργασία:** λογαριασμοί πελατών, συμβόλαια, ευκαιρίες (pipeline), ιστορικό ανανεώσεων/αποχωρήσεων, ηλικία πελατολογίου.
- **Fallback:** CSV που θα στείλει η εταιρεία-στόχος (ζητείται στο `RECON_FIX` ή στην αρχή). Αλλιώς `missing`.
- **Έξοδος:** `raw/crm.json`: `{status, mode, accounts[], contracts[], churn_events[], pipeline[]}`. Ανωνυμοποίηση φυσικών προσώπων.

### `dd-bank-extractor` — PLACEHOLDER (κόμβος `API1`)
- **Είσοδοι:** connector `bank` (Open Banking / PSD2 Account Information Service, με συναίνεση της εταιρείας-στόχου).
- **Εργασία:** υπόλοιπα και κινήσεις ανά λογαριασμό, δάνεια, εγγυητικές επιστολές.
- **Fallback:** αντίγραφα κίνησης (PDF/CSV) και βεβαιώσεις υπολοίπων τραπεζών. Αλλιώς μόνο λογ. 38 από καθολικό (`mode: ledger-only`, ο έλεγχος R4 = Μ.Ε.).
- **Έξοδος:** `raw/bank.json`: `{status, mode, accounts[{masked_iban, balance_eoy, transactions[]}], loans[], guarantees[]}`. IBAN πάντα μασκαρισμένο (`GR**…1234`).

### `dd-document-extractor` — ΔΙΑΘΕΣΙΜΟ (κόμβος `DOC1`)
- **Είσοδοι:** PDF στον κεντρικό φάκελο (για την EUROSOS: Ισολογισμός, Γενικό & Αναλυτικό Καθολικό). Συμβάσεις από data room `data-room/[εταιρεία]/contracts/` ή υποφάκελο Drive (PLACEHOLDER, βλ. `connectors.md`).
- **Εργασία:** με το skill `anthropic-skills:pdf` (OCR αν χρειάζεται), εξαγωγή πινάκων οικονομικών καταστάσεων, προσαρτήματος, έκθεσης ελεγκτή, καθολικών. Για συμβάσεις: αντισυμβαλλόμενος, διάρκεια, αξία, ρήτρες.
- **Έξοδος:** `raw/docs.json`: `{status, financial_statements{bs, pl, notes, auditor_report}, gl[], sub_ledger[], contracts[{id, counterparty, type, value, term, clauses[], page_ref}]}`.

### `dd-etl` — PLACEHOLDER (κόμβος `ETL`)
- **Είσοδοι:** `raw/*.json`, `raw/adjustments.json` (αν υπάρχει).
- **Εργασία:** κανονικοποίηση σε ενιαίο σχήμα, αντιστοίχιση σε Ελληνικό Λογιστικό Σχέδιο / ΕΛΠ, ενοποίηση ταυτοτήτων πελατών ERP↔CRM, εφαρμογή manual adjustments (με ίχνος), εντοπισμός διπλοεγγραφών.
- **Έξοδος:** `data-lake/` (πριν το «πάγωμα»): `accounts.json` (λογαριασμός, περιγραφή, ομάδα ΕΛΠ, υπόλοιπο ανά χρήση, `source`, `source_ref`), `customers.json`, `cash.json`, `contracts.json`, `adjustments_applied.json`.

---

## ΦΑΣΗ 3

### `AN_FIN` → skill `/4-financial-analysis` — ΥΠΑΡΧΟΝ
- **Πρόσθετη σύμβαση στο workflow:** χωρίς αυτόνομη αποθήκευση Drive/PR. Γράφει `nodes/AN_FIN.md`.
- **Structured outputs:** `{equity, share_capital, working_capital, cfo, ebitda, ebit, interest, net_debt, current_ratio, quick_ratio, dso, dpo, dio, net_debt_to_ebitda, interest_cover, loss_years[], revenue_trend}`.

### `dd-operational-analysis` — PLACEHOLDER (κόμβος `AN_OP`)
- **Είσοδοι:** `data-lake/customers.json`, πωλήσεις/κόστος ανά πελάτη, CRM churn. Πλαίσιο: `outputs/[εταιρεία]-diagnostic-*.md` (`/7-business-diagnostic`), αν υπάρχει.
- **Εργασία:** μικτό περιθώριο ανά πελάτη/κατηγορία, ABC/Pareto πελατών, συγκέντρωση (top-1, top-5, top-10 %), churn rate, μέση διάρκεια σχέσης, cohort ανάλυση, όπου επιτρέπουν τα δεδομένα.
- **Structured outputs:** `{top1_share, top5_share, top10_share, churn_rate, top_customer_at_risk: bool, margin_by_customer[], data_gaps[]}`.

### `AN_RISK` → skill `/5-due-diligence` — ΥΠΑΡΧΟΝ + `dd-contract-review` — PLACEHOLDER
- `/5-due-diligence`: όπως ορίζει το SKILL.md του, χωρίς αυτόνομη αποθήκευση. Γράφει `nodes/AN_RISK_bs.md`.
- `dd-contract-review`: ανάγνωση `data-lake/contracts.json` + πρωτότυπων PDF. Ελέγχει change of control, εγγυήσεις/εμπράγματες ασφάλειες, covenants, αποκλειστικότητες, ποινικές ρήτρες, λήξεις ≤12 μηνών, εκκρεμοδικίες. Γράφει `nodes/AN_RISK_contracts.md`.
- **Structured outputs (κοινά):** `{red_flags[{rule: "F1".."F8", description, amount, source_ref, confidence}], provisions_gap, related_party_exposure, off_balance_items[], tax_social_arrears}`.

---

## ΦΑΣΗ 4

### `dd-valuation-dcf` — PLACEHOLDER (κόμβος `VAL_DCF`)
- **Είσοδοι:** `AN_FIN`, `AN_OP` outputs, ιστορικά FCF από data lake, **εγκεκριμένες από τον χρήστη** παραδοχές (business plan διοίκησης ή πίνακας παραδοχών με πηγή: ανάπτυξη, περιθώρια, CAPEX, ΚΚ, WACC [επιτόκιο χωρίς κίνδυνο, ασφάλιστρο κινδύνου χώρας/αγοράς, beta κλάδου], ρυθμός τελικής αξίας).
- **Εργασία:** πρόβλεψη 5 ετών, FCFF, τελική αξία (Gordon), Enterprise Value → Equity Value (− καθαρός δανεισμός − debt-like items από `AN_RISK`). Σενάρια χαμηλό / βασικό / υψηλό, πίνακας ευαισθησίας WACC × g.
- **Structured outputs:** `{ev_low, ev_base, ev_high, equity_low, equity_base, equity_high, wacc, g, debt_like_items[], assumptions_ref}`.

### `dd-valuation-liquidation` — PLACEHOLDER (κόμβος `VAL_LIQ`)
- **Είσοδοι:** ενεργητικό ανά κατηγορία από data lake, εμπράγματα βάρη (`AN_RISK`), **εγκεκριμένοι από τον χρήστη** συντελεστές ρευστοποίησης (haircuts) ανά κατηγορία, κόστος εκκαθάρισης, σειρά κατάταξης πιστωτών.
- **Εργασία:** orderly vs forced liquidation, καθαρή αξία ρευστοποίησης, ποσοστό ικανοποίησης ανά τάξη πιστωτών (εξασφαλισμένοι, προνομιούχοι [εργαζόμενοι, Δημόσιο, ΕΦΚΑ], ανέγγυοι), υπόλοιπο για μετόχους.
- **Structured outputs:** `{nlv_orderly, nlv_forced, recovery_by_class[], residual_to_equity, assumptions_ref}`.

---

## ΦΑΣΗ 5

### `dd-synthesis` — PLACEHOLDER (κόμβος `SYNTHESIS` / `REVISIONS`)
- **Είσοδοι:** όλα τα `nodes/*.md`, `state.json` (διαδρομή και αποφάσεις), `templates/final-report-template.md`. Σε re-prompt: + `DRAFT_v[n].md` + λίστα αλλαγών του Review Partner.
- **Εργασία:** σύνθεση χωρίς νέα στοιχεία, παραπομπές σε κόμβο/πηγή, executive summary, συνεπής αριθμητική, ενότητα περιορισμών. Σε re-prompt: εφαρμόζει **μόνο** τις ζητούμενες αλλαγές και επιστρέφει πίνακα «Σχόλιο → Τι άλλαξε → Ενότητα».
- **Έξοδος:** `nodes/DRAFT_v[n+1].md`.
