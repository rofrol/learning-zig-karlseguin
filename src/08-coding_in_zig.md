# Kodowanie w Zigu

Ponieważ znaczna część języka została już omówiona, zamierzamy zakończyć sprawy, powracając do kilku tematów i przyglądając się kilku bardziej praktycznym aspektom korzystania z Ziga. W ten sposób wprowadzimy więcej standardowej biblioteki i przedstawimy mniej trywialne fragmenty kodu.

## Zwisające wskaźniki

Zaczniemy od przyjrzenia się większej liczbie przykładów zwisających wskaźników. Może się to wydawać dziwną rzeczą, na której należy się skupić, ale jeśli przychodzisz z języka z garbage collectorem, jest to prawdopodobnie największe wyzwanie, z jakim będziesz musiał się zmierzyć.

Czy potrafisz odgadnąć, co poniższe wypisze?

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var lookup = std.StringHashMap(User).init(allocator);
    defer lookup.deinit();

    const goku = User{.power = 9001};

    try lookup.put("Goku", goku);

    // zwraca opcjonalne, .? spanikowałoby, gdyby "Goku"
    // nie było w naszej hashmapie
    const entry = lookup.getPtr("Goku").?;

    std.debug.print("Goku's power is: {d}\n", .{entry.power});

    // zwraca prawdę/fałsz w zależności od tego, czy element został usunięty
    _ = lookup.remove("Goku");

    std.debug.print("Goku's power is: {d}\n", .{entry.power});
}

const User = struct {
    power: i32,
};
```

Kiedy to uruchomiłem, otrzymałem:

```
Goku's power is: 9001
Goku's power is: -1431655766
```

Ten kod wprowadza generyczną `std.StringHashMap` Ziga, która jest wyspecjalizowaną wersją `std.AutoHashMap` z typem klucza ustawionym na `[]const u8`. Nawet jeśli nie jesteś w 100% pewien, co się dzieje, to dobre odgadnięcie, że moje wyjście odnosi się do faktu, że nasz drugi `print` ma miejsce po usunięciu wpisu z `lookup`. Wykreśl wywołanie `remove`, a wynik będzie normalny.

Kluczem do zrozumienia powyższego kodu jest świadomość tego, gdzie istnieją dane/pamięć lub, mówiąc inaczej, kto jest ich _właścicielem_. Pamiętaj, że argumenty Ziga są przekazywane przez wartość, to znaczy przekazujemy [płytką] kopię wartości. `User` w naszym `lookup` nie jest tą samą pamięcią, do której odwołuje się `goku`. Nasz powyższy kod ma **dwóch** użytkowników, każdy z własnym właścicielem. `goku` jest własnością `main`, a jego kopia jest własnością `lookup`.

Metoda `getPtr` zwraca wskaźnik do wartości w mapie, w naszym przypadku zwraca `*User`. W tym tkwi problem, `remove` sprawia, że nasz wskaźnik `entry` jest nieważny. W tym przykładzie bliskość `getPtr` i `remove` sprawia, że problem jest dość oczywisty. Nietrudno jednak wyobrazić sobie kod wywołujący `remove` bez świadomości, że referencja do wpisu jest przechowywana gdzie indziej.

> Kiedy pisałem ten przykład, nie byłem pewien, co się stanie. Możliwe było zaimplementowanie `remove` poprzez ustawienie wewnętrznej flagi, opóźniając faktyczne usunięcie do późniejszego zdarzenia. Gdyby tak było, powyższy przykład mógłby "zadziałać" w naszych prostych przypadkach, ale zawiódłby przy bardziej skomplikowanym użyciu. Brzmi to przerażająco trudno do debugowania.

Oprócz nie wywoływania `remove`, możemy to naprawić na kilka różnych sposobów. Pierwszym z nich jest użycie `get` zamiast `getPtr`. Zwróciłoby to `User` zamiast `*User`, a tym samym zwróciłoby kopię wartości w `lookup`. Mielibyśmy wtedy trzech `User`.

1. Nasz oryginalny `goku`, powiązany z funkcją.
2. Kopia w `lookup`, będącą własnością `lookup`.
3. Oraz kopia naszej kopii, `entry`, również powiązana z funkcją.

Ponieważ `entry` byłby teraz własną, niezależną kopią użytkownika, usunięcie go z `lookup` nie spowodowałoby jego unieważnienia.

Inną opcją jest zmiana typu `lookup` z `StringHashMap(User)` na `StringHashMap(*const User)`. Ten kod działa:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    // User -> *const User
    var lookup = std.StringHashMap(*const User).init(allocator);
    defer lookup.deinit();

    const goku = User{.power = 9001};

    // goku -> &goku
    try lookup.put("Goku", &goku);

    // getPtr -> get
    const entry = lookup.get("Goku").?;

    std.debug.print("Goku's power is: {d}\n", .{entry.power});
    _ = lookup.remove("Goku");
    std.debug.print("Goku's power is: {d}\n", .{entry.power});
}

const User = struct {
    power: i32,
};
```

Powyższy kod zawiera kilka subtelności. Po pierwsze, mamy teraz jednego `User`, `goku`. Wartość w `lookup` i `entry` są obie referencjami do `goku`. Nasze wywołanie `remove` nadal usuwa wartość z naszego `lookup`, ale ta wartość jest tylko adresem `user`, nie jest samym `user`. Gdybyśmy pozostali przy `getPtr`, otrzymalibyśmy nieprawidłowy `**User`, nieważny z powodu `remove`. W obu rozwiązaniach musieliśmy użyć `get` zamiast `getPtr`, ale w tym przypadku kopiujemy tylko adres, a nie pełnego `User`. W przypadku dużych obiektów może to być znacząca różnica.

Ze wszystkim w jednej funkcji i małą wartością, taką jak `User`, nadal wydaje się to sztucznie stworzonym problemem. Potrzebujemy przykładu, który zasadnie sprawi, że własność danych stanie się bezpośrednim problemem.

## Własność (ownership)

Uwielbiam hashmapy, ponieważ są one czymś, co wszyscy znają i czego wszyscy używają. Mają one również wiele różnych zastosowań, z których większość prawdopodobnie doświadczyłeś na własnej skórze. Chociaż mogą być używane jako krótkotrwałe wyszukiwania, często są długotrwałe, a zatem wymagają równie długotrwałych wartości.

Ten kod wypełnia nasz `lookup` nazwami wprowadzanymi w terminalu. Pusta nazwa zatrzymuje pętlę zachęty. Na koniec wykrywa, czy "Leto" było jednym z podanych nazw.

```zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var lookup = std.StringHashMap(User).init(allocator);
    defer lookup.deinit();

  // stdin to std.io.Reader
  // przeciwieństwo std.io.Writer, które już widzieliśmy
    const stdin = std.io.getStdIn().reader();

  // stdout to std.io.Writer
    const stdout = std.io.getStdOut().writer();

    var i: i32 = 0;
    while (true) : (i += 1) {
        var buf: [30]u8 = undefined;
        try stdout.print("Please enter a name: ", .{});
        if (try stdin.readUntilDelimiterOrEof(&buf, '\n')) |line| {
            var name = line;
            if (builtin.os.tag == .windows) {
                // W systemie Windows linie są zakończone znakiem \r\n.
                // Musimy usunąć \r
                name = std.mem.trimRight(u8, name, "\r");
            }
            if (name.len == 0) {
                break;
            }
            try lookup.put(name, .{.power = i});
        }
    }

    const has_leto = lookup.contains("Leto");
    std.debug.print("{any}\n", .{has_leto});
}

const User = struct {
    power: i32,
};
```

W kodzie rozróżniana jest wielkość liter, ale bez względu na to, jak idealnie wpiszemy "Leto", `contains` zawsze zwraca `false`. Zdebugujmy to, iterując przez `lookup` i zrzucając klucze i wartości:

```zig
// Umieść ten kod po pętli while

var it = lookup.iterator();
while (it.next()) |kv| {
    std.debug.print("{s} == {any}\n", .{kv.key_ptr.*, kv.value_ptr.*});
}
```

Ten wzorzec iteratora jest powszechny w Zigu i opiera się na synergii między typami `while` i `optional`. Nasz iterator zwraca wskaźniki do naszego klucza i wartości, dlatego dereferencjonujemy je za pomocą `.*`, aby uzyskać dostęp do rzeczywistej wartości, a nie adresu. Wynik będzie zależał od tego, co wprowadzisz, ale mam:

```
Please enter a name: Paul
Please enter a name: Teg
Please enter a name: Leto
Please enter a name:

�� == learning.User{ .power = 1 }

��� == learning.User{ .power = 0 }

��� == learning.User{ .power = 2 }
false
```

Wartości wyglądają w porządku, ale nie klucze. Jeśli nie jesteś pewien, co się dzieje, to prawdopodobnie moja wina. Wcześniej celowo źle skierowałem twoją uwagę. Powiedziałem, że mapy hash są często długotrwałe, a zatem wymagają długotrwałych _wartości_. Prawda jest taka, że wymagają one zarówno długotrwałych wartości, **jak i** długotrwałych kluczy! Zauważ, że `buf` jest zdefiniowany wewnątrz naszej pętli `while`. Kiedy wywołujemy `put`, dajemy naszej hashmapie klucz, który ma znacznie krótszy czas życia niż sama hashmapa. Przeniesienie `buf` _poza_ pętlę `while` rozwiązuje nasz problem z czasem życia, ale ten bufor jest ponownie wykorzystywany w każdej iteracji. Nadal nie będzie działać, ponieważ mutujemy podstawowe dane klucza.

Dla naszego powyższego kodu istnieje tylko jedno rozwiązanie: nasz `lookup` musi przejąć klucze na własność. Musimy dodać jedną linię i zmienić inną:

```zig
// zastąpić istniejący lookup.put tymi dwoma liniami
const owned_name = try allocator.dupe(u8, name);

// name -> owned_name
try lookup.put(owned_name, .{.power = i});
```

`dupe` to metoda `std.mem.Allocator`, której wcześniej nie widzieliśmy. Alokuje ona duplikat podanej wartości. Kod teraz działa, ponieważ nasze klucze, znajdujące się teraz na stercie, przeżywają `lookup`. W rzeczywistości wykonaliśmy zbyt dobrą robotę, wydłużając czas życia tych łańcuchów: wprowadziliśmy wycieki pamięci.

Można by pomyśleć, że kiedy wywołamy `lookup.deinit`, nasze klucze i wartości zostaną dla nas zwolnione. Ale nie ma jednego uniwersalnego rozwiązania, którego `StringHashMap` mógłby użyć. Po pierwsze, klucze mogą być literałami łańcuchowymi, których nie można zwolnić. Po drugie, mogły one zostać utworzone przy użyciu innego alokatora. Wreszcie, choć bardziej zaawansowane, istnieją uzasadnione przypadki, w których klucze mogą nie być własnością hashmapy.

Jedynym rozwiązaniem jest samodzielne zwolnienie kluczy. W tym momencie prawdopodobnie sensowne byłoby utworzenie własnego typu `UserLookup` i enkapsulacja logiki czyszczenia w naszej funkcji `deinit`. Zachowamy bałagan:

```zig
// zastąpimy istniejące:
//   defer lookup.deinit();
// z:
defer {
    var it = lookup.keyIterator();
    while (it.next()) |key| {
        allocator.free(key.*);
    }
    lookup.deinit();
}
```

Nasza logika `defer`, pierwsza, jaką widzieliśmy z blokiem, zwalnia każdy klucz, a następnie deinicjalizuje `lookup`. Używamy `keyIterator` tylko do iteracji kluczy. Wartość iteratora jest wskaźnikiem do wpisu klucza w hashmapie, `*[]const u8`. Chcemy zwolnić rzeczywistą wartość, ponieważ to właśnie ją zaalokowaliśmy za pomocą `dupe`, więc dereferencjonujemy wartość za pomocą `.*`.

Obiecuję, że skończyliśmy rozmawiać o zwisających wskaźnikach i zarządzaniu pamięcią. To, co omówiliśmy, może być nadal niejasne lub zbyt abstrakcyjne. Dobrze jest wrócić do tego tematu, gdy będziesz miał bardziej praktyczny problem do rozwiązania. To powiedziawszy, jeśli planujesz napisać coś nietrywialnego, jest to coś, co prawie na pewno będziesz musiał opanować. Kiedy poczujesz się na siłach, zachęcam do skorzystania z przykładu pętli zachęty i pobawienia się nią na własną rękę. Wprowadź typ `UserLookup`, który enkapsuluje całe zarządzanie pamięcią, które musieliśmy wykonać. Wypróbuj wartości `*User` zamiast `User`, tworząc użytkowników na stercie i zwalniając ich tak, jak zrobiliśmy to z kluczami. Napisz testy obejmujące nową strukturę, używając `std.testing.allocator`, aby upewnić się, że nie wycieka żadna pamięć.

## ArrayList

Będziesz zadowolony wiedząc, że możesz zapomnieć o naszej `IntList` i generycznej alternatywie, którą stworzyliśmy. Zig ma odpowiednią implementację dynamicznej tablicy: `std.ArrayList(T)`.

To dość standardowa rzecz, ale jest to tak powszechnie potrzebna i używana struktura danych, że warto zobaczyć ją w akcji:

```zig
const std = @import("std");
const builtin = @import("builtin");
const Allocator = std.mem.Allocator;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var arr = std.ArrayList(User).init(allocator);
    defer {
        for (arr.items) |user| {
            user.deinit(allocator);
        }
        arr.deinit();
    }

  // stdin to std.io.Reader
  // przeciwieństwo std.io.Writer, które już widzieliśmy
    const stdin = std.io.getStdIn().reader();

  // stdout to std.io.Writer
    const stdout = std.io.getStdOut().writer();

    var i: i32 = 0;
    while (true) : (i += 1) {
        var buf: [30]u8 = undefined;
        try stdout.print("Please enter a name: ", .{});
        if (try stdin.readUntilDelimiterOrEof(&buf, '\n')) |line| {
            var name = line;
            if (builtin.os.tag == .windows) {
                // W systemie Windows linie są zakończone znakiem \r\n.
                // Musimy usunąć \r
                name = std.mem.trimRight(u8, name, "\r");
            }
            if (name.len == 0) {
                break;
            }
            const owned_name = try allocator.dupe(u8, name);
            try arr.append(.{.name = owned_name, .power = i});
        }
    }

    var has_leto = false;
    for (arr.items) |user| {
        if (std.mem.eql(u8, "Leto", user.name)) {
            has_leto = true;
            break;
        }
    }

    std.debug.print("{any}\n", .{has_leto});
}

const User = struct {
    name: []const u8,
    power: i32,

    fn deinit(self: User, allocator: Allocator) void {
        allocator.free(self.name);
    }
};
```

Powyżej znajduje się reprodukcja naszego kodu mapy hash, ale przy użyciu `ArrayList(User)`. Obowiązują te same zasady dotyczące czasu życia i zarządzania pamięcią. Zauważ, że nadal tworzymy duplikat nazwy i nadal zwalniamy każdą nazwę przed `deinit` `ArrayList`.

To dobry moment, aby podkreślić, że Zig nie ma właściwości (properties) lub pól prywatnych. Widać to, gdy uzyskujemy dostęp do `arr.items`, aby iterować po wartościach. Powodem braku właściwości jest wyeliminowanie źródła niespodzianek. W Zigu, jeśli coś wygląda jak dostęp do pola, to jest to dostęp do pola. Osobiście uważam, że brak prywatnych pól jest błędem, ale z pewnością jest to coś, co możemy obejść. Zacząłem poprzedzać pola podkreśleniem, aby zasygnalizować "tylko do użytku wewnętrznego".

Ponieważ "typ" łańcucha to `[]u8` lub `[]const u8`, `ArrayList(u8)` jest odpowiednim typem dla konstruktora łańcuchów, takiego jak `StringBuilder` .NET lub `strings.Builder` w Go. W rzeczywistości często będziesz go używać, gdy funkcja pobiera `Writer` i chcesz uzyskać ciąg znaków. Wcześniej widzieliśmy przykład którym używał `std.json.stringify` do wyprowadzenia JSON na wyjście stdout. Oto jak można użyć `ArrayList(u8)` do wyprowadzenia go do zmiennej:

```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var out = std.ArrayList(u8).init(allocator);
    defer out.deinit();

    try std.json.stringify(.{
        .this_is = "an anonymous struct",
        .above = true,
        .last_param = "are options",
    }, .{.whitespace = .indent_2}, out.writer());

    std.debug.print("{s}\n", .{out.items});
}
```

## Anytype

W części 1 krótko omówiliśmy `anytype`. Jest to całkiem przydatna forma duck-typingu w czasie kompilacji. Oto prosty logger:

```zig
pub const Logger = struct {
    level: Level,

    // "błąd" jest zarezerwowany, nazwy wewnątrz @"..." są zawsze
    // traktowane jako identyfikatory
    const Level = enum {
        debug,
        info,
        @"error",
        fatal,
    };

    fn info(logger: Logger, msg: []const u8, out: anytype) !void {
        if (@intFromEnum(logger.level) <= @intFromEnum(Level.info)) {
            try out.writeAll(msg);
        }
    }
};
```

Parametr `out` naszej funkcji `info` ma typ `anytype`. Oznacza to, że nasz `Logger` może rejestrować komunikaty do dowolnej struktury, która ma metodę `writeAll` akceptującą `[]const u8` i zwracającą `!void`. Nie jest to funkcja czasu wykonania. Sprawdzanie typu odbywa się w czasie kompilacji i dla każdego używanego typu tworzona jest prawidłowo otypowana funkcja. Jeśli spróbujemy wywołać `info` z typem, który nie ma wszystkich niezbędnych funkcji (w tym przypadku tylko `writeAll`), otrzymamy błąd kompilacji:

```zig
var l = Logger{.level = .info};
try l.info("sever started", true);
```

Daje nam: _no field or member function named 'writeAll' in 'bool'_. Użycie `writer` na `ArrayList(u8)` działa:

```zig
pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var l = Logger{.level = .info};

    var arr = std.ArrayList(u8).init(allocator);
    defer arr.deinit();

    try l.info("sever started", arr.writer());
    std.debug.print("{s}\n", .{arr.items});
}
```

Ogromną wadą `anytype` jest dokumentacja. Oto sygnatura funkcji `std.json.stringify`, której używaliśmy kilka razy:

```zig
// **Nienawidzę** wieloliniowych definicji funkcji
// Ale zrobię wyjątek dla przewodnika, który
// możesz czytać na małym ekranie.

fn stringify(
    value: anytype,
    options: StringifyOptions,
    out_stream: anytype
) @TypeOf(out_stream).Error!void
```

Pierwszy parametr,` value: anytype`, jest dość oczywisty. Jest to wartość do serializacji i może to być cokolwiek (w rzeczywistości istnieją pewne rzeczy, których serializator JSON Ziga nie może serializować). Możemy zgadywać, że `out_stream` jest _miejscem_ zapisu JSON, ale równie dobrze można zgadywać, jakie metody musi zaimplementować. Jedynym sposobem, aby się tego dowiedzieć, jest przeczytanie kodu źródłowego lub, alternatywnie, przekazanie fikcyjnej wartości i użycie błędów kompilatora jako naszej dokumentacji. Jest to coś, co może ulec poprawie dzięki lepszym automatycznym generatorom dokumentów. Ale nie po raz pierwszy żałuję, że Zig nie ma interfejsów.

## @TypeOf

W poprzednich częściach użyliśmy `@TypeOf`, aby pomóc nam zbadać typ różnych zmiennych. Na podstawie naszego użycia można by pomyśleć, że zwraca ona nazwę typu jako ciąg znaków. Jednak biorąc pod uwagę, że jest to funkcja PascalCase, powinieneś wiedzieć lepiej: zwraca ona typ.

Jednym z moich ulubionych zastosowań `anytype` jest sparowanie go z wbudowanymi funkcjami `@TypeOf` i `@hasField` do pisania pomocników testowych. Chociaż każdy typ `User`, który widzieliśmy, był bardzo prosty, poproszę cię o wyobrażenie sobie bardziej złożonej struktury z wieloma polami. W wielu naszych testach potrzebujemy `User`, ale chcemy określić tylko pola istotne dla testu. Stwórzmy więc `userFactory`:

```zig
fn userFactory(data: anytype) User {
    const T = @TypeOf(data);
    return .{
        .id = if (@hasField(T, "id")) data.id else 0,
        .power = if (@hasField(T, "power")) data.power else 0,
        .active  = if (@hasField(T, "active")) data.active else true,
        .name  = if (@hasField(T, "name")) data.name else "",
    };
}

pub const User = struct {
    id: u64,
    power: u64,
    active: bool,
    name: [] const u8,
};
```

Domyślny użytkownik może zostać utworzony przez wywołanie `userFactory(.{})` lub możemy nadpisać określone pola za pomocą `userFactory(.{.id = 100, .active = false})`. To mały wzorzec, ale naprawdę mi się podoba. To także miły krok w świat metaprogramowania.

Częściej `@TypeOf` jest łączone z `@typeInfo`, które zwraca `std.builtin.Type`. Jest to potężny tagowany związek, który w pełni opisuje typ. Funkcja `std.json.stringify` rekurencyjnie używa tego na dostarczonej `value`, aby dowiedzieć się, jak ją serializować.

## Zig Build

Jeśli przeczytałeś cały ten przewodnik, czekając na wgląd w konfigurowanie bardziej złożonych projektów, z wieloma zależnościami i różnymi celami, będziesz rozczarowany. Zig ma potężny system kompilacji, tak bardzo, że coraz więcej projektów innych niż Zig korzysta z niego, takich jak libsodium. Niestety, cała ta moc oznacza, że dla prostszych potrzeb nie jest on najłatwiejszy w użyciu ani zrozumieniu.

> Prawda jest taka, że nie rozumiem systemu kompilacji Ziga wystarczająco dobrze, aby go wyjaśnić.

Mimo to możemy przynajmniej uzyskać krótki przegląd. Aby uruchomić nasz kod Zig, użyliśmy `zig run learning.zig`. Raz użyliśmy również `zig test learning.zig`, aby uruchomić test. Polecenia `run` i `test` są dobre do zabawy, ale to polecenie `build` będzie potrzebne do wszystkiego, co bardziej złożone. Polecenie `build` opiera się na pliku `build.zig` ze specjalnym punktem wejścia `build`. Oto szkielet:

```zig
// build.zig

const std = @import("std");

pub fn build(b: *std.Build) !void {
    _ = b;
}
```

Każda kompilacja ma domyślny krok "install", który można teraz uruchomić za pomocą `zig build install`, ale ponieważ nasz plik jest w większości pusty, nie otrzymamy żadnych znaczących artefaktów. Musimy powiedzieć naszemu `build` o punkcie wejścia naszego programu, który znajduje się w `learning.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // konfiguracja pliku wykonywalnego
    const exe = b.addExecutable(.{
        .name = "learning",
        .target = target,
        .optimize = optimize,
        .root_source_file = b.path("learning.zig"),
    });
    b.installArtifact(exe);
}
```

Teraz, jeśli uruchomisz `zig build install`, otrzymasz plik binarny w `./zig-out/bin/learning`. Korzystanie ze standardowych celów i optymalizacji pozwala nam zastąpić wartości domyślne jako argumenty wiersza poleceń. Na przykład, aby zbudować zoptymalizowaną pod kątem rozmiaru wersję naszego programu dla systemu Windows, wykonalibyśmy następujące polecenie:

```
zig build install -Doptimize=ReleaseSmall -Dtarget=x86_64-windows-gnu
```

Plik wykonywalny często ma dwa dodatkowe kroki, poza domyślnym "install": "run" i "test". Biblioteka może mieć pojedynczy krok "test". Aby uzyskać podstawowy `run` bez argumentów, musimy dodać cztery linie na końcu naszej kompilacji:

```zig
// dodajemy po: b.installArtifact(exe);

const run_cmd = b.addRunArtifact(exe);
run_cmd.step.dependOn(b.getInstallStep());

const run_step = b.step("run", "Start learning!");
run_step.dependOn(&run_cmd.step);
```

Tworzy to dwie zależności poprzez dwa wywołania `dependOn`. Pierwsza wiąże nasze nowe polecenie `run` z wbudowanym krokiem instalacji. Druga wiąże krok "run" z naszym nowo utworzonym poleceniem "run". Być może zastanawiasz się, dlaczego potrzebujesz zarówno polecenia run, jak i kroku run. Uważam, że ta separacja istnieje, aby wspierać bardziej skomplikowane konfiguracje: kroki, które zależą od wielu poleceń lub poleceń, które są używane w wielu krokach. Jeśli uruchomisz `zig build --help` i przewiniesz do góry, zobaczysz nasz nowy krok "run". Możesz teraz uruchomić program, wykonując polecenie `zig build run`.

Aby dodać krok "test", zduplikujesz większość kodu run, który właśnie dodaliśmy, ale zamiast `b.addExecutable`, rozpoczniesz wszystko od `b.addTest`:

```zig
const tests = b.addTest(.{
    .target = target,
    .optimize = optimize,
    .root_source_file = b.path("learning.zig"),
});

const test_cmd = b.addRunArtifact(tests);
test_cmd.step.dependOn(b.getInstallStep());
const test_step = b.step("test", "Uruchom testy");
test_step.dependOn(&test_cmd.step);
```

Nadaliśmy temu krokowi nazwę "test". Uruchomienie `zig build --help` powinno teraz pokazać kolejny dostępny krok, "test". Ponieważ nie mamy żadnych testów, trudno powiedzieć, czy to działa, czy nie. W pliku `learning.zig` dodajemy:

```zig
test "dummy build test" {
    try std.testing.expectEqual(false, true);
}
```

Teraz, po uruchomieniu testu `zig build`, powinien pojawić się komunikat o niepowodzeniu testu. Jeśli naprawisz test i ponownie uruchomisz `zig build test`, nie otrzymasz żadnych danych wyjściowych. Domyślnie program uruchamiający testy Ziga generuje dane wyjściowe tylko w przypadku niepowodzenia. Użyj `zig build test --summary all` jeśli, tak jak ja, zawsze chcesz otrzymać podsumowanie.

Jest to minimalna konfiguracja potrzebna do rozpoczęcia pracy. Możesz jednak spać spokojnie, wiedząc, że jeśli zajdzie potrzeba jej zbudowania, Zig prawdopodobnie sobie z tym poradzi. Wreszcie, możesz i prawdopodobnie powinieneś użyć `zig init` w katalogu głównym projektu, aby Zig utworzył dla ciebie dobrze udokumentowany plik build.zig.

## Zależności od stron trzecich

Wbudowany menedżer pakietów Ziga jest stosunkowo nowy i w związku z tym ma wiele nieoszlifowanych krawędzi. Chociaż jest miejsce na ulepszenia, jest on użyteczny tak jak jest. Istnieją dwie części, którym musimy się przyjrzeć: tworzenie pakietu i korzystanie z pakietów. Przejdziemy przez to w całości.

Najpierw utwórz nowy folder o nazwie `calc` i utwórz trzy pliki. Pierwszy to `add.zig`, z następującą zawartością:

```zig
// O, ukryta lekcja, spójrz na typ b
// i typ zwracany!!!

pub fn add(a: anytype, b: @TypeOf(a)) @TypeOf(a) {
    return a + b;
}

const testing = @import("std").testing;
test "add" {
    try testing.expectEqual(@as(i32, 32), add(30, 2));
}
```

To trochę głupie, cały pakiet tylko po to, by dodać dwie wartości, ale pozwoli nam skupić się na aspekcie pakowania. Następnie dodamy równie głupi pakiet: `calc.zig`:

```zig
pub const add = @import("add.zig").add;

test {
  // Domyślnie, tylko testy w określonym pliku
  // są uwzględniane. Ta magiczna linia kodu
  // spowoduje, że odniesienie do wszystkich zagnieżdżonych kontenerów
  // do wszystkich zagnieżdżonych kontenerów.
    @import("std").testing.refAllDecls(@This());
}
```

Rozdzielamy to między `calc.zig` i `add.zig`, aby udowodnić, że `zig build` automatycznie zbuduje i spakuje wszystkie pliki naszego projektu. Na koniec możemy dodać `build.zig`:

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const tests = b.addTest(.{
        .target = target,
        .optimize = optimize,
        .root_source_file = b.path("calc.zig"),
    });

    const test_cmd = b.addRunArtifact(tests);
    test_cmd.step.dependOn(b.getInstallStep());
    const test_step = b.step("test", "Run the tests");
    test_step.dependOn(&test_cmd.step);
}
```

To wszystko jest powtórzeniem tego, co widzieliśmy w poprzedniej sekcji. W ten sposób można uruchomić `zig build test --summary all`.

Wracamy do naszego projektu `learning` i wcześniej utworzonego `build.zig`. Zaczniemy od dodania naszego lokalnego `calc` jako zależności. Musimy wprowadzić trzy dodatki. Po pierwsze, utworzymy moduł wskazujący na nasz `calc.zig`:

```zig
// Można go umieścić w górnej części funkcji build
// funkcji, przed wywołaniem addExecutable.

const calc_module = b.addModule("calc", .{
  .root_source_file = b.path("PATH_TO_CALC_PROJECT/calc.zig"),
});
```

Będziesz musiał dostosować ścieżkę do `calc.zig`. Teraz musimy dodać ten moduł do naszych istniejących zmiennych `exe` i `tests`. Ponieważ nasz `build.zig` staje się coraz bardziej zajęty, postaramy się trochę uporządkować rzeczy:

```zig
const std = @import("std");

pub fn build(b: *std.Build) !void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const calc_module = b.addModule("calc", .{
        .root_source_file = b.path("PATH_TO_CALC_PROJECT/calc.zig"),
    });

    {
    // skonfiguruj nasze polecenia "run"

        const exe = b.addExecutable(.{
            .name = "learning",
            .target = target,
            .optimize = optimize,
            .root_source_file = b.path("learning.zig"),
        });
    // dodaj to
        exe.root_module.addImport("calc", calc_module);
        b.installArtifact(exe);

        const run_cmd = b.addRunArtifact(exe);
        run_cmd.step.dependOn(b.getInstallStep());

        const run_step = b.step("run", "Start learning!");
        run_step.dependOn(&run_cmd.step);
    }

    {
    // skonfiguruj nasze polecenie "test"
        const tests = b.addTest(.{
            .target = target,
            .optimize = optimize,
            .root_source_file = b.path("learning.zig"),
        });
    // dodaj to
        tests.root_module.addImport("calc", calc_module);

        const test_cmd = b.addRunArtifact(tests);
        test_cmd.step.dependOn(b.getInstallStep());
        const test_step = b.step("test", "Run the tests");
        test_step.dependOn(&test_cmd.step);
    }
}
```

Z poziomu projektu możesz teraz `@import("calc")`:

```zig
const calc = @import("calc");
...
calc.add(1, 2);
```

Dodanie zdalnej zależności wymaga nieco więcej wysiłku. Najpierw musimy wrócić do projektu `calc` i zdefiniować moduł. Można by pomyśleć, że sam projekt jest modułem, ale projekt może eksponować wiele modułów, więc musimy go jawnie utworzyć. Używamy tego samego `addModule`, ale odrzucamy wartość zwracaną. Samo wywołanie `addModule` wystarczy, aby zdefiniować moduł, który inne projekty będą mogły zaimportować.

```zig
_ = b.addModule("calc", .{
  .root_source_file = b.path("calc.zig"),
});
```

To jedyna zmiana, jaką musimy wprowadzić w naszej bibliotece. Ponieważ jest to ćwiczenie polegające na posiadaniu zdalnej zależności, przesłałem ten projekt `calc` na Github, abyśmy mogli zaimportować go do naszego projektu edukacyjnego. Jest on dostępny pod adresem https://github.com/karlseguin/calc.zig.

W naszym projekcie edukacyjnym potrzebujemy nowego pliku, `build.zig.zon`. "ZON" oznacza Zig Object Notation i umożliwia wyrażanie danych Ziga w formacie czytelnym dla człowieka oraz przekształcanie tego formatu w kod Ziga. Zawartość `build.zig.zon` będzie następująca:

```zon
.{
  .name = "learning",
  .paths = .{""},
  .version = "0.0.0",
  .dependencies = .{
    .calc = .{
      .url = "https://github.com/karlseguin/calc.zig/archive/d1881b689817264a5644b4d6928c73df8cf2b193.tar.gz",
      .hash = "12ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
    },
  },
}
```

W tym pliku znajdują się dwie wątpliwe wartości, pierwsza to `d1881b689817264a5644b4d6928c73df8cf2b193` w adresie `url`. Jest to po prostu git commit hash. Drugi to wartość `hash`. O ile mi wiadomo, obecnie nie ma dobrego sposobu na określenie, jaka powinna być ta wartość, więc na razie używamy wartości fikcyjnej.

Aby użyć tej zależności, musimy dokonać jednej zmiany w naszym `build.zig`:

```zig
// zamień to:
const calc_module = b.addModule("calc", .{
  .root_source_file = b.path("calc/calc.zig"),
});

// z tym:
const calc_dep = b.dependency("calc", .{ .target = target, .optimize = optimize});
const calc_module = calc_dep.module("calc");
```

W `build.zig.zon` nazwaliśmy zależność `calc` i jest to zależność, którą ładujemy tutaj. Z poziomu tej zależności pobieramy moduł `calc`, który został nazwany w `build.zig` w `calc`.

Jeśli spróbujesz uruchomić `zig build test`, powinieneś zobaczyć błąd:

```
hash mismatch: manifest declares
122053da05e0c9348d91218ef015c8307749ef39f8e90c208a186e5f444e818672da

but the fetched package has
122036b1948caa15c2c9054286b3057877f7b152a5102c9262511bf89554dc836ee5
```

Skopiuj i wklej poprawny hash z powrotem do `build.zig.zon` i spróbuj ponownie uruchomić `zig build test`. Wszystko powinno teraz działać.

Wydaje się, że to dużo i mam nadzieję, że wszystko zostanie usprawnione. Ale jest to głównie coś, co można skopiować i wkleić z innych projektów, a po skonfigurowaniu można przejść dalej.

Słowo ostrzeżenia, zauważyłem, że buforowanie zależności w Zigu jest po agresywnej stronie. Jeśli próbujesz zaktualizować zależność, ale Zig wydaje się nie wykrywać zmiany...cóż, wyrzucam folder `zig-cache` projektu, a także `~/.cache/zig`.

---

Omówiliśmy wiele obszarów, badając kilka podstawowych struktur danych i łącząc ze sobą duże fragmenty poprzednich części. Nasz kod stał się nieco bardziej złożony, skupiając się mniej na konkretnej składni i wyglądając bardziej jak prawdziwy kod. Jestem podekscytowany możliwością, że pomimo tej złożoności, kod w większości miał sens. Jeśli nie, nie poddawaj się. Wybierz przykład i złam go, dodaj instrukcje wypisywania, napisz dla niego testy. Zajmij się kodem, stwórz własny, a następnie wróć i przeczytaj te części, które nie miały sensu.
