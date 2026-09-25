# Zapisane filtry list w projektach XAF

Ten dokument porównuje mechanizmy zapisanych filtrów w Fleetman, HIS, PathQ i DataDrive. Pokazuje, z jakich części składa się funkcja i gdzie projekty różnią się w zachowaniu. Nie wybiera jeszcze jednego wzorca do przeniesienia.

## Krótki obraz

We wszystkich czterech projektach zapisany filtr składa się z trzech elementów: encji w bazie, kontrolera `ListView` i tekstu kryterium XAF. Użytkownik zapisuje bieżące kryterium, a kontroler później parsuje je przez `ObjectSpace.ParseCriteria` i dodaje do `CollectionSource.Criteria`.

Różnice dotyczą dostępu do filtrów, pamięci podręcznej, sposobu odczytu aktywnego filtra, czyszczenia innych filtrów i zapisywania układu grida. HIS i PathQ mają blisko spokrewnione implementacje. DataDrive łączy część ich wzorców z rozwiązaniami rozwijanymi wcześniej w Fleetman.

## Porównanie

| Projekt | ORM encji | Przechowywanie i odświeżanie | Zakres filtra | Dodatkowe zachowanie |
|---|---|---|---|---|
| Fleetman | XPO | Encja `FilteringCriterion`; kontroler pobiera rekordy z `ObjectSpace` | Typ obiektu, właściciel, publiczność i role; brak `ViewId` w encji | Zapis i odtworzenie układu grida; akcja tworzenia reguły wyglądu |
| HIS | EF Core | Encja `FilteringCriteria` aktualizuje statyczny storage w `OnSaving`; kontroler czyta storage | Typ obiektu, `ViewId`, właściciel i publiczność | Filtry domyślne, czyszczenie filtrów, opcjonalne wskazanie filtra przez parametr URL |
| PathQ | EF Core | Ma tę samą encję i statyczny storage co HIS; `OnSaving` aktualizuje storage | Typ obiektu, `ViewId`, właściciel i publiczność | Dodaje akcję zapisu bieżącego filtra jako reguły wyglądu |
| DataDrive | EF Core | Kontroler pyta bazę przez `ObjectSpace`; odświeża listę po commicie i zmianach kolekcji | Typ obiektu, `ViewId`, właściciel i publiczność | Licznik rekordów, zarządzanie filtrami, układ DxGrid i czyszczenie filtra panelu AI |

## Wspólny przebieg

### 1. Encja przechowuje definicję filtra

Encja zapisuje kryterium XAF jako tekst, np. `Status = 2`. Przechowuje też typ obiektu i nazwę widoczną w akcji. Zależnie od projektu zapisuje właściciela, ustawienie publiczne, domyślność i identyfikator widoku.

W EF Core właściwość `ObjectType` jest zwykle `[NotMapped]`, a w bazie ląduje nazwa typu. Edytor kryterium otrzymuje typ przez `[CriteriaOptions(nameof(ObjectType))]`. Dzięki temu kreator kryteriów może podpowiadać pola właściwego obiektu.

Fleetman używa XPO i `TypeToStringConverter` dla właściwości `Type`. Encja ma też relację do ról, więc dostęp do filtra może ograniczać się do właściciela lub wskazanych ról.

### 2. Kontroler buduje pozycje akcji

Kontroler `ViewController<ListView>` tworzy zwykle `SingleChoiceAction` do wyboru filtra oraz akcje do zapisu i czyszczenia. W `OnActivated` lub `OnViewControlsCreated` pobiera filtry pasujące do widoku.

Typ filtra może obejmować typ pochodny. Projekty używają do tego sprawdzenia w rodzaju:

```csharp
criterion.ObjectType.IsAssignableFrom(View.ObjectTypeInfo.Type)
```

Jeśli encja przechowuje `ViewId`, kontroler może ograniczyć filtr do konkretnego widoku. To ważne, gdy ta sama klasa ma kilka list o różnym układzie lub znaczeniu.

### 3. Kontroler stosuje kryterium

Po wybraniu pozycji kontroler parsuje tekst przez `View.ObjectSpace.ParseCriteria(...)`. Następnie zapisuje wynik pod własnym kluczem w `View.CollectionSource.Criteria`. Edytor `ISupportFilter` może dostać ten sam tekst, by pole filtra grida pokazywało aktywne kryterium.

Każdy kontroler powinien używać własnego, stabilnego klucza kryterium. Czyszczenie klucza należącego do innego kontrolera może usunąć zabezpieczenie albo filtr aplikacyjny.

### 4. Kontroler zapisuje nowy filtr

Akcja odczytuje bieżące kryterium, tworzy `ObjectSpace` dla encji filtra i otwiera modalny `DetailView`. Użytkownik nadaje nazwę oraz wybiera widoczność lub domyślność. Po zatwierdzeniu kontroler zapisuje zmiany i odświeża pozycje akcji.

Źródło bieżącego filtra zależy od edytora. Warianty z Blazor sprawdzają `ISupportFilter`, a następnie `DxGridListEditor` i `GetFilterCriteria()`. DataDrive sprawdza też kryterium zapisane pod własnym kluczem `CollectionSource.Criteria`.

## Jak robi to każdy projekt

### Fleetman

Fleetman używa XPO-owej encji `FilteringCriterion` i kontrolera `CriteriaListViewController`. Filtry mogą być prywatne, publiczne albo dostępne dla wybranych ról. Kontroler dodatkowo chroni dostęp do listy wiadomości przez zachowanie osobnego kryterium bezpieczeństwa.

Encja ma `SaveViewLayout` i `ViewLayoutJson`. Po zaznaczeniu opcji zapisuje układ kolumn, sortowanie, grupowanie, szerokości i stan stronicowania. Wybranie filtra może ten układ przywrócić. Kontroler ma też osobną akcję, która tworzy regułę wyglądu na podstawie aktywnego kryterium.

Przy kopiowaniu trzeba uwzględnić różnicę ORM: encja i relacje są napisane dla XPO. Nie można przenieść ich bezpośrednio do projektu EF Core.

### HIS

HIS używa EF-owej encji `FilteringCriteria`, statycznej klasy `FilteringCriteriaStorage` i kontrolera `CriteriaListViewController`. Encja aktualizuje storage w `OnSaving`. Kontroler pobiera z niego rekordy pasujące do typu, `ViewId` i użytkownika.

W `AppConfigurationDetailViewController` znaleziono inicjalizację storage z rekordów bazy. Kontroler listy otwiera formularz zapisu, parsuje wybrany tekst kryterium i ma akcję czyszczenia. Obsługuje też parametr `filter` w adresie: może wybrać zapisany filtr lub filtr z modelu widoku.

Statyczny storage jest współdzielony przez proces. W Blazor Server trzeba ocenić jego zachowanie przy wielu użytkownikach i tenantach. Zmiana cache w `OnSaving` następuje przed potwierdzeniem commitu; błąd transakcji może więc zostawić pamięć w stanie niezgodnym z bazą.

### PathQ

PathQ zachowuje tę samą podstawę EF co HIS: encję `FilteringCriteria`, statyczny storage i kontroler `CriteriaListViewController`. Kontroler stosuje filtr jako `CollectionSource.Criteria` i synchronizuje tekst z edytorem.

W porównaniu z HIS wersja PathQ nie zawiera obsługi parametru URL z badanego kontrolera. Dodaje za to akcję, która tworzy `CustomApperance` z bieżącego filtra. Kod źródłowy modułu zawiera metodę inicjalizacji storage, ale wyszukiwanie w module nie znalazło miejsca, które ją wywołuje. Przed kopiowaniem trzeba potwierdzić, jak PathQ wypełnia storage przy starcie.

### DataDrive

DataDrive używa EF-owej encji `FilteringCriteria`, ale kontroler czyta dane bezpośrednio z bazy. Nie opiera się na statycznym storage. Odświeża pozycje po commicie i zmianach źródła kolekcji.

Kontroler obsługuje pozycję „Zarządzaj…”, ustawia właściciela i zakres widoku przy tworzeniu filtra oraz liczy rekordy dla pozycji akcji. Czyta aktywne kryterium z edytora, DxGrid lub własnego klucza `CollectionSource.Criteria`.

Przy czyszczeniu usuwa kryterium własne, pełnotekstowe i kryteria panelu AI z prefiksem `ContextualAI:`. Ten fragment jest specyficzny dla integracji DataDrive. Zanim skopiuje się go do innego projektu, trzeba znaleźć jego własne klucze filtrów i upewnić się, że nie usuwa kryteriów bezpieczeństwa.

DataDrive przechowuje też opcjonalny JSON układu DxGrid. W przeciwieństwie do pełnego przykładu XPO w Fleetman, encja EF przechowuje pola układu podzielone na DTO serializowane przez `System.Text.Json`.

## Różnice, które wpływają na wybór wzorca

### Cache albo zapytanie do bazy

HIS i PathQ używają statycznej listy w pamięci. To ogranicza liczbę zapytań, ale dodaje obowiązek inicjalizacji, synchronizacji po zapisie i kontroli zakresu tenantów. DataDrive czyta rekordy przez bieżący `ObjectSpace` i odświeża akcję po zmianach. To prostszy model spójności, ale wymaga sprawdzenia kosztu zapytania w konkretnym widoku.

### Właściciel, publiczność i role

HIS, PathQ i DataDrive wiążą filtr z użytkownikiem i flagą `AllowPublic`. Fleetman dodatkowo obsługuje role. Sama widoczność pozycji w akcji nie zastępuje uprawnień XAF do odczytu, edycji i usuwania encji.

### Widok a typ obiektu

Fleetman z badanego modelu dopasowuje filtr po typie obiektu. Pozostałe trzy projekty mają `ViewId`, co ogranicza filtr do konkretnej listy. Jeśli filtr zapisuje też układ grida, `ViewId` zapobiega użyciu go w niepasującym widoku.

### Czyszczenie filtrów

Lista może mieć kryteria z kilku źródeł: modelu XAF, zapisanego filtra, pełnotekstowej wyszukiwarki, filtrów kolumn i kontrolerów aplikacyjnych. Czyszczenie wszystkiego jest proste, ale może zdjąć ograniczenie, które musi pozostać. Najpierw trzeba zinwentaryzować klucze i właścicieli kryteriów.

### Kryteria zapisane jako tekst

Tekst może przestać działać po zmianie nazwy właściwości, relacji lub typu. Podczas ładowania trzeba obsłużyć błędne albo nieaktualne kryterium tak, by nie wywrócić całej listy. Dobrze też dać administratorowi informację, który zapisany filtr wymaga poprawy.

## Możliwe kierunki dalszej pracy

Po porównaniu można wybrać jeden z kierunków albo połączyć ich elementy:

1. **Wzorzec z HIS/PathQ:** EF Core i statyczny storage. Ma mało zapytań w czasie otwierania listy, ale wymaga pewnego startu cache i synchronizacji po commicie.
2. **Wzorzec z DataDrive:** EF Core i pobieranie filtrów z bieżącego `ObjectSpace`. Ułatwia spójność z bazą i filtrowanie według użytkownika, ale warto sprawdzić zapytania i odświeżanie listy.
3. **Rozszerzenie z Fleetman:** filtrowanie dostępu według ról oraz zapisywanie układu grida. Wymaga dopasowania do ORM i modelu uprawnień aplikacji.
4. **Nowszy wariant:** przechowywać definicje w bazie, filtrować je po typie, `ViewId`, użytkowniku, publiczności i tenantcie, a cache dodawać dopiero po pomiarach. Zmiany cache publikować po udanym commicie. Czyszczenie ma usuwać tylko jawnie wskazane klucze.

Punkt 4 jest propozycją do oceny, nie zachowaniem potwierdzonym w jednym z czterech projektów. Przed uznaniem go za wzorzec trzeba sprawdzić architekturę docelowej aplikacji, zwłaszcza multi-tenancy, uprawnienia encji i edytory wszystkich platform.

## Źródła w repozytoriach

- Fleetman: `Fleetman.Module/BusinessObjects/Common/FilteringCriterion.cs` oraz `Fleetman.Module/Controllers/ListControllers/FilterControllers/CriteriaListViewController.cs`.
- HIS: `HIS.Module/BusinessObjects/Helpers/FilteringCriteria.cs`, `HIS.Module/Storages/FilteringCriteriaStorage.cs`, `HIS.Module/Controllers/HelperControllers/CriteriaListViewController.cs` i `HIS.Module/Controllers/AppConfigurationDetailViewController.cs`.
- PathQ: analogiczne pliki w `HIS.Module`, z przestrzenią nazw `PathQ.Module`.
- DataDrive: `CS/DataDrive.Module/BusinessObjects/FilteringCriteria.cs` oraz `CS/DataDrive.Module/Features/CriteriaListViewController.cs`.
- Wspólny przewodnik DataDrive/HIS: `CS/docs/xaf-zapisywane-filtry-list-przewodnik.md`.
