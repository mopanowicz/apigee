# Generator tokenów JWSID — kroki i komponenty do utworzenia w Apigee Edge UI

Poniżej znajduje się kompletna lista kroków i komponentów do utworzenia w **Apigee Edge UI** w odpowiedniej kolejności, wraz ze spójną konwencją nazewnictwa (standardowe prefiksy polityk Apigee: `JTP-`, `JWT-`, `KVM-`, `JS-`, `AM-`, `RF-`).

---

## Krok 1: Utworzenie zaszyfrowanej KVM (konfiguracja środowiska)

Zanim zbudujesz proxy, przygotuj bezpieczne miejsce na klucz prywatny RSA (format PKCS#8 PEM) używany do podpisywania JWSID.

* **Gdzie w UI:** `Admin → Environments → Key Value Maps` *(wybierz środowisko, np. `test`)* → **`+ Key Value Map`**
* **Nazwa KVM:** `jwsid-signing-keys`
* **Opcja:** zaznacz **`Encrypted`**
* **Wpisy (Key / Value):**
  * `private_key_pem` — treść klucza prywatnego RSA (PEM)
  * `active_kid` — identyfikator klucza w nagłówku JWS, np. `jwsid-key-2026-01`
  * *(opcjonalnie, dla łatwego przenoszenia między środowiskami)* `keycloak_jwks_uri` oraz `keycloak_issuer`

> **Uwaga dot. Apigee Edge UI:** w wielu wersjach Edge UI pozwala utworzyć zaszyfrowaną KVM, ale dodanie/aktualizację samych wpisów do **Encrypted KVM** wykonuje się przez **Management API** (REST) lub jednorazową politykę `KeyValueMapOperations` z operacją `<Put>`.

---

## Krok 2: Utworzenie API Proxy (No Target)

* **Gdzie w UI:** `Develop → API Proxies` → **`+ Proxy`**
* **Typ proxy:** **`No Target`** *(generator sam zwraca odpowiedź, nie łączy się z backendem)*
* **Proxy Name:** `jwsid-token-generator-v1`
* **Proxy Base Path:** `/v1/jwsid`
* **Common Policies:** `Pass through (none)`
* **Virtual Hosts:** `secure` *(HTTPS)*
* **Deploy:** wybierz docelowe środowisko (np. `test`) i kliknij **Create and Deploy**.

---

## Krok 3: Utworzenie Flowów w `Proxy Endpoints → default`

Otwórz proxy `jwsid-token-generator-v1`, przejdź do zakładki **Develop**, kliknij **`Proxy Endpoints → default`** i dodaj dwa Conditional Flows przyciskiem **`+`**:

1. **Flow główny (generowanie tokena):**
   * **Flow Name:** `GenerateToken`
   * **Condition:** `(proxy.pathsuffix MatchesPath "/token") and (request.verb = "POST")`
   * *(Docelowy endpoint: `POST https://<host-apigee>/v1/jwsid/token`)*
2. **Flow catch-all (nieznane ścieżki/metody — na końcu listy):**
   * **Flow Name:** `NotFound`
   * **Condition:** *(puste — dopasuje wszystko inne)*

---

## Krok 4: Dodanie zasobu skryptu JavaScript

* **Gdzie w UI:** w zakładce **Develop**, w lewym panelu **Navigator** przy sekcji **`Scripts`** kliknij **`+`** → **New Script** → typ **`JavaScript`**
* **Nazwa zasobu:** `prepare-jwsid-claims.js`
* **Zadanie:** parsowanie i walidacja JSON-a z body, usunięcie zastrzeżonych claimów (`iss`, `iat`, `exp`, `nbf`, `jti`, `client_id`), odczyt `client_id` / `azp` ze zweryfikowanego tokena Keycloaka oraz zapis wyniku do zmiennej `jwsid.claims`.

---

## Krok 5: Utworzenie i podpięcie Polityk (Policies)

W zakładce **Develop** dodaj poniższe polityki i podepnij je w podanej kolejności w Flow **`GenerateToken`**:

| Kolejność | Typ polityki w UI (`+ Step`) | Proponowana nazwa | Sekcja Flow | Rola |
| :---: | :--- | :--- | :--- | :--- |
| **1** | **JSON Threat Protection** | `JTP-Validate-Payload` | `GenerateToken` → **Request** | Ochrona przed uszkodzonym lub zbyt dużym JSON-em |
| **2** | **Verify JWT** | `JWT-Verify-Keycloak-Bearer` | `GenerateToken` → **Request** | Weryfikacja podpisu (JWKS Keycloaka), `exp` i `iss` tokena `Authorization: Bearer` |
| **3** | **Key Value Map Operations** | `KVM-Get-Signing-Key` | `GenerateToken` → **Request** | Odczyt `private_key_pem` (do zmiennej `private.jwsid.private_key`) i `active_kid` z `jwsid-signing-keys` |
| **4** | **JavaScript** | `JS-Prepare-JWSID-Claims` | `GenerateToken` → **Request** | Uruchamia `jsc://prepare-jwsid-claims.js` |
| **5** | **Generate JWT** *(lub **Generate JWS**)* | `JWT-Generate-JWSID` *(lub `JWS-Generate-JWSID`)* | `GenerateToken` → **Request** | Wygenerowanie i podpisanie tokena JWSID algorytmem `RS256` |
| **6** | **Assign Message** | `AM-Build-Token-Response` | `GenerateToken` → **Response** | Złożenie odpowiedzi HTTP `200 OK` (`{"token": "{jwsid.token}"}`) + nagłówek `Cache-Control: no-store` |
| **7** | **Raise Fault** | `RF-Not-Found` | `NotFound` → **Request** | Zwrócenie `404 Not Found` dla nieobsługiwanych metod/ścieżek |

---

## Krok 6: Polityki obsługi błędów (`FaultRules`)

W lewym panelu **Navigator → Policies** utwórz 3 polityki typu **Assign Message** *(w `FaultRules` używa się `AssignMessage`, a nie `RaiseFault`, aby uniknąć ponownego rzucenia błędu)* i podepnij je w XML `Proxy Endpoints → default` w sekcjach `<FaultRules>` oraz `<DefaultFaultRule>`:

| Typ polityki | Proponowana nazwa | Kiedy uruchamiana (Condition) | Odpowiedź HTTP |
| :--- | :--- | :--- | :--- |
| **Assign Message** | `AM-Fault-Unauthorized` | `jwt.JWT-Verify-Keycloak-Bearer.failed = true` | `401 Unauthorized` (niepoprawny/wygasły token Bearer) |
| **Assign Message** | `AM-Fault-BadRequest` | `(fault.name = "ScriptExecutionFailed") or (fault.name = "ExecutionFailed")` | `400 Bad Request` (błędny JSON lub brak `client_id`) |
| **Assign Message** | `AM-Fault-Default` | `<DefaultFaultRule>` (z `<AlwaysEnforce>true</AlwaysEnforce>`) | `500 Internal Server Error` (ogólny komunikat bez stack trace) |

---

## Krok 7: *(Opcjonalnie)* Komponenty w sekcji `Publish` (dodatkowy API Key)

Jeśli autoryzacja ma polegać **wyłącznie** na tokenie Bearer z Keycloaka, ten krok pomijasz. Jeśli chcesz dodatkowo wymagać klucza aplikacji Apigee i mieć raportowanie per aplikacja w analityce:
* **Polityka w `PreFlow`:** `VAK-Verify-ApiKey` (*Verify API Key*)
* **API Product** (`Publish → API Products`): `JWSID-Generator-Product` *(podpięte proxy `jwsid-token-generator-v1`)*
* **Developer** (`Publish → Developers`): np. `jwsid-consumers@twoja-domena.pl`
* **App** (`Publish → Apps`): np. `JWSID-Consumer-App` *(subskrybuje `JWSID-Generator-Product`)*

---

## Krok 8: Zapis, wdrożenie i weryfikacja w Trace

1. Kliknij **Save** (zapis rewizji) i sprawdź na pasku **Deployment**, że rewizja jest wdrożona na wybrane środowisko.
2. Przejdź do zakładki **Trace** → **Start Trace Session**.
3. Wyślij z Postmana żądanie `POST https://<host-apigee>/v1/jwsid/token` z nagłówkiem `Authorization: Bearer <token-z-keycloaka>` i przykładowym JSON-em w body.
4. W **Trace** zweryfikuj:
   * że `JWT-Verify-Keycloak-Bearer` odczytał `client_id` / `azp`,
   * że `private.jwsid.private_key` jest zamaskowane (ukryte przed podglądem),
   * że zwrócony token JWSID po zdekodowaniu zawiera wszystkie pola z JSON-a + `client_id` z tokena Bearer + właściwy `kid` w nagłówku.
