## Przewodnik po stylach

W tej krótkiej części omówimy dwie zasady kodowania wymuszane przez kompilator, a także konwencję nazewnictwa biblioteki standardowej.

## Nieużywane zmienne

Zig nie zezwala na zostawienie nieużytych zmiennych. Poniższy przykład daje dwa błędy kompilacji:

```zig
const std = @import("std");

pub fn main() void {
    const sum = add(8999, 2);
}

fn add(a: i64, b: i64) i64 {
    // zauważ, że to jest a + a, a nie a + b
    return a + a;
}
```

Pierwszy błąd wynika z faktu, że `sum` jest _nieużywaną stałą lokalną_. Drugi błąd wynika z faktu, że `b` jest nieużywanym parametrem funkcji. W przypadku tego kodu są to oczywiste błędy. Możesz jednak mieć uzasadnione powody, aby mieć nieużywane zmienne i parametry funkcji. W takich przypadkach można przypisać zmienne do podkreślenia (`_`):

```zig
const std = @import("std");

pub fn main() void {
    _ = add(8999, 2);

    // lub

    const sum = add(8999, 2);
    _ = sum;
}

fn add(a: i64, b: i64) i64 {
    _ = b;
    return a + a;
}
```

Jako alternatywę dla `_ = b;`, mogliśmy nazwać parametr funkcji `_`, choć moim zdaniem pozostawia to czytelnikowi domysły, czym jest nieużywany parametr:

```zig
fn add(a: i64, _: i64) i64 {
```

Zauważ, że `std` jest również nieużywany, ale nie generuje błędu. W pewnym momencie w przyszłości należy oczekiwać, że Zig potraktuje to również jako błąd czasu kompilacji.

## Przesłanianie (shadowing)

Zig nie pozwala, by jeden identyfikator "ukrywał" inny, używając tej samej nazwy. Ten kod do odczytu z gniazda jest nieprawidłowy:

```zig
fn read(stream: std.net.Stream) ![]const u8 {
    var buf: [512]u8 = undefined;
    const read = try stream.read(&buf);
    if (read == 0) {
        return error.Closed;
    }
    return buf[0..read];
}
```

Nasza zmienna `read` przesłania nazwę naszej funkcji. Nie jestem fanem tej zasady, ponieważ zazwyczaj prowadzi ona programistów do używania krótkich, bezsensownych nazw. Na przykład, aby skompilować ten kod, zmieniłbym `read` na `n`. Jest to przypadek, w którym moim zdaniem programiści są w znacznie lepszej pozycji, aby wybrać najbardziej czytelną opcję.

## Konwencja nazewnictwa

Poza regułami narzuconymi przez kompilator, możesz oczywiście stosować dowolną konwencję nazewnictwa. Pomocne jest jednak zrozumienie konwencji nazewnictwa Ziga, ponieważ większość kodu, z którym będziesz wchodzić w interakcje, od biblioteki standardowej po biblioteki stron trzecich, korzysta z niej.

Kod źródłowy Ziga jest wcięty 4 spacjami. Osobiście używam tabulatora, który jest obiektywnie lepszy dla dostępności.

Nazwy funkcji są camelCase, a zmienne lowercase_with_underscores (zwane snake case). Typy są pisane PascalCase. Istnieje interesujące skrzyżowanie tych trzech reguł. Zmienne, które odwołują się do typu lub funkcje, które zwracają typ, są zgodne z regułą typu i są PascalCase. Już to widzieliśmy, ale mogłeś to przegapić.

```zig
std.debug.print("{any}\n", .{@TypeOf(.{.year = 2023, .month = 8})});
```

Widzieliśmy już inne wbudowane funkcje: `@import`, `@rem` i `@intCast`. Ponieważ są to funkcje, są one pisane camelCase. `@TypeOf` jest również wbudowaną funkcją, ale jest to PascalCase, dlaczego? Ponieważ zwraca typ, a zatem używana jest konwencja nazewnictwa typów. Gdybyśmy chcieli przypisać wynik `@TypeOf` do zmiennej, używając konwencji nazewnictwa Ziga, zmienna ta również powinna być PascalCase:

```zig
const T = @TypeOf(3)
std.debug.print("{any}\n", .{T});
```

Plik wykonywalny `zig` ma polecenie `fmt`, które, biorąc pod uwagę plik lub katalog, sformatuje plik w oparciu o własny przewodnik stylu Ziga. Nie obejmuje on jednak wszystkiego, na przykład dostosuje wcięcia i pozycje nawiasów klamrowych, ale nie zmieni rozmiaru znaku identyfikatorów.
