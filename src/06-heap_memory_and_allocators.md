# Pamięć sterty i alokatory

Wszystko, co do tej pory widzieliśmy, było ograniczone przez wymaganie z góry określonego rozmiaru. Tablice zawsze mają znaną w czasie kompilacji długość (w rzeczywistości długość jest częścią typu). Wszystkie nasze ciągi były literałami łańcuchowymi, które mają znaną w czasie kompilacji długość.

Co więcej, dwa rodzaje strategii zarządzania pamięcią, które widzieliśmy, dane globalne i stos wywołań, choć proste i wydajne, są ograniczające. Żadna z nich nie radzi sobie z danymi o dynamicznym rozmiarze i obie są sztywne w odniesieniu do czasu życia danych.

Ta część podzielona jest na dwa tematy. Pierwszym z nich jest ogólny przegląd naszego trzeciego obszaru pamięci, sterty. Drugi to proste, ale unikalne podejście Ziga do zarządzania pamięcią sterty. Nawet jeśli jesteś zaznajomiony z pamięcią sterty, powiedzmy z używania `malloc` w C, będziesz chciał przeczytać pierwszą część, ponieważ jest ona dość specyficzna dla Ziga.

## Sterta (heap)

Sterta jest trzecim i ostatnim obszarem pamięci do naszej dyspozycji. W porównaniu do danych globalnych i stosu wywołań, sterta to trochę dziki zachód: wszystko jest dozwolone. W szczególności, w ramach sterty możemy tworzyć pamięć w czasie wykonywania o znanym rozmiarze i mieć pełną kontrolę nad jej żywotnością.

Stos wywołań jest niesamowity ze względu na prosty i przewidywalny sposób zarządzania danymi (poprzez wypychanie i wyskakiwanie ramek stosu). Zaleta ta jest jednocześnie wadą: czas życia danych jest powiązany z ich miejscem na stosie wywołań. Sterta jest dokładnym przeciwieństwem. Nie ma wbudowanego cyklu życia, więc nasze dane mogą żyć tak długo lub tak krótko, jak to konieczne. Ta zaleta jest również jej wadą: nie ma wbudowanego cyklu życia, więc jeśli nie zwolnimy danych, nikt tego nie zrobi.

Spójrzmy na przykład:

```zig
const std = @import("std");

pub fn main() !void {
    // wkrótce porozmawiamy o alokatorach
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    // ** Następne dwie linie są najważniejsze **
    var arr = try allocator.alloc(usize, try getRandomCount());
    defer allocator.free(arr);

    for (0..arr.len) |i| {
        arr[i] = i;
    }
    std.debug.print("{any}\n", .{arr});
}

fn getRandomCount() !u8 {
    var seed: u64 = undefined;
    try std.posix.getrandom(std.mem.asBytes(&seed));
    var random = std.Random.DefaultPrng.init(seed);
    return random.random().uintAtMost(u8, 5) + 5;
}
```

Wkrótce zajmiemy się alokatorami Zig, na razie wiedz, że `allocator` to `std.mem.Allocator`. Używamy dwóch jego metod: `alloc` i `free`. Ponieważ wywołujemy `allocator.alloc` z `try`, wiemy, że może się to nie udać. Obecnie jedynym możliwym błędem jest `OutOfMemory`. Jego parametry w większości mówią nam, jak to działa: wymaga typu (T), a także liczby, a po powodzeniu zwraca wycinek []T. Ta alokacja odbywa się w czasie wykonywania - musi, nasza liczba jest znana tylko w czasie wykonywania.

Zgodnie z ogólną zasadą, każdy `alloc` będzie miał odpowiadający mu `free`. Tam gdzie `alloc` alokuje pamięć, `free` ją zwalnia. Nie pozwól, aby ten prosty kod ograniczał twoją wyobraźnię. Ten wzorzec `try alloc` + `defer free` jest powszechny i nie bez powodu: zwalnianie w pobliżu miejsca alokacji jest stosunkowo niezawodne. Ale równie powszechne jest przydzielanie w jednym miejscu i zwalnianie w innym. Jak powiedzieliśmy wcześniej, sterta nie ma wbudowanego zarządzania cyklem życia. Możesz przydzielić pamięć w programie obsługi HTTP i zwolnić ją w wątku tła, dwóch całkowicie oddzielnych częściach kodu.

## defer & errdefer

W ramach odskoczni, powyższy kod wprowadził nową funkcję językową: `defer`, która wykonuje dany kod lub blok po wyjściu z zakresu. "Wyjście z zakresu" obejmuje osiągnięcie końca zakresu lub powrót z zakresu. `defer` nie jest ściśle związany z alokatorami lub zarządzaniem pamięcią; można go użyć do wykonania dowolnego kodu. Ale powyższe użycie jest powszechne.

`defer` w Zig jest podobny do `defer` w Go, z jedną istotną różnicą. W Zig, `defer` zostanie uruchomiony na końcu zakresu zawierającego. W Go, `defer` jest uruchamiany na końcu funkcji zawierającej. Podejście Zig jest prawdopodobnie mniej zaskakujące, chyba że jesteś programistą Go.

Krewnym `defer` jest `errdefer`, który podobnie wykonuje dany kod lub blok na wyjściu z zakresu, ale tylko wtedy, gdy zwrócony zostanie błąd. Jest to przydatne podczas wykonywania bardziej złożonej konfiguracji i konieczności cofnięcia poprzedniej alokacji z powodu błędu.

Poniższy przykład stanowi skok w złożoność. Pokazuje zarówno `errdefer`, jak i powszechny wzorzec, w którym `init` alokuje, a `deinit` zwalnia:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

pub const Game = struct {
    players: []Player,
    history: []Move,
    allocator: Allocator,

    fn init(allocator: Allocator, player_count: usize) !Game {
        var players = try allocator.alloc(Player, player_count);
        errdefer allocator.free(players);

        // przechowaj 10 ostatnich ruchów dla każdego gracza
        var history = try allocator.alloc(Move, player_count * 10);

        return .{
            .players = players,
            .history = history,
            .allocator = allocator,
        };
    }

    fn deinit(game: Game) void {
        const allocator = game.allocator;
        allocator.free(game.players);
        allocator.free(game.history);
    }
};
```

Mam nadzieję, że podkreśla to dwie rzeczy. Po pierwsze, przydatność `errdefer`. W normalnych warunkach `players` są alokowani w `init` i zwalniani w `deinit`. Istnieje jednak przypadek brzegowy, gdy inicjalizacja `history` nie powiedzie się. W tym i tylko w tym przypadku musimy cofnąć alokację `players`.

Drugim godnym uwagi aspektem tego kodu jest to, że cykl życia naszych dwóch dynamicznie alokowanych wycinków, `players` i `history`, opiera się na logice naszej aplikacji. Nie ma reguły, która dyktowałaby, kiedy `deinit` musi zostać wywołany lub kto musi go wywołać. Jest to dobre, ponieważ daje nam arbitralne czasy życia, ale złe, ponieważ możemy to zepsuć, nigdy nie wywołując `deinit` lub wywołując go więcej niż raz.

> Nazwy init i `deinit` nie są niczym specjalnym. Są po prostu tym, czego używa standardowa biblioteka Zig i tym, co przyjęła społeczność. W niektórych przypadkach, w tym w bibliotece standardowej, używane są nazwy open i close lub inne bardziej odpowiednie nazwy.

## Podwójne zwolnienie i wycieki pamięci

Powyżej wspomniałem, że nie ma żadnych zasad określających, kiedy coś musi zostać zwolnione. Ale to nie do końca prawda, istnieje kilka ważnych zasad, po prostu nie są one egzekwowane, z wyjątkiem własnej skrupulatności.

Pierwszą zasadą jest to, że nie można zwolnić tej samej pamięci dwa razy.

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var arr = try allocator.alloc(usize, 4);
    allocator.free(arr);
    allocator.free(arr);

    std.debug.print("This won't get printed\n", .{});
}
```

Ostatnia linia tego kodu jest prorocza, _nie zostanie_ wypisana. Dzieje się tak, ponieważ dwukrotnie zwalniamy tę samą pamięć. Jest to znane jako podwójne zwolnienie i nie jest prawidłowe. Może się to wydawać dość proste do uniknięcia, ale w dużych projektach o złożonym czasie życia może być trudne do wyśledzenia.

Druga zasada mówi, że nie można zwalniać pamięci, do której nie mamy referencji. Może się to wydawać oczywiste, ale nie zawsze jest jasne, kto jest odpowiedzialny za jej zwolnienie. Poniższa instrukcja tworzy nowy ciąg znaków pisany małymi literami:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

fn allocLower(allocator: Allocator, str: []const u8) ![]const u8 {
    var dest = try allocator.alloc(u8, str.len);

    for (str, 0..) |c, i| {
        dest[i] = switch (c) {
            'A'...'Z' => c + 32,
            else => c,
        };
    }

    return dest;
}
```

Powyższy kod jest w porządku. Ale następujące użycie nie jest:

```zig
// Dla tego konkretnego kodu, powinniśmy byli użyć std.ascii.eqlIgnoreCase
fn isSpecial(allocator: Allocator, name: [] const u8) !bool {
    const lower = try allocLower(allocator, name);
    return std.mem.eql(u8, lower, "admin");
}
```

To jest wyciek pamięci. Pamięć utworzona w funkcji `allocLower` nigdy nie jest zwalniana. Nie tylko to, ale po zwróceniu `isSpecial` nigdy nie może zostać zwolniona. W językach z garbage collectorami, gdy dane stają się nieosiągalne, zostaną ostatecznie zwolnione przez garbage collector. Ale w powyższym kodzie, gdy zwraca `isSpecial`, tracimy nasze jedyne referencję do przydzielonej pamięci, zmiennej `lower`. Pamięć zniknie, dopóki nasz proces nie zakończy działania. Z naszej funkcji może wycieknąć tylko kilka bajtów, ale jeśli jest to długo działający proces i funkcja ta jest wywoływana wielokrotnie, będzie się to sumować i ostatecznie zabraknie nam pamięci.

Przynajmniej w przypadku podwójnego zwolnienia, otrzymamy twardy crash. Wycieki pamięci mogą być podstępne. Nie chodzi tylko o to, że główna przyczyna może być trudna do zidentyfikowania. Naprawdę małe wycieki lub wycieki w rzadko wykonywanym kodzie mogą być jeszcze trudniejsze do wykrycia. Jest to tak powszechny problem, że Zig zapewnia pomoc, którą zobaczymy podczas omawiania alokatorów.

## tworzenie i niszczenie

Metoda `alloc` z `std.mem.Allocator` zwraca wycinek o długości przekazanej jako drugi parametr. Jeśli chcesz uzyskać pojedynczą wartość, użyj `create` i `destroy` zamiast `alloc` i `free`. Kilka części temu, gdy uczyliśmy się o wskaźnikach, stworzyliśmy `User` i próbowaliśmy zwiększyć jego moc. Oto działająca wersja tego kodu oparta na stercie, wykorzystująca `create`:

```zig
const std = @import("std");

pub fn main() !void {
    // ponownie, wkrótce porozmawiamy o alokatorach!
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    // utworzenie User na stercie
    var user = try allocator.create(User);

    // zwolnienie pamięci przydzielonej dla user na końcu tego zakresu
    defer allocator.destroy(user);

    user.id = 1;
    user.power = 100;

    // ta linia została dodana
    levelUp(user);
    std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

fn levelUp(user: *User) void {
    user.power += 1;
}

pub const User = struct {
    id: u64,
    power: i32,
};
```

Metoda `create` przyjmuje pojedynczy parametr, typ (`T`). Zwraca wskaźnik do tego typu lub błąd, czyli `!*T`. Być może zastanawiasz się, co by się stało, gdybyśmy utworzyli naszego `user`, ale nie ustawili `id` i/lub `powe`. To tak, jakbyśmy ustawili te pola na `undefined`, a zachowanie jest, cóż, niezdefiniowane.

Kiedy badaliśmy zwisające wskaźniki, mieliśmy funkcję, która niepoprawnie zwracała adres lokalnego użytkownika:

```zig
pub const User = struct {
    fn init(id: u64, power: i32) *User{
        var user = User{
            .id = id,
            .power = power,
        };
        // to jest zwisający wskaźnik
        return &user;
    }
};
```

W tym przypadku bardziej sensowne byłoby zwrócenie `User`. Ale czasami _będziesz_ chciał, aby funkcja zwróciła wskaźnik do czegoś, co utworzyła. Zrobisz to, gdy chcesz, aby czas życia był wolny od sztywności stosu wywołań. Aby rozwiązać problem zwisającego wskaźnika powyżej, mogliśmy użyć `create`:

```zig
// nasz typ zwracany uległ zmianie, ponieważ init może teraz zawieść
// *User -> !*User
fn init(allocator: std.mem.Allocator, id: u64, power: i32) !*User{
    const user = try allocator.create(User);
    user.* = .{
        .id = id,
        .power = power,
    };
    return user;
}
```

Wprowadziłem nową składnię, `user.* = .{...}`. To trochę dziwne i nie podoba mi się to, ale zobaczysz to. Prawa strona jest czymś, co już widzieliście: jest to inicjalizator struktury z wywnioskowanym typem. Mogliśmy być jawni i użyć: `user.* = User{...}`. Lewa strona, `user.*`, jest sposobem na dereferencję wskaźnika. `&` pobiera `T` i daje nam `*T`. `.*` jest przeciwieństwem, zastosowane do wartości typu `*T` daje nam `T`. Pamiętaj, że `create` zwraca `!*User`, więc nasz użytkownik jest typu `*User`.

## Alokatory

Jedną z podstawowych zasad Ziga jest _brak ukrytych alokacji pamięci_. W zależności od twoje praktyki, może to nie brzmieć zbyt szczególnie. Jest to jednak ostry kontrast z tym, co można znaleźć w języku C, gdzie pamięć jest przydzielana za pomocą funkcji `malloc` biblioteki standardowej. W języku C, jeśli chcesz wiedzieć, czy funkcja alokuje pamięć, czy nie, musisz przeczytać źródło i poszukać wywołań funkcji `malloc`.

Zig nie posiada domyślnego alokatora. We wszystkich powyższych przykładach funkcje alokujące pamięć przyjmowały parametr `std.mem.Allocator`. Zgodnie z konwencją, jest to zwykle pierwszy parametr. Cała standardowa biblioteka Ziga i większość bibliotek innych firm wymaga, aby wywołujący podał alokator, jeśli zamierza zaalokować pamięć.

Ta jawność może przybrać jedną z dwóch form. W prostych przypadkach, alokator jest dostarczany przy każdym wywołaniu funkcji. Istnieje wiele takich przykładów, ale `std.fmt.allocPrint` jest jednym z nich, który prawdopodobnie będzie potrzebny prędzej czy później. Jest on podobny do `std.debug.print`, którego używaliśmy, ale alokuje i zwraca ciąg znaków zamiast zapisywać go do `stderr`:

```zig
const say = std.fmt.allocPrint(allocator, "It's over {d}!!!", .{user.power});
defer allocator.free(say);
```

Inną formą jest sytuacja, w której alokator jest przekazywany do `init`, a następnie używany wewnętrznie przez obiekt. Widzieliśmy to powyżej z naszą strukturą `Game`. Jest to mniej jednoznaczne, ponieważ przekazałeś obiektowi alokator do użycia, ale nie wiesz, które wywołania metod faktycznie dokonają alokacji. Podejście to jest bardziej praktyczne w przypadku długotrwałych obiektów.

Zaletą wstrzykiwania alokatora jest nie tylko jawność, ale także elastyczność. `std.mem.Allocator` to interfejs, który udostępnia funkcje `alloc`, free, `create` i `destroy`, a także kilka innych. Do tej pory widzieliśmy tylko `std.heap.GeneralPurposeAllocator`, ale inne implementacje są dostępne w bibliotece standardowej lub jako biblioteki stron trzecich.

> Zig nie ma ładnego cukru składniowego do tworzenia interfejsów. Jednym ze wzorców zachowania podobnego do interfejsu są unie tagowane, choć jest to stosunkowo ograniczone w porównaniu do prawdziwych interfejsów. Inne wzorce pojawiły się i są używane w całej bibliotece standardowej, na przykład w `std.mem.Allocator`. Jeśli jesteś zainteresowany, napisałem [osobny wpis na blogu wyjaśniający interfejsy](https://www.openmymind.net/Zig-Interfaces/).

Jeśli tworzysz bibliotekę, najlepiej jest zaakceptować `std.mem.Allocator` i pozwolić użytkownikom biblioteki zdecydować, której implementacji alokatora użyć. W przeciwnym razie będziesz musiał wybrać odpowiedni alokator, a jak zobaczymy, nie wykluczają się one wzajemnie. Mogą istnieć dobre powody do tworzenia różnych alokatorów w programie.

## Alokator ogólnego przeznaczenia (GeneralPurposeAllocator)

Jak sama nazwa wskazuje, `std.heap.GeneralPurposeAllocator` jest `alokatorem` "ogólnego przeznaczenia", bezpiecznym dla wątków, który może służyć jako główny alokator aplikacji. Dla wielu programów będzie to jedyny potrzebny alokator. Podczas uruchamiania programu, alokator jest tworzony i przekazywany do funkcji, które go potrzebują. Przykładowy kod z mojej biblioteki serwera HTTP jest dobrym przykładem:

```zig
const std = @import("std");
const httpz = @import("httpz");

pub fn main() !void {
    // utworzenie naszego alokatora ogólnego przeznaczenia
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};

    // pobieramy z niego std.mem.Allocator
    const allocator = gpa.allocator();

    // przekazujemy nasz alokator do funkcji i bibliotek, które go wymagają
    var server = try httpz.Server().init(allocator, .{.port = 5882});

    var router = server.router();
    router.get("/api/user/:id", getUser);

    // blokuje bieżący wątek
    try server.listen();
}
```

Tworzymy alokator `GeneralPurposeAllocator`, pobieramy z niego `std.mem.Allocator` i przekazujemy go do funkcji `init` serwera HTTP. W bardziej złożonym projekcie `alocator` byłby przekazywany do wielu części kodu, z których każda mogłaby przekazywać go do własnych funkcji, obiektów i zależności.

Można zauważyć, że składnia wokół tworzenia `gpa` jest nieco dziwna. Co to jest: `GeneralPurposeAllocator(.{}){}`? To wszystko rzeczy, które widzieliśmy już wcześniej, tylko rozbite razem. `std.heap.GeneralPurposeAllocator` jest funkcją, a ponieważ używa `PascalCase`, wiemy, że zwraca typ. (Porozmawiamy więcej o generykach w następnej części). Wiedząc, że zwraca typ, być może ta bardziej jawna wersja będzie łatwiejsza do rozszyfrowania:

```zig
const T = std.heap.GeneralPurposeAllocator(.{});
var gpa = T{};

// to to samo co:

var gpa = std.heap.GeneralPurposeAllocator(.{}){};
```

Być może nadal nie jesteś pewien znaczenia `.{}`. Jest to również coś, co widzieliśmy wcześniej: jest to inicjalizator struktury z niejawnym typem. Jaki jest typ i gdzie znajdują się pola? Typem jest` std.heap.general_purpose_allocator.Config`, chociaż nie jest on bezpośrednio eksponowany w ten sposób, co jest jednym z powodów, dla których nie jesteśmy jednoznaczni. Żadne pola nie są ustawione, ponieważ struktura `Config` definiuje wartości domyślne, których będziemy używać. Jest to powszechny wzorzec konfiguracji / opcji. W rzeczywistości widzimy to ponownie kilka linijek niżej, kiedy przekazujemy `.{.port = 5882}` do `init`. W tym przypadku używamy wartości domyślnej dla wszystkich pól z wyjątkiem jednego - portu.

## std.testing.allocator

Mam nadzieję, że byłeś wystarczająco zaniepokojony, gdy rozmawialiśmy o wyciekach pamięci, a następnie chętny, aby dowiedzieć się więcej, gdy wspomniałem, że Zig może pomóc. Pomoc ta pochodzi z `std.testing.allocator`, który jest `std.mem.Allocator`. Obecnie jest on zaimplementowany przy użyciu `GeneralPurposeAllocator` z dodatkową integracją z runnerem testowym Ziga, ale to szczegół implementacji. Ważną rzeczą jest to, że jeśli użyjemy `std.testing.allocator` w naszych testach, możemy wychwycić większość wycieków pamięci.

Prawdopodobnie znasz już tablice dynamiczne, często nazywane ArrayListami. W wielu językach programowania dynamicznego wszystkie tablice są tablicami dynamicznymi. Tablice dynamiczne obsługują zmienną liczbę elementów. Zig ma odpowiednią ogólną ArrayListę, ale stworzymy ją specjalnie do przechowywania liczb całkowitych i zademonstrowania wykrywania wycieków:

```zig
pub const IntList = struct {
    pos: usize,
    items: []i64,
    allocator: Allocator,

    fn init(allocator: Allocator) !IntList {
        return .{
            .pos = 0,
            .allocator = allocator,
            .items = try allocator.alloc(i64, 4),
        };
    }

    fn deinit(self: IntList) void {
        self.allocator.free(self.items);
    }

    fn add(self: *IntList, value: i64) !void {
        const pos = self.pos;
        const len = self.items.len;

        if (pos == len) {
            // zabrakło nam miejsca
            // utwórz nowy wycinek, który jest dwa razy większy
            var larger = try self.allocator.alloc(i64, len * 2);

            // skopiuj elementy, które wcześniej dodaliśmy do naszej nowej przestrzeni
            @memcpy(larger[0..len], self.items);

            self.items = larger;
        }

        self.items[pos] = value;
        self.pos = pos + 1;
    }
};
```

Interesująca część dzieje się w `add`, gdy `pos == len`, wskazując, że zapełniliśmy naszą obecną tablicę i musimy utworzyć większą. Możemy użyć `IntList` w następujący sposób:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var list = try IntList.init(allocator);
    defer list.deinit();

    for (0..10) |i| {
        try list.add(@intCast(i));
    }

    std.debug.print("{any}\n", .{list.items[0..list.pos]});
}
```

Kod działa i drukuje prawidłowy wynik. Jednakże, mimo że _wywołaliśmy_ `deinit` na liście, wystąpił wyciek pamięci. Nic się nie stało, że tego nie zauważyłeś, ponieważ zamierzamy napisać test i użyć `std.testing.allocator`:

```zig
const testing = std.testing;
test "IntList: add" {
    // Używamy tutaj testing.allocator!
    var list = try IntList.init(testing.allocator);
    defer list.deinit();

    for (0..5) |i| {
        try list.add(@intCast(i+10));
    }

    try testing.expectEqual(@as(usize, 5), list.pos);
    try testing.expectEqual(@as(i64, 10), list.items[0]);
    try testing.expectEqual(@as(i64, 11), list.items[1]);
    try testing.expectEqual(@as(i64, 12), list.items[2]);
    try testing.expectEqual(@as(i64, 13), list.items[3]);
    try testing.expectEqual(@as(i64, 14), list.items[4]);
}
```

> `@as` to wbudowana funkcja, która wykonuje wymuszanie typów. Jeśli zastanawiasz się, dlaczego nasz test musiał użyć tak wielu z nich, nie jesteś jedyny. Technicznie rzecz biorąc, dzieje się tak dlatego, że drugi parametr, "rzeczywisty", jest wymuszany na pierwszym, "oczekiwanym". W powyższym przykładzie wszystkie "oczekiwane" to `comptime_int`, co powoduje problemy. Wiele osób, w tym ja, uważa to za [dziwne i niefortunne zachowanie](https://github.com/ziglang/zig/issues/4437).

Jeśli jesteś na bieżąco, umieść test w tym samym pliku co `IntList` i `main`. Testy Ziga są zwykle pisane w tym samym pliku, często w pobliżu kodu, który testują. Kiedy używamy `zig test learning.zig` do uruchomienia naszego testu, otrzymujemy niesamowitą porażkę:

```
Test [1/1] test.IntList: add... [gpa] (err): memory address 0x101154000 leaked:
/code/zig/learning.zig:26:32: 0x100f707b7 in init (test)
   .items = try allocator.alloc(i64, 2),
                               ^
/code/zig/learning.zig:55:29: 0x100f711df in test.IntList: add (test)
 var list = try IntList.init(testing.allocator);

... MORE STACK INFO ...

[gpa] (err): memory address 0x101184000 leaked:
/code/test/learning.zig:40:41: 0x100f70c73 in add (test)
   var larger = try self.allocator.alloc(i64, len * 2);
                                        ^
/code/test/learning.zig:59:15: 0x100f7130f in test.IntList: add (test)
  try list.add(@intCast(i+10));
```

Mamy wiele wycieków pamięci. Na szczęście alokator testowy mówi nam dokładnie, gdzie przydzielono wyciekającą pamięć. Czy jesteś teraz w stanie zauważyć wyciek? Jeśli nie, pamiętaj, że generalnie każda `aloc` powinna mieć odpowiadające jej zwolnienie. Nasz kod wywołuje `free` raz, w `deinit`. Jednak `alloc` jest wywoływane raz w `init`, a następnie za każdym razem, gdy wywoływane jest `add` i potrzebujemy więcej miejsca. Za każdym razem, gdy `alloc`ujemy więcej miejsca, musimy `free` poprzednie `self.items`:

```zig
// istniejący kod
var larger = try self.allocator.alloc(i64, len * 2);
@memcpy(larger[0..len], self.items);

// Dodany kod
// zwolnienie poprzedniej alokacji
self.allocator.free(self.items);
```

Dodanie tej ostatniej linii, po skopiowaniu elementów do naszego wycinka `larger`, rozwiązuje problem. Jeśli uruchomisz `zig test learning.zig`, nie powinien pojawić się żaden błąd.

## ArenaAllocator

Alokator GeneralPurposeAllocator jest rozsądnym domyślnym rozwiązaniem, ponieważ działa dobrze we wszystkich możliwych przypadkach. Jednak w ramach programu można napotkać wzorce alokacji, które mogą skorzystać z bardziej wyspecjalizowanych alokatorów. Jednym z przykładów jest potrzeba krótkotrwałego stanu, który można wyrzucić po zakończeniu przetwarzania. Parsery często mają takie wymagania. Szkieletowa funkcja `parse` może wyglądać następująco:

```zig
fn parse(allocator: Allocator, input: []const u8) !Something {
    const state = State{
        .buf = try allocator.alloc(u8, 512),
        .nesting = try allocator.alloc(NestType, 10),
    };
    defer allocator.free(state.buf);
    defer allocator.free(state.nesting);

    return parseInternal(allocator, state, input);
}
```

Chociaż nie jest to zbyt trudne w zarządzaniu, `parseInternal` może potrzebować innych krótkotrwałych alokacji, które będą musiały zostać zwolnione. Alternatywnie możemy utworzyć ArenaAllocator, który pozwoli nam zwolnić wszystkie alokacje za jednym razem:

```zig
fn parse(allocator: Allocator, input: []const u8) !Something {
    // utworzenie ArenaAllocator z dostarczonego alokatora
    var arena = std.heap.ArenaAllocator.init(allocator);

   // spowoduje to zwolnienie wszystkiego, co zostało utworzone z tej areny
    defer arena.deinit();

    // utworzenie std.mem.Allocator z areny, będzie to
    // alokator, którego użyjemy wewnętrznie
    const aa = arena.allocator();

    const state = State{
        // używamy tutaj aa!
        .buf = try aa.alloc(u8, 512),

        // używamy tutaj aa!
        .nesting = try aa.alloc(NestType, 10),
    };

    // przekazujemy tutaj aa, więc mamy gwarancję, że
    // każda inna alokacja będzie w naszej arenie
    return parseInternal(aa, state, input);
}
```

`ArenaAllocator` pobiera alokator potomny, w tym przypadku alokator, który został przekazany do `init`, i tworzy nowy `std.mem.Allocator`. Kiedy ten nowy alokator jest używany do alokacji lub tworzenia pamięci, nie musimy wywoływać `free` ani `destroy`. Wszystko zostanie zwolnione, gdy wywołamy `deinit` na `arena`. W rzeczywistości, `free` i `destroy` alokatora ArenaAllocator nie robią nic.

`ArenaAllocator` musi być używany ostrożnie. Ponieważ nie ma sposobu na zwolnienie poszczególnych alokacji, musisz mieć pewność, że `deinit` areny zostanie wywołany w rozsądnym przyroście pamięci. Co ciekawe, wiedza ta może być wewnętrzna lub zewnętrzna. Na przykład w naszym powyższym szkielecie wykorzystanie ArenaAllocator ma sens z poziomu Parsera, ponieważ szczegóły dotyczące czasu życia stanu są kwestią wewnętrzną.

> Alokatory takie jak ArenaAllocator, które mają mechanizm zwalniania wszystkich poprzednich alokacji, mogą łamać zasadę, że każda `aloc` powinna mieć odpowiadające jej zwolnienie. Jeśli jednak otrzymasz `std.mem.Allocator`, nie powinieneś przyjmować żadnych założeń dotyczących jego implementacji.

Tego samego nie można powiedzieć o naszej liście `IntList`. Może ona służyć do przechowywania 10 lub 10 milionów wartości. Może mieć żywotność mierzoną w milisekundach lub tygodniach. Nie jest w stanie zdecydować, jakiego typu alokatora użyć. To kod korzystający z `IntList` posiada tę wiedzę. Pierwotnie zarządzaliśmy naszą listą `IntList` w następujący sposób:

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
const allocator = gpa.allocator();

var list = try IntList.init(allocator);
defer list.deinit();
```

Zamiast tego mogliśmy zdecydować się na dostarczenie ArenaAllocator:

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
const allocator = gpa.allocator();

var arena = std.heap.ArenaAllocator.init(allocator);
odroczyć arena.deinit();
const aa = arena.allocator();

var list = try IntList.init(aa);

// Szczerze mówiąc, jestem rozdarty co do tego, czy powinniśmy wywoływać list.deinit.
// Technicznie rzecz biorąc, nie musimy tego robić, ponieważ powyżej wywołaliśmy defer arena.deinit().
defer list.deinit();

...
```

Nie musimy zmieniać `IntList`, ponieważ zajmuje się ona tylko `std.mem.Allocator`. A gdyby `IntList` wewnętrznie tworzyła własną arenę, to też by działało. Nie ma powodu, dla którego nie można utworzyć areny wewnątrz areny.

Jako ostatni szybki przykład, serwer HTTP, o którym wspomniałem powyżej, udostępnia alokator areny w `Response`. Po wysłaniu odpowiedzi arena jest czyszczona. Przewidywalny czas życia areny (od początku do końca żądania) sprawia, że jest to efektywna opcja. Efektywną pod względem wydajności i łatwości użytkowania.

## FixedBufferAllocator

Ostatnim alokatorem, któremu się przyjrzymy, jest `std.heap.FixedBufferAllocator`, który alokuje pamięć z dostarczonego przez nas bufora (tj. `[]u8`). Alokator ten ma dwie główne zalety. Po pierwsze, ponieważ cała pamięć, której mógłby użyć, jest tworzona z góry, jest szybki. Po drugie, naturalnie ogranicza ilość pamięci, która może zostać przydzielona. Ten twardy limit może być również postrzegany jako wada. Kolejną wadą jest to, że `free` i `destroy` działają tylko na ostatnio przydzielonym/utworzonym elemencie (pomyśl o stosie). Zwolnienie nieostatniej alokacji jest bezpieczne, ale nic nie da.

```zig
const std = @import("std");

pub fn main() !void {
    var buf: [150]u8 = undefined;
    var fa = std.heap.FixedBufferAllocator.init(&buf);

    // spowoduje to zwolnienie całej pamięci przydzielonej za pomocą tego alokatora
    defer fa.reset();

    const allocator = fa.allocator();

    const json = try std.json.stringifyAlloc(allocator, .{
        .this_is = "an anonymous struct",
        .above = true,
        .last_param = "are options",
    }, .{.whitespace = .indent_2});

    // Możemy zwolnić tę alokację, ale ponieważ wiemy, że nasz alokator jest
    // FixedBufferAllocator, możemy polegać na powyższym `defer fa.reset()`.
    defer allocator.free(json);

    std.debug.print("{s}\n", .{json});
}
```

The above prints:

```
{
  "this_is": "an anonymous struct",
  "above": true,
  "last_param": "are options"
}
```

Ale zmień nasz `buf` na `[120]u8`, a otrzymasz błąd `OutOfMemory`.

Powszechnym wzorcem dla alokatorów FixedBufferAllocator, i w mniejszym stopniu ArenaAllocator, jest ich resetowanie i ponowne użycie. Zwalnia to wszystkie poprzednie alokacje i pozwala na ponowne użycie alokatora.

---

Nie posiadając domyślnego alokatora, Zig jest zarówno przejrzysty, jak i elastyczny w odniesieniu do alokacji. Interfejs `std.mem.Allocator` jest potężny, umożliwiając wyspecjalizowanym alokatorom zawijanie bardziej ogólnych, jak widzieliśmy w przypadku `ArenaAllocator`.

Ogólnie rzecz biorąc, moc i związane z nią obowiązki alokacji na stercie są, miejmy nadzieję, oczywiste. Możliwość alokacji pamięci o dowolnym rozmiarze i dowolnym czasie życia jest niezbędna dla większości programów.

Jednak ze względu na złożoność związaną z pamięcią dynamiczną, powinieneś mieć oko otwarte na alternatywy. Na przykład powyżej użyliśmy `std.fmt.allocPrint`, ale biblioteka standardowa ma również `std.fmt.bufPrint`. Ta ostatnia pobiera bufor zamiast alokatora:

```zig
const std = @import("std");

pub fn main() !void {
    const name = "Leto";

    var buf: [100]u8 = undefined;
    const greeting = try std.fmt.bufPrint(&buf, "Hello {s}", .{name});

    std.debug.print("{s}\n", .{greeting});
}
```

Ten interfejs API przenosi ciężar zarządzania pamięcią na wywołującego. Gdybyśmy mieli dłuższą `name` lub mniejszy `buf`, nasz `bufPrint` mógłby zwrócić błąd `NoSpaceLeft`. Istnieje jednak wiele scenariuszy, w których aplikacja ma znane ograniczenia, takie jak maksymalna długość nazwy. W takich przypadkach `bufPrint` jest bezpieczniejszy i szybszy.

Inną możliwą alternatywą dla dynamicznych alokacji jest strumieniowe przesyłanie danych do `std.io.Writer`. Podobnie jak nasz `Allocator`, `Writer` jest interfejsem implementowanym przez wiele typów, takich jak pliki. Powyżej użyliśmy `stringifyAlloc` do serializacji JSON do dynamicznie alokowanego ciągu znaków. Mogliśmy użyć `stringify` i dostarczyć `Writer`:

```zig
const std = @import("std");

pub fn main() !void {
    const out = std.io.getStdOut();

    try std.json.stringify(.{
        .this_is = "an anonymous struct",
        .above = true,
        .last_param = "are options",
    }, .{.whitespace = .indent_2}, out.writer());
}
```

> Podczas gdy alokatory są często podawane jako pierwszy parametr funkcji, `writer` są zwykle ostatnimi. ಠ_ಠ

W wielu przypadkach zawinięcie naszego writera w `std.io.BufferedWriter` dałoby niezły wzrost wydajności.

Celem nie jest wyeliminowanie wszystkich dynamicznych alokacji. To nie zadziała, ponieważ te alternatywy mają sens tylko w określonych przypadkach. Ale teraz masz do dyspozycji wiele opcji. Od ramek stosu po alokator ogólnego przeznaczenia i wszystko pomiędzy, takie jak bufory statyczne, zapisy strumieniowe i wyspecjalizowane alokatory.
