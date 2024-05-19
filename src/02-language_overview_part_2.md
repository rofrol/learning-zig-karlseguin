# Przegląd języka - część 2

Ta część jest kontynuacją poprzedniej: zapoznanie się z językiem. Zbadamy przepływ sterowania Ziga i inne typy oprócz struktur. Wraz z pierwszą częścią omówimy większość składni języka, co pozwoli nam zająć się większą częścią języka i biblioteki standardowej.

## Przepływ sterowania

Przepływ sterowania Ziga jest prawdopodobnie Ci znany, ale musimy go jeszcze zbadać z dodatkowymi synergiami z aspektami języka. Zaczniemy od szybkiego przeglądu przepływu sterowania i wrócimy do omawiania funkcji, które wywołują specjalne zachowanie przepływu sterowania.

Zauważysz, że zamiast operatorów logicznych `&&` i `||`, używamy `and` i `or`. Podobnie jak w większości języków, `and` i `or` kontrolują przepływ wykonania: robią krótkie spięcie. Prawa strona `and` nie jest obliczana, jeśli lewa strona jest fałszywa, a prawa strona `or` nie jest obliczana, jeśli lewa strona jest prawdziwa. W Zigu przepływ sterowania odbywa się za pomocą słów kluczowych, a zatem używane są `and` i `or`.

Ponadto operator porównania, `==`, nie działa z wycinkami, takimi jak `[]const u8`, tj. łańcuchami. W większości przypadków należy użyć `std.mem.eql(u8, str1, str2)`, który porówna długość, a następnie bajty dwóch wycinków.

`if`, `else if` i `else` są powszechne w Zigu:

```zig
// std.mem.eql porównuje bajt po bajcie
// dla łańcucha będzie rozróżniana wielkość liter
if (std.mem.eql(u8, method, "GET") or std.mem.eql(u8, method, "HEAD")) {
    // obsłuż żądanie GET
} else if (std.mem.eql(u8, method, "POST")) {
    // obsłuż żądanie POST
} else {
    // ...
}
```

> Pierwszym argumentem funkcji `std.mem.eql` jest typ, w tym przypadku `u8`. Jest to pierwsza funkcja generyczna, którą widzieliśmy. Omówimy to bardziej szczegółowo w dalszej części.

Powyższy przykład porównuje łańcuchy ASCII i raczej powinien być niewrażliwy na wielkość liter. `std.ascii.eqlIgnoreCase(str1, str2)` jest prawdopodobnie lepszą opcją.

Nie ma operatora trójargumentowego, ale można użyć `if/else` w następujący sposób:

```zig
const super = if (power > 9000) true else false;
```

`switch` jest podobny do `if/else`, ale ma tę zaletę, że jest wyczerpujący. Oznacza to, że jeśli nie wszystkie przypadki zostaną uwzględnione, wystąpi błąd kompilacji. Ten kod nie zostanie skompilowany:

```zig
fn anniversaryName(years_married: u16) []const u8 {
    switch (years_married) {
        1 => return "papier",
        2 => return "bawełna",
        3 => return "skóra",
        4 => return "kwiat",
        5 => return "drewno",
        6 => return "cukier",
    }
}
```

Powiedziano nam: _switch musi obsługiwać wszystkie możliwości_. Ponieważ nasze `years_married` jest 16-bitową liczbą całkowitą, czy oznacza to, że musimy obsłużyć wszystkie 64K przypadków? Tak, ale na szczęście jest `else`:

```zig
// ...
6 => return "sugar",
else => return "no more gifts for you",
```

Możemy łączyć wiele przypadków lub używać zakresów, a także używać bloków dla złożonych przypadków:

```zig
fn arrivalTimeDesc(minutes: u16, is_late: bool) []const u8 {
    switch (minutes) {
        0 => return "arrived",
        1, 2 => return "soon",
        3...5 => return "no more than 5 minutes",
        else => {
            if (!is_late) {
                return "sorry, it'll be a while";
            }
            // todo, something is very wrong
            return "never";
        },
    }
}
```

Podczas gdy `switch` jest przydatny w wielu przypadkach, jego wyczerpująca natura naprawdę błyszczy, gdy mamy do czynienia z enumami, które omówimy wkrótce.

Pętla `for` Ziga służy do iteracji po tablicach, wycinkach i zakresach. Na przykład, aby sprawdzić, czy tablica zawiera wartość, możemy napisać:

```zig
fn contains(haystack: []const u32, needle: u32) bool {
    for (haystack) |value| {
        if (needle == value) {
            return true;
        }
    }
    return false;
}
```

Pętle `for` mogą działać na wielu sekwencjach jednocześnie, o ile sekwencje te są tej samej długości. Powyżej użyliśmy funkcji `std.mem.eql`. Oto jak to (prawie) wygląda:

```zig
pub fn eql(comptime T: type, a: []const T, b: []const T) bool {
    // jeśli nie mają tej samej długości, nie mogą być równe
    if (a.len != b.len) return false;

    for (a, b) |a_elem, b_elem| {
        if (a_elem != b_elem) return false;
    }

    return true;
}
```

Początkowe sprawdzenie `if` to nie tylko miła optymalizacja wydajności, to niezbędny strażnik. Jeśli go usuniemy i przekażemy argumenty o różnych długościach, otrzymamy runtime panic: _for loop over objects with non-equal lengths_.

Pętle `for` mogą również iterować po zakresach, takich jak:

```zig
for (0..10) |i| {
    std.debug.print("{d}\n", .{i});
}
```

> Nasz zakres `switch` używał trzech kropek, `3...6`, podczas gdy ten zakres używa dwóch, `0..10`. Dzieje się tak, ponieważ przypadki `switch` obejmują obie liczby, podczas gdy `for` wyklucza górną granicę.

To naprawdę błyszczy w połączeniu z jedną (lub więcej!) sekwencją:

```zig
fn indexOf(haystack: []const u32, needle: u32) ?usize {
    for (haystack, 0..) |value, i| {
        if (needle == value) {
            return i;
        }
    }
    return null;
}
```

> To jest tylko krótki rzut oka na typy nullable.

Koniec zakresu jest wywnioskowany z długości `haystack`, chociaż moglibyśmy się ukarać i napisać: `0..hastack.len`. Pętle `for` nie obsługują bardziej ogólnego idiomu `init; compare; step`. W tym celu polegamy na pętli `while`.

Ponieważ `while` jest prostsze, przyjmując formę `while (warunek) { }`, mamy większą kontrolę nad iteracją. Na przykład, podczas liczenia ilości sekwencji escape w łańcuchu, musimy zwiększyć nasz iterator o 2, aby uniknąć podwójnego liczenia `\`:

```zig
{
	var i: usize = 0;
	while (i < src.len) {
    // odwrotny ukośnik jest używany jako znak uwolnienia, więc musimy go uwolnić...
   // z odwrotnym ukośnikiem.
		if (src[i] == '\\') {
			i += 2;
			escape_count += 1;
		} else {
			i += 1;
		}
	}
}
```

Dodaliśmy jawny blok wokół naszej tymczasowej zmiennej `i` i pętli `while`. Zawęża to zakres `i`. Bloki takie jak ten mogą być przydatne, choć w tym przypadku jest to prawdopodobnie przesada. Jednak powyższy przykład to najbliższe co jest w Zigu do tradycyjnej pętli `for(init; compare; step)`.

`while` może mieć klauzulę `else`, która jest wykonywana, gdy warunek jest fałszywy. Akceptuje również instrukcję do wykonania po każdej iteracji. Może być wiele instrukcji oddzielonych `;`. Ta funkcja była powszechnie używana zanim `for` obsługiwało wielokrotne sekwencje. Powyższe można zapisać jako:

```zig
var i: usize = 0;
var escape_count: usize = 0;

// ta część
while (i < src.len) : (i += 1) {
    if (src[i] == '\\') {
        // +1 tutaj, oraz i +1 powyżej == +2
        i += 1;
        escape_count += 1;
    }
}
```

`break` i `continue` są obsługiwane w celu przerwania wewnętrznej pętli lub przejścia do następnej iteracji.

Bloki mogą być oznaczone etykietami, a `break` i `continue` mogą odnosić się do konkretnej etykiety. Wymyślony przykład:

```zig
outer: for (1..10) |i| {
    for (i..10) |j| {
        if (i * j > (i+i + j+j)) continue :outer;
        std.debug.print("{d} + {d} >= {d} * {d}\n", .{i+i, j+j, i, j});
    }
}
```

`break` ma jeszcze jedno interesujące zachowanie, zwracając wartość z bloku:

```zig
const personality_analysis = blk: {
    if (tea_vote > coffee_vote) break :blk "sane";
    if (tea_vote == coffee_vote) break :blk "whatever";
    if (tea_vote < coffee_vote) break :blk "dangerous";
};
```

Bloki takie jak ten muszą być zakończone średnikiem.

Później, gdy będziemy badać tagowane unie (tagged unions), unie błędów (error unions) i typy opcjonalne, zobaczymy, co jeszcze mają do zaoferowania te struktury przepływu sterowania.

## Enumy

Enumy są stałymi całkowitymi, które otrzymują etykietę. Są one zdefiniowane podobnie jak struktura:

```zig
// może to być "pub"
const Status = enum {
    ok,
    bad,
    unknown,
};
```

I, podobnie jak struktura, może zawierać inne definicje, w tym funkcje, które mogą, ale nie muszą, przyjmować enuma jako parametr:

```zig
const Stage = enum {
    validate,
    awaiting_confirmation,
    confirmed,
    completed,
    err,

    fn isComplete(self: Stage) bool {
        return self == .confirmed or self == .err;
    }
};
```

> Jeśli chcesz uzyskać reprezentację łańcuchową enuma, możesz użyć wbudowanej funkcji `@tagName(enum)`.

Przypomnijmy, że typy struktura można wywnioskować na podstawie ich przypisania lub typu zwracanego przy użyciu notacji `.{...}`. Powyżej widzimy, że typ enum jest wnioskowany na podstawie porównania z `self`, który jest typu `Stage`. Mogliśmy napisać wprost: `return self == Stage.confirmed or self == Stage.err;`. Jednak w przypadku enumów często można spotkać się z pominięciem typu enum za pomocą notacji `.$value`. Nazywa się to _literałem enum_.

Wyczerpująca natura `switch` sprawia, że dobrze łączy się z enumami, ponieważ zapewnia obsługę wszystkich możliwych przypadków. Zachowaj jednak ostrożność podczas korzystania z klauzuli `else` w `switch`, ponieważ będzie ona pasować do wszystkich nowo dodanych wartości do enuma, co może, ale nie musi być zachowaniem, które chcesz.

## Tagowane unie (tagged unions)

Unia definiuje zestaw typów, które wartość może mieć. Na przykład, ta unia `Number` może być `integer`, `float` lub `nan` (not a number - nie liczba):

```zig
const std = @import("std");

pub fn main() void {
    const n = Number{.int = 32};
    std.debug.print("{d}\n", .{n.int});
}

const Number = union {
    int: i64,
    float: f64,
    nan: void,
};
```

Unia może mieć ustawione tylko jedno pole na raz; próba uzyskania dostępu do nieustawionego pola jest błędem. Ponieważ ustawiliśmy pole `int`, gdybyśmy następnie spróbowali uzyskać dostęp do `n.float`, otrzymalibyśmy błąd. Jedno z naszych pól, `nan`, ma typ `void`. Jak moglibyśmy ustawić jego wartość? Używając `{}`:

```zig
const n = Number{.nan = {}};
```

Wyzwaniem w przypadku unii jest wiedza, które pole jest ustawione. W tym miejscu do gry wkraczają tagowane unie. Tagowana unia łączy enuma z unią, która może być użyta w instrukcji `switch`. Rozważmy następujący przykład:

```zig
pub fn main() void {
    const ts = Timestamp{.unix = 1693278411};
    std.debug.print("{d}\n", .{ts.seconds()});
}

const TimestampType = enum {
    unix,
    datetime,
};

const Timestamp = union(TimestampType) {
    unix: i32,
    datetime: DateTime,

    const DateTime = struct {
        year: u16,
        month: u8,
        day: u8,
        hour: u8,
        minute: u8,
        second: u8,
    };

    fn seconds(self: Timestamp) u16 {
        switch (self) {
            .datetime => |dt| return dt.second,
            .unix => |ts| {
                const seconds_since_midnight: i32 = @rem(ts, 86400);
                return @intCast(@rem(seconds_since_midnight, 60));
            },
        }
    }
};
```

Zauważ, że każdy przypadek w naszym `switch` przechwytuje wpisaną wartość pola. Oznacza to, że `dt` to `Timestamp.DateTime`, a `ts` to `i32`. Jest to również pierwszy raz, kiedy widzimy strukturę zagnieżdżoną w innym typie. `DateTime` mógł zostać zdefiniowany poza unią. Widzimy również dwie nowe wbudowane funkcje: `@rem`, aby uzyskać resztę i `@intCast`, aby przekonwertować wynik na `u16` (`@intCast` wnioskuje, że chcemy `u16` z naszego typu zwracanego, ponieważ zwracana jest wartość).

Jak widać na powyższym przykładzie, tagowane unie mogą być używane w pewnym sensie jak interfejsy, o ile wszystkie możliwe implementacje są znane z wyprzedzeniem i mogą być dodane do tagowanej unii.

Wreszcie, typ enum tagowanej unii może być wywnioskowany. Zamiast definiować `TimestampType`, mogliśmy zrobić:

```zig
const Timestamp = union(enum) {
    unix: i32,
    datetime: DateTime,

    ...
```

a Zig utworzyłby niejawny enum oparty na polach naszej unii.

## Opcjonalne

Każda wartość może być zadeklarowana jako opcjonalny poprzez dodanie znaku zapytania `?` do typu. Typy opcjonalne mogą mieć wartość `null` lub wartość zdefiniowanego typu:

```zig
var home: ?[]const u8 = null;
var name: ?[]const u8 = "Leto";
```

Potrzeba posiadania wyraźnego typu powinna być jasna: gdybyśmy po prostu zrobili `const name = "Leto";`, wówczas wnioskowanym typem byłby nie-opcjonalny `[]const u8`.

`.?` służy do uzyskania dostępu do wartości kryjącej się za typem opcjonalnym:

```zig
std.debug.print("{s}\n", .{name.?});
```

Jeśli jednak użyjemy `.?` na wartości `null`, otrzymamy `runtime panic`. Instrukcja `if` może bezpiecznie rozpakować opcjonalny:

```zig
if (home) |h| {
    // h jest []const u8
    // mamy wartość home
} else {
    // nie mamy wartości home
}
```

`orelse` może być używane do rozpakowania opcjonalnego lub wykonania kodu. Jest to często używane do określenia wartości domyślnej lub powrotu z funkcji:

```zig
const h = home orelse "unknown"
// lub może

// wyjście z funkcji
const h = home orelse return;
```

Jednak `orelse` może również otrzymać blok i wykonywać bardziej złożoną logikę. Typy opcjonalne również integrują się z `while` i są często używane do tworzenia iteratorów. Nie będziemy implementować iteratora, ale miejmy nadzieję, że ten fikcyjny kod ma sens:

```zig
while (rows.next()) |row| {
    // zrób coś z naszym wierszem
}
```

## Undefined

Jak dotąd, każda pojedyncza zmienna, którą widzieliśmy, została zainicjowana do sensownej wartości. Czasami jednak nie znamy wartości zmiennej w momencie jej deklaracji. Typy opcjonalne są jedną z opcji, ale nie zawsze mają sens. W takich przypadkach możemy ustawić zmienne na `undefined`, aby pozostawić je niezainicjalizowane.

Jednym z miejsc, w których jest to często wykonywane, jest tworzenie tablicy, która ma zostać wypełniona przez jakąś funkcję:

```zig
var pseudo_uuid: [16]u8 = undefined;
std.crypto.random.bytes(&pseudo_uuid);
```

Powyższe rozwiązanie nadal tworzy tablicę 16 bajtów, ale pozostawia pamięć niezainicjowaną.

## Błędy (errors)

Zig posiada proste i pragmatyczne możliwości obsługi błędów. Wszystko zaczyna się od zestawów błędów (error sets), które wyglądają i zachowują się jak enumy:

```zig
// Podobnie jak nasza struktura w części 1, OpenError może być oznaczony jako "pub"
// aby był dostępny poza plikiem, w którym jest zdefiniowany
const OpenError = error {
    AccessDenied,
    NotFound,
};
```

Funkcja, w tym `main`, może teraz zwrócić ten błąd:

```zig
pub fn main() void {
    return OpenError.AccessDenied;
}

const OpenError = error {
    AccessDenied,
    NotFound,
};
```

Jeśli spróbujesz to uruchomić, otrzymasz błąd: _expected type 'void', found 'error{AccessDenied,NotFound}'_. Ma to sens: zdefiniowaliśmy `main` z typem zwracanym `void`, ale zwracamy coś (błąd, oczywiście, ale to wciąż nie jest `void`). Aby rozwiązać ten problem, musimy zmienić typ zwracany naszej funkcji.

```zig
pub fn main() OpenError!void {
    return OpenError.AccessDenied;
}
```

Nazywa się to typem unii błędów i wskazuje, że nasza funkcja może zwrócić albo błąd `OpenError`, albo `void` (czyli nic). Do tej pory byliśmy dość jednoznaczni: utworzyliśmy zestaw błędów dla możliwych błędów, które może zwrócić nasza funkcja, i użyliśmy tego zestawu błędów w typie zwracanym naszej funkcji. Ale jeśli chodzi o błędy, Zig ma kilka zgrabnych sztuczek w rękawie. Po pierwsze, zamiast określać związek błędów jako `error set!return type`, możemy pozwolić Zigowi wywnioskować zestaw błędów za pomocą: `!return type`. Moglibyśmy więc, i prawdopodobnie byśmy to zrobili, zdefiniować nasz `main` jako:

```zig
pub fn main() !void
```

Po drugie, Zig jest w stanie niejawnie tworzyć dla nas zestawy błędów. Zamiast tworzyć nasz zestaw błędów, moglibyśmy zrobić:

```zig
pub fn main() !void {
    return error.AccessDenied;
}
```

Nasze całkowicie jawne i niejawne podejścia nie są dokładnie równoważne. Na przykład odniesienia do funkcji z niejawnymi zestawami błędów wymagają użycia specjalnego typu `anyerror`. Deweloperzy bibliotek mogą dostrzec zalety bycia bardziej jawnym, takie jak samodokumentujący się kod. Mimo to uważam, że zarówno niejawne zestawy błędów, jak i wywnioskowana unia błędów są pragmatyczne; intensywnie korzystam z obu.

Prawdziwą wartością unii błędów jest wbudowane wsparcie językowe w postaci `catch` i `try`. Wywołanie funkcji zwracającej unię błędów może zawierać klauzulę `catch`. Na przykład, biblioteka serwera http może mieć kod, który wygląda następująco:

```zig
action(req, res) catch |err| {
    if (err == error.BrokenPipe or err == error.ConnectionResetByPeer) {
        return;
    } else if (err == error.BodyTooBig) {
        res.status = 431;
        res.body = "Request body is too big";
    } else {
        res.status = 500;
        res.body = "Internal Server Error";
        // todo: log err
    }
};
```

Wersja ze `switch` jest bardziej idiomatyczna:

```zig
action(req, res) catch |err| switch (err) {
    error.BrokenPipe, error.ConnectionResetByPeer) => return,
    error.BodyTooBig => {
        res.status = 431;
        res.body = "Request body is too big";
    },
    else => {
        res.status = 500;
        res.body = "Internal Server Error";
    }
};
```

To wszystko jest dość wymyślne, ale bądźmy szczerzy, najbardziej prawdopodobną rzeczą, jaką zamierzasz zrobić w `catch`, jest przekazanie błędu do wywołującego:

```zig
action(req, res) catch |err| return err;
```

Jest to tak powszechne, że właśnie to robi `try`. Zamiast powyższego, robimy:

```zig
try action(req, res);
```

Jest to szczególnie przydatne, gdy **błąd musi zostać obsłużony**. Najprawdopodobniej zrobisz to za pomocą `try` lub `catch`.

> Programiści Go zauważą, że `try` wymaga mniej naciśnięć klawiszy niż `if err != nil { return err }`.

Przez większość czasu będziesz używać `try` i `catch`, ale unie błędów są również obsługiwane przez `if` i `while`, podobnie jak typy opcjonalne. W przypadku `while`, jeśli warunek zwróci błąd, wykonywana jest klauzula `else`.

Istnieje specjalny typ `anyerror`, który może przechowywać dowolny błąd. Chociaż możemy zdefiniować funkcję jako zwracającą `anyerror!TYPE` zamiast `!TYPE`, te dwa typy nie są równoważne. Wywnioskowany zestaw błędów jest tworzony na podstawie tego, co funkcja może zwrócić. `anyerror` jest globalnym zestawem błędów, podzbiorem wszystkich zestawów błędów w programie. Dlatego użycie `anyerror` w sygnaturze funkcji może sygnalizować, że funkcja może zwracać błędy, których w rzeczywistości nie może. `anyerror` jest używany dla parametrów funkcji lub pól struktury, które mogą działać z dowolnym błędem (wyobraź sobie bibliotekę logowania).

Nierzadko zdarza się, że funkcja zwraca typ opcjonalny unii błędów. Z wywnioskowanym zestawem błędów wygląda to następująco:

```zig
// wczytanie ostatnio zapisanej gry
pub fn loadLast() !?Save {
    // TODO
    return null;
}
```

Istnieją różne sposoby korzystania z takich funkcji, ale najbardziej kompaktowym jest użycie `try` do rozpakowania naszego błędu, a następnie `orelse` do rozpakowania opcjonalnego. Oto działający szkielet:

```zig
const std = @import("std");

pub fn main() void {
    // To jest linia, na której chcesz się skupić
    const save = (try Save.loadLast()) orelse Save.blank();
    std.debug.print("{any}\n", .{save});
}

pub const Save = struct {
    lives: u8,
    level: u16,

    pub fn loadLast() !?Save {
        //todo
        return null;
    }

    pub fn blank() Save {
        return .{
            .lives = 3,
            .level = 1,
        };
    }
};
```

Podczas gdy Zig ma większą głębię, a niektóre funkcje języka mają większe możliwości, to co widzieliśmy w tych dwóch pierwszych częściach jest znaczącą częścią języka. Będzie to służyć jako podstawa, pozwalając nam odkrywać bardziej złożone tematy bez zbytniego rozpraszania się składnią.
