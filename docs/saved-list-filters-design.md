# Projekt: zapisane filtry list dla aplikacji XAF

## Cel

Zaprojektować wspólny, bezpieczny wzorzec zapisanych filtrów dla aplikacji XAF z EF Core. Użytkownik ma móc zapisać filtr z listy, wrócić do niego później, udostępnić go innym osobom i opcjonalnie ustawić jako domyślny.

Projekt korzysta z rozwiązań z DataDrive, HIS, PathQ i Fleetman. Punktem wyjścia jest model DataDrive: rekordy pozostają w bazie, a kontroler pobiera je przez `ObjectSpace`. Nie wprowadzamy statycznego cache, dopóki pomiary nie pokażą, że jest potrzebny.

## Założenia i granice

- Wzorzec dotyczy XAF z EF Core. Encję XPO z Fleetman należy przenieść osobno, jeśli aplikacja używa XPO.
- Filtry opisują tylko kryterium listy. Uprawnienia XAF nadal odpowiadają za dostęp do rekordów.
- Filtrowanie rekordów filtrów musi działać razem z izolacją tenantów aplikacji.
- Wersja podstawowa nie zapisuje układu kolumn, sortowania ani stanu grida. Można dodać tę funkcję później.
- Nie ustalamy nazw projektów, encji użytkownika, bazy ani tenantów. Przed implementacją trzeba je dopasować do aplikacji docelowej.

## Kryteria powodzenia

1. Użytkownik może zapisać aktywne kryterium z listy i zastosować je później.
2. Lista pokazuje wyłącznie filtry dostępne dla bieżącego użytkownika, typu, widoku i tenanta.
3. Filtr prywatny nie jest widoczny ani edytowalny przez innych użytkowników.
4. Filtr publiczny nie ujawnia danych spoza uprawnień użytkownika.
5. Błędne albo nieaktualne kryterium nie powoduje awarii całego widoku.
6. Wyczyszczenie filtra zapisanego nie usuwa filtrów bezpieczeństwa ani kryteriów innych funkcji.
7. Zapis lub edycja filtra odświeża akcję bez restartu aplikacji.
8. Rozwiązanie nie używa współdzielonej statycznej listy filtrów.

## Proponowane części

```text
EF Core encja SavedListFilter
        │
        ▼
SavedListFilterService ─── dostęp, zapis, walidacja i wybór domyślnego filtra
        │
        ▼
SavedListFilterListViewController ─── akcje XAF i ListView
        │
        ├── CollectionSource.Criteria[klucz kontrolera]
        └── ISupportFilter / API edytora dla widocznego kryterium
```

`SavedListFilterService` oddziela reguły dostępu i wyboru rekordów od kodu akcji. Kontroler odpowiada za cykl życia widoku i interakcję z edytorem. Encja przechowuje trwałą definicję filtra.

## Model danych

Encja `SavedListFilter` powinna dziedziczyć po EF-owym `BaseObject` i zawierać:

| Pole | Rola |
|---|---|
| `Name` | Nazwa pokazywana na liście filtrów; wymagana i ograniczona długością. |
| `ObjectTypeFullName` | Stabilizuje wybór typu w bazie jako tekst. |
| `ObjectType` | Właściwość niepersistowana do edytora kryterium; rozwiązywana przez `XafTypesInfo`. |
| `ViewId` | Ogranicza filtr do konkretnego ListView; puste może oznaczać wszystkie widoki tego typu, jeśli aplikacja tego chce. |
| `Criterion` | Tekst kryterium XAF; edytowany przez criteria editor dla `ObjectType`. |
| `Owner` | Użytkownik, który utworzył filtr. Null jest dozwolony wyłącznie dla publicznego filtra administracyjnego, jeśli taki wariant jest potrzebny. |
| `AllowPublic` | Określa, czy inni uprawnieni użytkownicy mogą użyć filtra. |
| `IsDefault` | Wskazuje filtr domyślny w zdefiniowanym zakresie. |
| `CreatedOn`, `ModifiedOn` | Opcjonalne pola audytowe, jeśli nie zapewnia ich wspólna encja bazowa lub mechanizm audytu. |

Jeśli aplikacja ma współdzieloną bazę dla wielu tenantów, encja lub zapytanie musi zawierać tenant scope. Jeśli każdy tenant ma osobną bazę i osobny `ObjectSpace`, nie dodawaj drugiego identyfikatora tenanta bez uzasadnienia. Potwierdź rzeczywisty model przed wyborem.

`ObjectType` powinien rozwiązywać typ po nazwie przez `XafTypesInfo.Instance.FindTypeInfo(...)`, a nie polegać wyłącznie na `Type.GetType(...)`. Refaktor namespace może pozostawić nierozwiązywalne stare wartości. Obsłuż taki rekord jako nieaktywny i pokaż administratorowi informację do naprawy.

## Zakres widoczności i uprawnienia

Serwis ma zwracać filtry zgodne ze wszystkimi poniższymi warunkami:

1. Ten sam tenant lub ten sam izolowany kontekst danych.
2. `ObjectType` pasuje do typu ListView. Filtr typu bazowego może pasować do typu pochodnego, jeśli to zachowanie zostanie zaakceptowane dla danej aplikacji.
3. `ViewId` jest zgodny z bieżącym widokiem albo pusty według ustalonej reguły.
4. Filtr jest publiczny albo jego `Owner` jest bieżącym użytkownikiem.
5. Użytkownik ma uprawnienia XAF do odczytu filtra.

Podstawowa wersja nie wprowadza dostępu według ról. Jeśli produkt tego wymaga, dodaj relację do ról i sprawdzaj ją w tym samym serwisie. Nie kopiuj automatycznie relacji XPO z Fleetman do encji EF Core.

Sam filtr nie nadaje praw do rekordów biznesowych. Widok i `ObjectSpace` muszą pozostać zabezpieczone przez XAF. Publiczny filtr może być używany przez różnych użytkowników, lecz każdy nadal widzi tylko rekordy, do których ma uprawnienia.

## Zachowanie domyślnego filtra

Ustal jedną, przewidywalną regułę:

- filtr domyślny bieżącego użytkownika ma pierwszeństwo przed publicznym filtrem domyślnym;
- filtr domyślny musi pasować do tenanta, typu i widoku;
- gdy nie ma filtra domyślnego, widok otwiera się bez zapisanego filtra;
- jeśli wykryto kilka domyślnych filtrów w tym samym zakresie, nie wybieraj losowo. Zastosuj brak filtra domyślnego i zgłoś błąd diagnostyczny administratorowi.

Wymuszaj najwyżej jeden filtr domyślny w zakresie `(tenant, user lub public, ObjectType, ViewId)`. Zrealizuj to w serwisie podczas zapisu. Unikalny indeks w bazie dodaj tylko wtedy, gdy da się go poprawnie wyrazić dla dostawcy bazy oraz wariantu publicznego i prywatnego.

## Przepływ użytkownika

### Otworzenie listy

Kontroler dla głównego `ListView` prosi serwis o dostępne filtry. Dodaje pozycje do `SingleChoiceAction`, a następnie wybiera domyślny filtr zgodnie z regułą powyżej. Akcje mają jawne podpisy, stabilne identyfikatory i działają wyłącznie w odpowiednim rodzaju widoku.

### Zapis filtra

1. Odczytaj kryterium z `ISupportFilter`.
2. Jeśli edytor nie udostępnia kryterium, użyj API konkretnego grid editora, tak jak robią to obecne implementacje DxGrid.
3. Jeśli tekstu nadal nie ma, zakończ akcję bez otwierania pustego formularza.
4. Parsuj kryterium dla typu bieżącego widoku. Pokaż czytelny komunikat, jeśli kryterium jest niepoprawne.
5. Utwórz encję w nowym `ObjectSpace`, ustaw typ, `ViewId`, właściciela i domyślną prywatność.
6. Otwórz modalny `DetailView` do nazwania i zapisania filtra.
7. Przy zatwierdzeniu sprawdź własność, zakres, unikalność domyślności i tenant. Następnie wykonaj commit.
8. Odśwież pozycje akcji po udanym commicie. Nie aktualizuj współdzielonego cache przed commitem.

Jeśli użytkownik zapisze publiczny filtr, sprawdź jawne uprawnienie do tworzenia filtrów publicznych. Nie traktuj samej wartości `AllowPublic` jako decyzji bezpieczeństwa.

### Zastosowanie filtra

1. Pobierz tekst kryterium z wybranego rekordu.
2. Ponownie sprawdź, że użytkownik może odczytać i zastosować ten rekord.
3. Rozwiąż typ i sparsuj kryterium. Obsłuż błędną składnię lub brak pola.
4. Ustaw je pod stałym kluczem należącym do kontrolera, np. `nameof(SavedListFilterListViewController)`.
5. Jeśli edytor wspiera tekst filtra, pokaż w nim zastosowane kryterium.
6. Odśwież widok.

Przy błędzie zachowaj pozostałe kryteria. Usuń tylko klucz kontrolera zapisanego filtra, wróć do pozycji „Wszystkie” i pokaż komunikat lub informację diagnostyczną z nazwą filtra.

### Czyszczenie

Akcja „Wyczyść filtry” usuwa tylko zapisany filtr oraz inne jawnie wskazane filtry użytkownika, których właściciele są znani. Przed dodaniem czyszczenia wyszukiwarki pełnotekstowej lub kryteriów z innych kontrolerów sprawdź ich klucze i kontrakty.

Nie usuwaj kryteriów bezpieczeństwa, ograniczeń tenantowych ani kryteriów systemowych. Nie polegaj na `View.Model.Filter = string.Empty` jako jedynym sposobie czyszczenia runtime criteria.

## `SavedListFilterService`

Serwis powinien obsługiwać co najmniej:

- pobranie dostępnych filtrów dla tenanta, użytkownika, typu i `ViewId`;
- sprawdzenie dostępu podczas odczytu oraz przed edycją lub usunięciem;
- utworzenie filtra z prywatnością domyślną;
- walidację tekstu kryterium dla wybranego typu;
- ustawienie jednego filtra domyślnego w danym zakresie;
- usunięcie filtra wyłącznie przez właściciela lub uprawnioną rolę administracyjną.

Serwis korzysta z `IObjectSpace` przekazanego przez kontroler albo sam tworzy zakres zgodnie z rejestracją DI projektu. Nie używa statycznych danych procesu. Nie przechowuje obiektów XAF z zamkniętego `ObjectSpace` jako długowiecznych obiektów cache.

## Obsługa zmian w modelu

Kryterium to tekst odwołujący się do nazw właściwości i relacji. Po refaktorze może przestać działać. W czasie budowania akcji parsuj dostępne kryteria, pomijaj błędne wpisy i rejestruj błąd wraz z identyfikatorem filtra. Nie ukrywaj go bez śladu.

Walidacja przy zapisie ogranicza liczbę błędnych rekordów, ale nie zastępuje walidacji przy użyciu. Kod może zmienić się po zapisaniu filtra.

## Decyzje do sprawdzenia w aplikacji docelowej

Przed implementacją odpowiedz na te pytania w kontekście konkretnego repozytorium:

1. Czy tenanci mają osobne bazy, osobne schematy czy wspólną bazę?
2. Czy typ użytkownika jest `ApplicationUser`, `Practitioner`, czy inny?
3. Czy filtry mają być dostępne tylko w jednym `ListView`, czy w każdym widoku tego typu?
4. Czy produkt wymaga filtrów publicznych, domyślnych i dostępu według ról?
5. Czy używane edytory Blazor i WinForms udostępniają kryterium przez ten sam interfejs?
6. Jakie klucze `CollectionSource.Criteria` należą do pozostałych kontrolerów?
7. Jak XAF role nadają uprawnienia do odczytu, edycji, tworzenia i usuwania nowej encji?

Jeśli odpowiedź zmienia zakres bezpieczeństwa lub izolację tenantów, rozstrzygnij ją przed implementacją.

## Plan wdrożenia

### Etap 1: rozpoznanie repozytorium

- Sprawdź wersję XAF i EF Core, platformy UI, model tenantów, klasę użytkownika i wzorce zabezpieczeń.
- Sprawdź kontrolery istniejących filtrów, edytory list oraz wszystkie używane klucze kryteriów.
- Potwierdź bieżące API DevExpress w dokumentacji dla wersji projektu.

### Etap 2: model i dostęp

- Dodaj encję EF i jej `DbSet` lub rejestrację zgodną z konwencją projektu.
- Dodaj migrację.
- Dodaj uprawnienia XAF dla encji oraz kontrolę dostępu do filtrów publicznych.
- Dodaj testy serwisu dla zakresu widoczności, własności i wyboru domyślnego filtra.

### Etap 3: kontroler i edytory

- Dodaj `SavedListFilterService` i kontroler `ListView`.
- Obsłuż `ISupportFilter`, a potem fallback dla konkretnych edytorów, które tego wymagają.
- Dodaj akcje wyboru, zapisu i czyszczenia.
- Zachowaj kryteria aplikacyjne i zabezpieczenia.

### Etap 4: weryfikacja zachowania

- Sprawdź filtr prywatny, publiczny, domyślny i przypisany do jednego `ViewId`.
- Sprawdź użytkowników o różnych rolach i tenantach.
- Sprawdź kryterium, które stało się niepoprawne po zmianie modelu.
- Sprawdź czyszczenie, gdy lista ma równocześnie filtry z innych kontrolerów.
- Sprawdź zapis i wybór na każdej wspieranej platformie UI.

## Poza pierwszą wersją

- Zapisywanie układu DxGrid, szerokości kolumn, sortowania, grupowania i paginacji.
- Współdzielenie filtra z wybranymi rolami.
- Statyczny cache lub rozproszony cache.
- Automatyczne migrowanie kryteriów po zmianie nazw właściwości.
- Tworzenie reguł wyglądu na podstawie zapisanych filtrów.

Dodaj te funkcje dopiero po potwierdzeniu potrzeby. Każda z nich rozszerza model dostępu lub cykl życia danych.

## Podstawa porównania

Opis opiera się na kodzie z czterech repozytoriów i na [porównaniu wzorców](saved-list-filters-comparison.md). Przykładowe ścieżki źródłowe:

- Fleetman: `Fleetman.Module/BusinessObjects/Common/FilteringCriterion.cs` i `Fleetman.Module/Controllers/ListControllers/FilterControllers/CriteriaListViewController.cs`.
- HIS: `HIS.Module/BusinessObjects/Helpers/FilteringCriteria.cs`, `HIS.Module/Storages/FilteringCriteriaStorage.cs` i `HIS.Module/Controllers/HelperControllers/CriteriaListViewController.cs`.
- PathQ: odpowiednie pliki w `HIS.Module`, z implementacją dla PathQ.
- DataDrive: `CS/DataDrive.Module/BusinessObjects/FilteringCriteria.cs` i `CS/DataDrive.Module/Features/CriteriaListViewController.cs`.

Szczegóły wywołań DevExpress trzeba potwierdzić dla wersji XAF używanej w repozytorium, które wdraża ten wzorzec.
