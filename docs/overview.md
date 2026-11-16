# Iterativní agent pro tvorbu dokumentů a kódu: cíl a dílčí problémy

*Stav: předběžný přehled z 16. 11. 2026. Všechny návrhy jsou předběžné, nic zatím není
ověřeno v provozu.*

## Cíl

Vybudovat opakovaně spouštěného agenta (princip jako dsh-monitor), jehož běhy nejsou
nezávislé, ale postupně budují jeden produkt (nejdřív dokumenty v md, později případně
i kód) po dobu jednotek měsíců s lidským zásahem v řádu minut týdně.

Kritéria úspěchu:
- nejvýše zhruba 15 min lidské práce týdně
- běh bez zásahu minimálně 1–2 týdny
- výsledek lze zkontrolovat ze stručné verze za několik minut

První případ: komplexní socialistický program pro dobu nástupu AI a polykrize,
včetně konkrétních kroků změny v rámci systému (dokument v md).

## Zadání

### Požadavky zadavatele
- Hlavní cíl definuje zadavatel (s chatbotem). Podcíle nižších úrovní a plán tvoří a aktualizuje agent.
- Stav a výsledky se na konci každého běhu ukládají do veřejného GitHub repa (nic tajného).
- Zásah zvenčí: kanál pro korekce směru a správa stavů, zvažují se GitHub issues.
- Role worker / planner / reviewer.
- Výstup ve více úrovních podrobnosti (podrobný a stručný).
- Zabránit pokusu vyřešit vše najednou.
- Free tier, 1–2 běhy denně (i víc), možnost více providerů (Gemini, Mistral…).
- Nejdřív dokumenty (md), kód později, případně v jiném repu.
- Primárně GitHub Actions, alternativa jen s podobnými možnostmi. DSH je výchozí harness, jiný je možný. Žádný lokální běh.

### Omezení prostředí (z README a workflow dsh-monitor)
- Gemini free tier cca 5 požadavků/min a 20/den (dle README, ověřit aktuální stav).
- Monitory běží sekvenčně, timeout monitoru 30 min, job 180 min.
- Agent nemá mezi běhy paměť, jen soubory v repu.
- Logy se nahrávají jako artefakt. U veřejného repa zkontrolovat, co obsahují.

## Přehled dílčích problémů

Stupnice stavu: nezpracováno → návrh → rozhodnuto → implementováno → ověřeno v provozu.

| ID | Dílčí problém | Stav |
|----|---------------|------|
| P1 | Rozklad na malé úkoly, prevence „vše najednou" | návrh |
| P2 | Reprezentace plánu a stavu | návrh |
| P3 | Role (worker/planner/reviewer) a jejich rotace | návrh |
| P4 | Zásahy zvenčí (issues, inbox/outbox, digest) | návrh |
| P5 | Kontrola kvality a kumulace chyb | návrh |
| P6 | Kapacita, kvóty, více providerů | návrh (čísla neověřena) |
| P7 | Integrita repa, souběh, validace změn | návrh |
| P8 | Víceúrovňový výstup | návrh |
| P9 | První případ: osnova, kritéria hotového, rubrika | nezpracováno |
| P10 | Ukončení, rozpočet, ochrana proti nekonečnému leštění | nezpracováno |
| P11 | Platforma a harness (GH Actions, alternativy, DSH) | nezpracováno |

## Detail dílčích problémů

### P1 Rozklad a prevence „vše najednou"
**Problém:** LLM má tendenci vyrobit celý výstup najednou a povrchně.
**Předběžný návrh:**
- Úkol vybírá deterministický runner, agentovi se předá jen ten úkol a jeho kontext.
- Jeden úkol = jeden výstupní soubor s kritérii hotového a limitem rozsahu.
- Dokument se skládá ze sekcí skriptem, ne generováním celku.
- Zpřesňování po vrstvách: osnova → poznámky → koncept → revize.
- Úkol uzavírá jen reviewer, worker ho nastaví na `review`.

**Otevřené:** velikost úkolu vzhledem ke kvótě LLM volání.

### P2 Plán a stav
**Předběžný návrh:** soubory v repu.
- `GOAL.md` (jen zadavatel), `tasks/T-nnn.md` (frontmatter: status, parent, depends_on, role, deliverable, acceptance)
- `state/STATE.md` (krátký, přepisovaný), `state/LOG.md` (append-only, agent čte jen konec)
- `research/` (zdroje s URL), `output/`

Plán po vlnách: detailně jen nejbližší úkoly, 2–3 úrovně.
**Otevřené:** kdo mění statusy (runner, nebo agent).

### P3 Role a rotace
**Předběžný návrh:** tatáž smyčka s různými prompty. Rotaci řídí runner:
- reviewer, pokud existují úkoly ve `review`
- planner, pokud je neprázdný inbox nebo uplynulo N běhů
- jinak worker

Planner a reviewer na silnějším modelu a méně často.
**Otevřené:** hodnota N.

### P4 Zásahy zvenčí
**Předběžný návrh:** GitHub Issues jako kanál, ale agent je nevidí přímo.
- Workflow před během zapíše issues do `inbox/` a po běhu pošle `outbox/` jako komentáře.
- Štítky: `steer`, `question`, `hold`.
- Týdenní digest jako jedno issue.
- Výchozí chování je pokračovat, blokuje jen úkol s otevřenou otázkou.
- Změny `GOAL.md` přes PR.

**Alternativa:** ručně commitovaný adresář korekcí.
**Otevřené:** dostupnost `gh` v kontejneru, práva `issues: write`.

### P5 Kvalita a kumulace chyb
**Předběžný návrh:**
- Reviewer s rubrikou: tvrzení odkazují na zdroje uložené v `research/`, rozlišení fakt / hodnota / prognóza, vnitřní konzistence. U kódu testy a lint.
- U sporných témat (první případ) navíc povinná sekce nejsilnějších námitek oponentů a výstup formulovaný jako argumentace pro daný směr, ne jako neutrální fakt. Jinak hrozí, že agent potvrzuje vlastní předpoklady.
- Reviewer může mít variantu „kritik".

**Otevřené:** jak měřit kvalitu bez čtení všeho, případně vzorkování zadavatelem.

### P6 Kapacita a providery
**Předběžný návrh:**
- Při limitu ~20 požadavků/den na Gemini počítat s 1–2 běhy denně a malými úkoly.
- Role rozdělit mezi providery (levnější model na worker, silnější na planner/reviewer).
- Fallback při 429.
- Spotřebu LLM volání na úkol měřit z existujícího `llm-requests.tsv`.

**Otevřené:** aktuální limity Gemini a Mistralu (ověřit), reálný počet úkolů za den.

### P7 Integrita repa
**Předběžný návrh:**
- `concurrency` group, `git pull --rebase` před pushem.
- Validace diffu po běhu: povolené cesty, limit počtu a velikosti souborů, jinak revert a status `failed`.
- Agent píše jen do cesty svého úkolu (pryč s přesouváním `*.md` z kořene).
- Commit po každém běhu.

**Otevřené:** řešení konfliktů s ručními commity zadavatele.

### P8 Víceúrovňový výstup
**Předběžný návrh:** podrobná verze je zdroj pravdy. `SUMMARY.md` a digest vznikají odvozeně samostatným úkolem, ne inkrementálně.
**Otevřené:** potřeba mezistupně (souhrny po kapitolách).

### P9 První případ
**Stav:** nezpracováno. Chybí osnova dokumentu, kritéria hotového, hranice rozsahu a rubrika (viz P5).

### P10 Ukončení a rozpočet
**Předběžný návrh:**
- Limit počtu běhů a úkolů.
- Konec, když jsou všechny úkoly `done` a reviewer schválil.
- Kill-switch (štítek `hold` nebo vypnutí cronu).
- Limit revizních kol na úkol proti nekonečnému leštění.

**Otevřené:** konkrétní hodnoty limitů.

### P11 Platforma a harness
**Stav:** nezpracováno.
- GH Actions: zdarma pro veřejná repa, cron je best-effort. Plánované workflow se u neaktivního repa vypínají (ověřit pravidla a započítávání bot commitů).
- Alternativy (např. GitLab CI, Forgejo/Codeberg) a jiný agentský harness než DSH nejsou prozkoumány.

## Další krok
1. Rozhodnout P9 (osnova prvního dokumentu a rubrika).
2. Změřit spotřebu LLM volání na úkol (P6).
3. Připravit kostru workflow, formát `tasks/*.md` a tři prompty (P1–P4, P7).