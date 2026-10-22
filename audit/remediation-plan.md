# Helyreállítási terv — apex-mcp

## Audit-azonosító

- **Termék:** apex-mcp
- **Audit dátuma:** 2026-08-13
- **Protokoll:** Enterprise Readiness Audit v2-EN
- **Feltételezések:** multi-tenant SaaS, nyilvános internetes kitettség, GDPR, APEX T-28 fenyegetés
- **Jelenlegi érettség:** 28,8%

---

## 1. Vérzéscsillapítás (azonnal — 0-7 nap)

| ID | Finding | Teendő | Elfogadási kritérium | Felelős | Effort |
|---|---|---|---|---|---|
| R-STB-001 | F-AM-P2c-001: HTTP/SSE auth hiánya | Bearer-token vagy mTLS hitelesítés bevezetése a HTTP/SSE transport rétegbe | Nem hitelesített kérés 401-et kap; tool hívás csak érvényes tokennel fut | Backend fejlesztő | M |
| R-STB-002 | F-AM-P2c-002: `apex_run_sql` tetszőleges SQL-t futtat | Olvass-only és író eszköz szétválasztása; DML/DDL alapból tiltva, külön privilege + megerősítés szükséges | `DROP TABLE`, `DELETE`, `UPDATE` alapból elutasítva; admin módban auditált | Backend fejlesztő / DBA | M |
| R-STB-003 | F-AM-P2d-001: hitelesítő adatok `.mcp.json`-ben | Titkoskezelő (OpenBao / OS keyring / short-lived token) integrálása; `.mcp.json` `.gitignore`-ba, csak `.example` marad | `grep -R ORACLE_DB_PASS .mcp.json` üres; szerver csak tokennel indul | DevOps / Backend | M |
| R-STB-004 | F-AM-P2c-005: single-user session singleton | HTTP mód esetén per-kapcsolat/session állapot; konkurens kliensek nem osztoznak ugyanazon `ImportSession`-ön | Két egyidejű `apex_create_app` külön app-et épít, nincs keresztszennyezés | Backend fejlesztő | L |

## 2. Gyors győzelmek (1-4 hét)

| ID | Finding | Teendő | Elfogadási kritérium | Felelős | Effort |
|---|---|---|---|---|---|
| R-QW-001 | F-AM-P2c-003: tool annotációk nem kényszerítenek megerősítést | Destruktív műveletekhez (`apex_delete_app`, `apex_delete_page`, stb.) server-side `confirm=true` + audit token kötelező | `confirm` nélküli törlés elutasítva; minden törlés audit logban rögzül | Backend fejlesztő | M |
| R-QW-002 | F-AM-P2c-006: `apex_export_app` tetszőleges útvonalra ír | Export útvonal allowlist / sandbox; `..` és abszolút path tiltva | `/etc/passwd` írása elutasítva; sandbox mappa működik | Backend fejlesztő | S |
| R-QW-003 | F-AM-P2d-002: gyenge titokmaszkolás | `apex_check_requirements` teljes titok elrejtése; logokban semmilyen secret nem szerepel | Rövid jelszó is `***`-ként jelenik meg; grep nem talál secret-et logban | Backend fejlesztő | S |
| R-QW-004 | F-AM-P4-001: lebegő dependency verziók | Lockfile (`uv.lock`, `poetry.lock`, `requirements-lock.txt`) generálása és commit; CI lockfile-ból telepít | CI reproducible build; `pip-audit` high/critical találat nélkül | DevOps | S |
| R-QW-005 | F-AM-P5-001: nincs CI minőségi kapu | GitHub Actions workflow: ruff, mypy, bandit, pytest `-m 'not integration'`, pip-audit; merge block | PR sikertelen bandit/mypy teszttel nem merge-ölhető | DevOps | M |
| R-QW-006 | F-AM-P6-002: nincs rate limiting | Per-IP és per-token rate limit; konkurens `apex_run_sql` korlátozása | 1000 kérés/1s után 429; Oracle connection count korlátos | Backend fejlesztő | S |

## 3. Középtáv (1-3 hónap)

| ID | Finding | Teendő | Elfogadási kritérium | Felelős | Effort |
|---|---|---|---|---|---|
| R-MT-001 | F-AM-P2c-004: generált PL/SQL string interpoláció | Azonosítók allowlist / `DBMS_ASSERT`; értékekhez bind változók; dry-run review lépés | Prompt-injection tábla név elutasítva; generált kód review-ben látható | Backend fejlesztő | L |
| R-MT-002 | F-AM-P1-001: nincs jogosultsági réteg | Több Oracle connection profil (read-only, dev, admin); tool dispatch előtt szerep ellenőrzés | Read-only token nem tud törölni vagy `apex_run_sql`-t írni | Architekt / Backend | L |
| R-MT-003 | F-AM-P3-001: GDPR hiányosságok | Jogalap, retention, törlés, export dokumentálása; `APEX_MCP_AUDIT_RETENTION_DAYS` + automatikis törlés; export API | Adatvédelmi szabályzat publikus; régi audit log automatikusan törlődik; export működik | Product / Jogi / Backend | L |
| R-MT-004 | F-AM-P6-001: helyi audit log | Audit események append-only külső rendszerbe (SIEM / syslog / OpenBao audit); integritás védelem | Host kompromittálódása után is elérhető audit trail; törlés alertet küld | DevOps / Backend | M |

## 4. Stratégiai (3-12 hónap)

| ID | Cél | Teendő | Elfogadási kritérium | Felelős | Effort |
|---|---|---|---|---|---|
| R-STR-001 | Enterprise sales readiness | SOC 2 / ISO 27001 igényekhez igazított kontrollok; penetration test | Külső pentest jelentés; kritikus/high találatok javítva | Security / Vezetés | XL |
| R-STR-002 | Multi-tenant SaaS üzemmód | Per-bérlő konfiguráció, izolált session-ök, skálázható deploy modell | Több bérlő egyetlen pool-ban, adat nem keveredik | Architekt / Backend | XL |
| R-STR-003 | Supply-chain hardening | Reproducible build, SBOM, aláírt artefaktok, dependency update policy | Minden release-hez SBOM és aláírás | DevOps | L |

## 5. Findingenkénti részletes lépések

### F-AM-P2c-001 — HTTP/SSE auth hiánya
1. Válassz auth modellt: bearer token (ajánlott) vagy mTLS.
2. Adj hozzá middleware-t a `mcp.run()` elé, amely ellenőrzi a tokent.
3. Konfigurálható legyen `MCP_AUTH_TOKEN` vagy hasonló env var.
4. Teszteld: érvénytelen token → 401; hiányzó token → 401; érvényes token → tool fut.
5. Dokumentáld a reverse-proxy integrációt.

### F-AM-P2c-002 — `apex_run_sql` tetszőleges SQL
1. Bontsd két tool-ra: `apex_run_select` (csak SELECT/WITH) és `apex_run_plsql` (gated).
2. Az író tool csak explicit engedélyezett műveleteket fogadjon el, vagy kérjen `role=admin` + `confirm`.
3. Minden írás legyen audit logban.
4. Teszteld a bypass kísérleteket: whitespace, commentek, `/*...*/SELECT`.

### F-AM-P2c-004 — generált PL/SQL interpoláció
1. `validate_table_name` kiterjesztése: csak ismert táblák / metaadatból kiolvasott lista.
2. Azonosítókat `DBMS_ASSERT` használatával kezeld (ahol Oracle elérhető), vagy allowlist.
3. Értékekhez kényszerítsd a bind változókat.
4. Dry-run módban a generált kód emberi review-ra kerülhessen.

### F-AM-P2d-001 — hitelesítő adatok `.mcp.json`-ben
1. Távolítsd el a példa konfigból a valós jelszavakat; kommentezd, hogy soha ne commitoljanak.
2. Implementálj OpenBao / OS keyring / cloud secrets manager olvasást.
3. Támogasd a short-lived tokeneket (pl. OIDC → Oracle wallet secret).
4. Adj hozzá `.gitignore` bejegyzést `.mcp.json`-re.

### F-AM-P3-001 — GDPR
1. Határozd meg az adatkezelési jogalapot és az adatkategóriákat.
2. Definiálj retention időt audit loghoz és session adatokhoz.
3. Implementálj törlési/export API-t bérlő/workspace szinten.
4. Készíts adatvédelmi tájékoztatót és processor listát.

## 6. Ellenőrzési terv

| Ellenőrzés | Módszer | Sikeres, ha |
|---|---|---|
| Auth működik | `curl` HTTP endpointre token nélkül és tokennel | 401 / sikeres tool válasz |
| SQL korlátozás | `apex_run_sql("DROP TABLE user_tables")` | elutasítva |
| Secret nem szivárog | `grep -R` secret kulcsokra repo + logokban | nincs találat |
| CI minőségi kapu | PR tesztelése hibás kóddal | nem merge-ölhető |
| GDPR export | workspace export meghívása | csak az adott workspace adatai visszaadva |
| Audit integritás | audit fájl törlése, SIEM ellenőrzése | események megmaradnak, alert érkezik |

## 7. Ütemterv

| Időszak | Cél |
|---|---|
| 0-7 nap | Vérzéscsillapítás: auth, SQL korlátozás, secret kezelés, session izoláció |
| 1-4 hét | Gyors győzelmek: confirm token, export sandbox, rate limit, lockfile, CI |
| 1-3 hónap | Középtáv: PL/SQL hardening, szerepkörök, GDPR, külső audit log |
| 3-12 hónap | Stratégiai: pentest, multi-tenant SaaS, SBOM, compliance certification |

## 8. Függőségek és kockázatok

- **Külső függőség:** `fastmcp`/`oracledb` sebezhetőségei — rendszeres `pip-audit` szükséges.
- **Szervezeti:** a csapat kicsi; több párhuzamos R tétel DevOps/Backend erőforrást igényel.
- **Runtime:** az ellenőrzésekhez élő Oracle ADB szükséges; integrációs teszteket be kell ütemezni.
