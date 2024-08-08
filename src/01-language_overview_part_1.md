# Przegląd języka – część 1

Zig jest silnie typowanym językiem kompilowanym. Obsługuje generyki, ma potężne możliwości metaprogramowania w czasie kompilacji i **nie** zawiera garbage collectora. Wiele osób uważa Ziga za nowoczesną alternatywę dla C. Jako alternatywa ma składnię języka podobną do C. Mówimy o instrukcjach zakończonych średnikiem i blokach ograniczonych nawiasami klamrowymi.

Oto jak wygląda kod Zig:

```zig
const std = @import("std");

// Ten kod nie skompiluje się, jeśli `main` nie jest `pub` (publiczny)
pub fn main() void {
    const user = User{
        .power = 9001,
        .name = "Goku",
    };

    std.debug.print("{s}'s power is {d}", .{user.name, user.power});
}

pub const User = struct {
    power: u64,
    name: []const u8,
};
```

Jeśli zapiszesz powyższe jako _learning.zig_ i uruchomisz `zig run learning.zig`, powinieneś zobaczyć: `Goku's power is 9001`.

Jest to prosty przykład, gdzie możesz podążać za kodem, nawet jeśli pierwszy raz widzisz Ziga. Mimo to, przejrzymy go linijka po linijce.

> Zobacz sekcję dotyczącą [instalacji Ziga](https://www.openmymind.net/learning_zig/#install), aby szybko rozpocząć pracę.

## Importowanie

Bardzo niewiele programów jest napisanych jako pojedynczy plik bez standardowej biblioteki lub bibliotek zewnętrznych. Nasz pierwszy program nie jest wyjątkiem i wykorzystuje standardową bibliotekę Ziga do wypisania naszych danych wyjściowych. System importu Ziga jest prosty i opiera się na funkcji `@import` i słowie kluczowym `pub` (aby kod był dostępny poza bieżącym plikiem).

> Funkcje zaczynające się od `@` są funkcjami wbudowanymi. Są one dostarczane przez kompilator, a nie przez bibliotekę standardową.

Importujemy moduł określając jego nazwę. Standardowa biblioteka Ziga jest dostępna przy użyciu nazwy "std". Aby zaimportować określony plik, używamy jego ścieżki względem pliku wykonującego import. Na przykład, jeśli przenieśliśmy strukturę `User` do jej własnego pliku, powiedzmy _models/user.zig_:

```zig
// models/user.zig
pub const User = struct {
    power: u64,
    name: []const u8,
};
```

Następnie zaimportujemy go za pośrednictwem:

```zig
// main.zig
const User = @import("models/user.zig").User;
```

> Jeśli nasza _struktura_ `User` nie została oznaczona jako `pub`, otrzymamy następujący błąd: _'User' is not marked 'pub'_.

_models/user.zig_ może eksportować więcej niż jedną rzecz. Na przykład, możemy również wyeksportować stałą:

```zig
// models/user.zig
pub const MAX_POWER = 100_000;

pub const User = struct {
    power: u64,
    name: []const u8,
};
```

W takim przypadku moglibyśmy zaimportować oba:

```zig
const user = @import("models/user.zig");
const User = user.User;
const MAX_POWER = user.MAX_POWER;
```

W tym momencie możesz mieć więcej pytań niż odpowiedzi. Czym jest `user` w powyższym fragmencie? Jeszcze tego nie widzieliśmy, ale co jeśli użyjemy `var` zamiast `const`? A może zastanawiasz się, jak korzystać z bibliotek stron trzecich. To wszystko są dobre pytania, ale aby na nie odpowiedzieć, musimy najpierw dowiedzieć się więcej o Zigu. Na razie będziemy musieli zadowolić się tym, czego się nauczyliśmy: jak importować standardową bibliotekę Ziga, jak importować inne pliki i jak eksportować definicje.

## Komentarze

Następna linia naszego przykładu Zig jest komentarzem:

```zig
// Ten kod nie skompiluje się, jeśli `main` nie jest `pub` (publiczny)
```

Zig nie posiada wieloliniowych komentarzy, tak jak `/* ... */` w C.

Istnieje eksperymentalne wsparcie dla automatycznego generowania dokumentów na podstawie komentarzy. Jeśli widziałeś [dokumentację biblioteki standardowej](https://ziglang.org/documentation/master/std) Zig, to widziałeś to w akcji. `//!` jest znany jako komentarz dokumentu najwyższego poziomu i może być umieszczony na początku pliku. Komentarz z potrójnym ukośnikiem (`///`), znany jako komentarz dokumentu, może być umieszczony w określonych miejscach, na przykład przed deklaracją. Próba użycia któregokolwiek typu komentarza dokumentu w niewłaściwym miejscu spowoduje błąd kompilatora.

## Funkcje

Następny wiersz kodu jest początkiem naszej głównej funkcji:

```zig
pub fn main() void
```

Każdy plik wykonywalny potrzebuje funkcji o nazwie `main`: jest to punkt wejścia do programu. Gdybyśmy zmienili nazwę `main` na coś innego, na przykład `doIt`, i spróbowali uruchomić `zig run learning.zig`, otrzymalibyśmy błąd informujący, że `'learning' has no member named 'main'`.

Pomijając specjalną rolę `main` jako punktu wejścia naszego programu, jest to naprawdę podstawowa funkcja: nie przyjmuje żadnych parametrów i nic nie zwraca, czyli `void`. Poniższy przykład jest _nieco_ bardziej interesujący:

```zig
const std = @import("std");

pub fn main() void {
    const sum = add(8999, 2);
    std.debug.print("8999 + 2 = {d}\n", .{sum});
}

fn add(a: i64, b: i64) i64 {
    return a + b;
}
```

Programiści C i C++ zauważą, że Zig nie wymaga wcześniejszej deklaracji, tj. `add` jest wywoływany przed jego zdefiniowaniem.

Kolejną rzeczą, na którą należy zwrócić uwagę, jest typ `i64`: 64-bitowa liczba całkowita ze znakiem. Inne typy liczbowe to: `u8`, `i8`, `u16`, `i16`, `u32`, `i32`, `u47`, `i47`, `u64`, `i64`, `f32` i `f64`. Włączenie `u47` i `i47` nie jest testem, aby upewnić się, że nadal nie śpisz; Zig obsługuje liczby całkowite o dowolnej szerokości bitowej. Chociaż prawdopodobnie nie będziesz ich często używać, mogą się przydać. Jednym z często _używanych_ typów jest `usize`, który jest liczbą całkowitą bez znaku o rozmiarze wskaźnika i ogólnie typem reprezentującym długość/rozmiar czegoś.

> Oprócz `f32` i `f64`, Zig obsługuje również typy zmiennoprzecinkowe `f16`, `f80` i `f128`.

Chociaż nie ma dobrego powodu, aby to robić, jeśli zmienimy implementację `add` na:

```zig
fn add(a: i64, b: i64) i64 {
    a += b;
    return a;
}
```

Otrzymamy błąd na `a += b;`: _cannot assign to constant_. Jest to ważna lekcja, do której wrócimy bardziej szczegółowo później: parametry funkcji są stałymi.

Ze względu na lepszą czytelność, nie ma przeciążania funkcji (ta sama funkcja zdefiniowana z różnymi typami parametrów i/lub liczbą parametrów). Na razie to wszystko, co musimy wiedzieć o funkcjach.

## Struktury _(struct)_

Następną linią kodu jest utworzenie typu `User`, który jest zdefiniowany na końcu naszego snippetu. Definicja `User` to:

```zig
pub const User = struct {
    power: u64,
    name: []const u8,
};
```

> Ponieważ nasz program jest pojedynczym plikiem, a zatem `User` jest używany tylko w pliku, w którym jest zdefiniowany, nie musieliśmy go robić `pub`. Ale wtedy nie zobaczylibyśmy, jak wyeksponować deklarację innym plikom.

Pola `struct` są zakończone przecinkiem i mogą mieć wartość domyślną::

```zig
pub const User = struct {
    power: u64 = 0,
    name: []const u8,
};
```

Kiedy tworzymy strukturę, **każde** pole musi być ustawione. Na przykład w oryginalnej definicji, w której `power` nie miało wartości domyślnej, wystąpiłby następujący błąd: _missing struct field: power_.

```zig
const user = User{.name = "Goku"};
```

Jednak z naszą domyślną wartością, powyższe kompiluje się dobrze.

Struktury mogą mieć metody, mogą zawierać deklaracje (w tym inne struktury), a nawet mogą zawierać zero pól, w którym to momencie działają bardziej jak przestrzeń nazw.

```zig
pub const User = struct {
    power: u64 = 0,
    name: []const u8,

    pub const SUPER_POWER = 9000;

    pub fn diagnose(user: User) void {
        if (user.power >= SUPER_POWER) {
            std.debug.print("it's over {d}!!!", .{SUPER_POWER});
        }
    }
};
```

Metody to zwykłe funkcje, które można wywołać za pomocą składni kropki. Oba te sposoby działają:

```zig
// wywołaj diagnose na userze
user.diagnose();

// Powyższe jest cukrem składniowym dla:
User.diagnose(user);
```

Przez większość czasu będziesz używać składni kropki, ale metody jako cukier składniowy nad zwykłymi funkcjami mogą się przydać.

> Instrukcja `if` jest pierwszym przepływem sterowania, który widzieliśmy. To całkiem proste, prawda? Zbadamy to bardziej szczegółowo w następnej części.

`diagnose` jest zdefiniowana w naszym typie `User` i akceptuje `User` jako pierwszy parametr. W związku z tym możemy wywołać ją za pomocą składni kropki. Ale funkcje wewnątrz struktury _nie muszą_ podążać za tym wzorcem. Jednym z typowych przykładów jest funkcja `init` inicjująca naszą strukturę:

```zig
pub const User = struct {
    power: u64 = 0,
    name: []const u8,

    pub fn init(name: []const u8, power: u64) User {
        return User{
            .name = name,
            .power = power,
        };
    }
}
```

Użycie `init` jest jedynie konwencją i w niektórych przypadkach `open` lub inna nazwa może mieć więcej sensu. Jeśli jesteś podobny do mnie i nie jesteś programistą C++, składnia inicjalizacji pól, `.$field = $value`, może być nieco dziwna, ale szybko się do niej przyzwyczaisz.

Kiedy utworzyliśmy "Goku", zadeklarowaliśmy zmienną `user` jako `const`:

```zig
const user = User{
    .power = 9001,
    .name = "Goku",
};
```

Oznacza to, że nie możemy modyfikować `user`. Aby zmodyfikować zmienną, należy ją zadeklarować za pomocą `var`. Być może zauważyłeś również, że typ `user` jest wnioskowany na podstawie tego, co jest do niego przypisane. Moglibyśmy być jawni:

```zig
const user: User = User{
    .power = 9001,
    .name = "Goku",
};
```

Takie użycie jest jednak dość nietypowe. Jednym z miejsc, w których jest to bardziej powszechne, jest zwracanie struktury z funkcji. Tutaj typ można wywnioskować z typu zwracanego przez funkcję. Nasza funkcja `init` prawdopodobnie zostałaby napisana w ten sposób:

```zig
pub fn init(name: []const u8, power: u64) User {
    // zamiast zwracać User{...}
    return .{
        .name = name,
        .power = power,
    };
}
```

Jak przypadku większości rzeczy, które do tej pory zbadaliśmy, w przyszłości powrócimy do struktur, gdy będziemy mówić o innych częściach języka. Ale w przeważającej części są one proste.

## Tablice _(arrays)_ i wycinki _(slices)_

Moglibyśmy pominąć ostatnią linię naszego kodu, ale biorąc pod uwagę, że nasz mały fragment zawiera dwa łańcuchy, **"Goku"** i **"{s}'s power is {d}\n"**, prawdopodobnie jesteś ciekawy łańcuchów w Zigu. Aby lepiej zrozumieć łańcuchy, najpierw zbadajmy tablice i wycinki.

Tablice mają stały rozmiar i długość znaną w czasie kompilacji. Długość jest częścią typu, więc tablica 4 liczb całkowitych ze znakiem, `[4]i32`, jest innego typu niż tablica 5 liczb całkowitych ze znakiem, `[5]i32`.

Długość tablicy można wywnioskować z inicjalizacji. W poniższym kodzie wszystkie trzy zmienne są typu `[5]i32`:

```zig
const a = [5]i32{1, 2, 3, 4, 5};

// widzieliśmy już tę składnię .{...} ze strukturami
// działa to również z tablicami
const b: [5]i32 = .{1, 2, 3, 4, 5};

// użyj _, aby pozwolić kompilatorowi wywnioskować długość
const c = [_]i32{1, 2, 3, 4, 5};
```

Z drugiej strony, _wycinek_ jest wskaźnikiem do tablicy o określonej długości. Długość jest znana w czasie wykonywania. Wskaźniki omówimy w późniejszej części, ale można myśleć o _wycinku_ jako o widoku tablicy.

> Jeśli jesteś zaznajomiony z Go, być może zauważyłeś, że wycinki w Zigu są nieco inne: nie mają pojemności, a jedynie wskaźnik i długość.

Biorąc pod uwagę następujące,

```zig
const a = [_]i32{1, 2, 3, 4, 5};
const b = a[1..4];
```

Chciałbym móc powiedzieć, że `b` jest wycinkiem o długości 3 i wskaźnikiem do `a`. Ale ponieważ "pokroiliśmy" naszą tablicę przy użyciu wartości znanych w czasie kompilacji, tj. `1` i `4`, nasza długość, `3`, jest również znana w czasie kompilacji. Zig rozgryzł to wszystko i dlatego `b` nie jest wycinkiem, ale raczej wskaźnikiem do tablicy liczb całkowitych o długości 3. Konkretnie, jego typ to `*const [3]i32`. Tak więc ta demonstracja wycinka została udaremniona przez spryt Ziga.

W prawdziwym kodzie prawdopodobnie będziesz używał wycinków częściej niż tablic. Na dobre i na złe, programy mają tendencję do posiadania większej ilości informacji w czasie wykonania (runtime) niż w czasie kompilacji (compile time). W tym małym przykładzie musimy jednak oszukać kompilator, aby uzyskać to, czego chcemy:

```zig
const a = [_]i32{1, 2, 3, 4, 5};
var end: usize = 3;
end += 1;
const b = a[1..end];
```

`b` jest teraz prawidłowym wycinkiem, a konkretnie jego typem jest `[]const i32`. Można zauważyć, że długość wycinka nie jest częścią typu, ponieważ długość jest właściwością czasu wykonania, a typy są zawsze w pełni znane w czasie kompilacji. Podczas tworzenia wycinka możemy pominąć górną granicę, aby utworzyć wycinek do końca tego, co kroimy (tablicy lub wycinka), np. `const c = b[2...]`;.

> Gdybyśmy zrobili `const end: usize = 4` bez inkrementacji, to `1...end` stałoby się znaną w czasie kompilacji długością dla `b`, a tym samym utworzyłoby wskaźnik do tablicy, a nie wycinek. Uważam, że jest to trochę mylące, ale nie jest to coś, co pojawia się zbyt często i nie jest zbyt trudne do opanowania. Chciałbym pominąć to w tym momencie, ale nie mogłem znaleźć uczciwego sposobu na uniknięcie tego szczegółu.

Nauka Ziga nauczyła mnie, że typy są bardzo opisowe. To nie tylko liczba całkowita lub logiczna, czy nawet tablica 32-bitowych liczb całkowitych ze znakiem. Typy zawierają również inne ważne informacje. Rozmawialiśmy o tym, że długość jest częścią typu tablicy, a wiele przykładów pokazało, że stałość jest również jego częścią. Na przykład, w naszym ostatnim przykładzie, typem `b` jest `[]const i32`. Można to zobaczyć na przykładzie poniższego kodu:

```zig
const std = @import("std");

pub fn main() void {
    const a = [_]i32{1, 2, 3, 4, 5};
    var end: usize = 4;
    end += 1;
    const b = a[1..end];
    std.debug.print("{any}", .{@TypeOf(b)});
}
```

Gdybyśmy próbowali wpisać do `b`, np. `b[2] = 5;` otrzymalibyśmy błąd kompilacji: _cannot assign to constant_. Jest to spowodowane typem `b`.

Aby rozwiązać ten problem, można pokusić się o wprowadzenie następującej zmiany:

```zig
// zamień const na var
var b = a[1..end];
```

ale otrzymasz ten sam błąd, dlaczego? Jako podpowiedź, jaki jest typ `b`, lub bardziej ogólnie, czym jest `b`? Wycinek jest długością i wskaźnikiem do [części] tablicy. Typ wycinka jest zawsze pochodną tego, co jest wycinane. Niezależnie od tego, czy `b` jest zadeklarowana jako stała, czy nie, jest to wycinek `[5]const i32`, więc `b` musi być typu `[]const i32`. Jeśli chcemy mieć możliwość zapisu do `b`, musimy zmienić `a` z `const` na `var`.

```zig
const std = @import("std");

pub fn main() void {
    var a = [_]i32{1, 2, 3, 4, 5};
    var end: usize = 3;
    end += 1;
    const b = a[1..end];
    b[2] = 99;
}
```

Działa to, ponieważ nasz wycinek nie jest już `[]const i32`, ale raczej `[]i32`. Można się zastanawiać, dlaczego to działa, skoro `b` wciąż jest stałą. Ale stałość `b` odnosi się do samego `b`, a nie do danych, na które `b` wskazuje. Cóż, nie jestem pewien, czy to świetne wyjaśnienie, ale dla mnie ten kod podkreśla różnicę:

```zig
const std = @import("std");

pub fn main() void {
    var a = [_]i32{1, 2, 3, 4, 5};
    var end: usize = 3;
    end += 1;
    const b = a[1..end];
    b = b[1..];
}
```

To się nie skompiluje; jak mówi nam kompilator, _cannot assign to constant_. Ale jeśli zrobilibyśmy `var b = a[1..end];`, kod zadziałałby, ponieważ samo `b` nie jest już stałą.

Więcej o tablicach i wycinkach dowiemy się przyglądając się innym aspektom języka, z których łańcuchy nie są najmniej ważnym.

## Łańcuchy _(strings)_

Chciałbym móc powiedzieć, że Zig ma typ łańcuch i że jest niesamowity. Niestety tak nie jest. W najprostszym ujęciu, łańcuchy Ziga są sekwencjami (tj. tablicami lub wycinkami) bajtów (`u8`). Widzieliśmy to w definicji pola `name`: `name: []const u8,`.

Zgodnie z konwencją, i tylko zgodnie z konwencją, takie łańcuchy powinny zawierać tylko wartości UTF-8, ponieważ kod źródłowy Ziga jest sam w sobie zakodowany w UTF-8. Ale nie jest to egzekwowane i tak naprawdę nie ma różnicy między `[]const u8`, który reprezentuje łańcuch ASCII lub UTF-8, a `[]const u8`, który reprezentuje dowolne dane binarne. Jak mogłoby być inaczej, są tego samego typu.

Z tego, czego nauczyliśmy się o tablicach i wycinkach, można się domyślić, że `[]const u8` jest wycinkiem do stałej tablicy bajtów (gdzie bajt jest 8-bitową liczbą całkowitą bez znaku). Ale nigdzie w naszym kodzie nie wycięliśmy tablicy, ani nawet nie mieliśmy tablicy, prawda? Wszystko, co zrobiliśmy, to przypisanie "Goku" do `user.name`. Jak to zadziałało?

Literały łańcuchowe, te które widzisz w kodzie źródłowym, mają znaną długość w czasie kompilacji. Kompilator wie, że "Goku" ma długość 4. Można by więc pomyśleć, że "Goku" najlepiej reprezentuje tablica, coś w rodzaju `[4]const u8`. Ale literały łańcuchowe mają kilka specjalnych właściwości. Są one przechowywane w specjalnym miejscu w pliku binarnym i deduplikowane. Tak więc zmienna do literału łańcuchowego będzie wskaźnikiem do tej specjalnej lokalizacji. Oznacza to, że typ "Goku" jest bliższy `*const [4]u8`, wskaźnikowi do stałej tablicy 4 bajtów.

To nie wszystko. Literały łańcuchowe są zakończone zerem. Oznacza to, że zawsze mają `\0` na końcu. Łańcuchy zakończone zerem są ważne podczas interakcji z C. W pamięci, "Goku" wyglądałoby tak: `{'G', 'o', 'k', 'u', 0}`, więc można by pomyśleć, że typem jest `*const [5]u8`. Byłoby to jednak w najlepszym przypadku niejednoznaczne, a w gorszym niebezpieczne (można by nadpisać terminator zerowy). Zamiast tego, Zig ma odrębną składnię do reprezentowania tablic zakończonych zerem. "Goku" ma typ: `*const [4:0]u8`, wskaźnik do zakończonej zerem tablicy 4 bajtów. Mówiąc o łańcuchach, skupiamy się na tablicach bajtów zakończonych znakiem null (ponieważ w ten sposób łańcuchy są zwykle reprezentowane w C), składnia jest bardziej ogólna: `[LENGTH:SENTINEL]`, gdzie "SENTINEL" to specjalna wartość znajdująca się na końcu tablicy. Tak więc, chociaż nie mogę wymyślić, dlaczego byłoby to potrzebne, poniższe jest całkowicie poprawne:

```zig
const std = @import("std");

pub fn main() void {
    // tablica 3 wartości logicznych z false jako wartością wartownika
    const a = [3:false]bool{false, true, false};

    // Ta linia jest bardziej zaawansowana i nie zostanie wyjaśniona!
    std.debug.print("{any}\n", .{std.mem.asBytes(&a).*});
}
```

Co daje wynik: `{ 0, 1, 0, 0}`.

> Waham się, czy dołączyć ten przykład, ponieważ ostatnia linia jest dość zaawansowana i nie zamierzam jej wyjaśniać. Z drugiej strony, jest to działający przykład, który możesz uruchomić i pobawić się nim, aby lepiej zbadać trochę z tego, co omówiliśmy do tej pory, jeśli masz taką ochotę.

Jeśli udało mi się to wyjaśnić w zadowalający sposób, prawdopodobnie nadal jest jedna rzecz, której nie jesteś pewien. Jeśli "Goku" jest `*const [4:0]u8`, jak to się stało, że mogliśmy przypisać go do `name`, które jest `[]const u8`? Odpowiedź jest prosta: Zig wymusi typ za ciebie. Zrobi to między kilkoma różnymi typami, ale jest to najbardziej oczywiste w przypadku łańcuchów. Oznacza to, że jeśli funkcja ma parametr `[]const u8` lub struktura ma pole `[]const u8`, można użyć literałów łańcuchowych. Ponieważ łańcuchy zakończone nullem są tablicami, a tablice mają znaną długość, ta koercja jest tania, tj. **nie** wymaga iteracji przez łańcuch w celu znalezienia zakończenia nullem.

Tak więc, mówiąc o łańcuchach, zwykle mamy na myśli `[]const u8`. W razie potrzeby wyraźnie podajemy łańcuch zakończony zerem, który może zostać automatycznie przekształcony w `[]const u8`. Należy jednak pamiętać, że `[]const u8` jest również używany do reprezentowania dowolnych danych binarnych i jako taki, Zig nie ma pojęcia łańcucha, które mają języki programowania wyższego poziomu. Co więcej, biblioteka standardowa Ziga ma tylko bardzo podstawowy moduł unicode.

Oczywiście w prawdziwym programie większość łańcuchów (i bardziej ogólnie, tablic) nie jest znana w czasie kompilacji. Klasycznym przykładem są dane wprowadzane przez użytkownika, które nie są znane podczas kompilacji programu. Jest to coś, do czego będziemy musieli powrócić, mówiąc o pamięci. Ale krótka odpowiedź jest taka, że dla takich danych, które mają nieznaną wartość w czasie kompilacji, a tym samym nieznaną długość, będziemy dynamicznie alokować pamięć w czasie wykonywania. Nasze zmienne łańcuchowe, wciąż typu `[]const u8`, będą wycinkami wskazującymi na tę dynamicznie przydzielaną pamięć.

## comptime i anytype

W naszej ostatniej niezbadanej linii kodu dzieje się o wiele więcej niż na pierwszy rzut oka:

```zig
std.debug.print("{s}'s power is {d}\n", .{user.name, user.power});
```

Prześledzimy go tylko pobieżnie, ale stanowi on okazję do podkreślenia niektórych z bardziej zaawansowanych funkcji Ziga. Są to rzeczy, o których powinieneś przynajmniej wiedzieć, nawet jeśli ich nie opanowałeś.

Pierwszą z nich jest koncepcja wykonywania w czasie kompilacji, czyli `comptime`. Jest to rdzeń możliwości metaprogramowania Ziga i, jak sama nazwa wskazuje, obraca się wokół uruchamiania kodu w czasie kompilacji, a nie w czasie wykonywania. W tym przewodniku tylko zbadamy po łebkach co jest możliwe z `comptime`, ale jest to coś, co stale istnieje.

Być może zastanawiasz się, co takiego jest w powyższej linii, że wymaga ona wykonania w czasie kompilacji. Definicja funkcji `print` wymaga, aby nasz pierwszy parametr, format łańcucha, był znany w czasie kompilacji:

```zig
// zwróć uwagę na "comptime" przed zmienną "fmt"
pub fn print(comptime fmt: []const u8, args: anytype) void {
```

Powodem tego jest to, że `print` wykonuje dodatkowe sprawdzenia w czasie kompilacji, których nie można uzyskać w większości innych języków. Jakiego rodzaju sprawdzenia? Cóż, powiedzmy, że zmieniłeś format na `"it's over {d}\n"`, ale zachowałeś dwa argumenty. Otrzymasz błąd czasu kompilacji: _unused argument in 'it's over {d}'_. Będą również sprawdzenia typu: zmień format łańcuchowy na `"{s}'s power is {s}\n"`, a otrzymasz `invalid format string 's' for type 'u64'`. Te sprawdzenia nie byłyby możliwe do wykonania w czasie kompilacji, gdyby format łańcuchowy nie był znany w czasie kompilacji. Stąd wymóg wartości znanej w czasie kompilacji.

Jedynym miejscem, w którym `comptime` natychmiast wpłynie na twoje kodowanie, są domyślne typy dla literałów całkowitych i zmiennoprzecinkowych, specjalne `comptime_int` i `comptime_float`. Ten wiersz kodu jest nieprawidłowy: `var i = 0;`. Otrzymasz błąd kompilacji: _variable of type 'comptime_int' must be const or comptime_. Kod `comptime` może działać tylko z danymi, które są znane w czasie kompilacji, a dla liczb całkowitych i zmiennoprzecinkowych takie dane są identyfikowane przez specjalne typy `comptime_int` i `comptime_float`. Wartość tego typu może być użyta w czasie wykonywania kompilacji. Prawdopodobnie jednak nie będziesz spędzać większości czasu na pisaniu kodu do wykonania w czasie kompilacji, więc nie jest to szczególnie przydatna wartość domyślna. To, co musisz zrobić, to nadać zmiennym jawny typ:

```zig
var i: usize = 0;
var j: f64 = 0;
```

> Zauważ, że ten błąd wystąpił tylko dlatego, że użyliśmy `var`. Gdybyśmy użyli `const`, nie mielibyśmy błędu, ponieważ cała istota błędu polega na tym, że `comptime_int` musi być `const`.

W przyszłej części przyjrzymy się nieco bliżej `comptime` podczas eksploracji generyczności.

Inną szczególną rzeczą w naszej linii kodu jest dziwne `.{user.name, user.power}`, które, jak wiemy z powyższej definicji `print`, odwzorowuje na zmienną typu `anytype`. Typ ten nie powinien być mylony z czymś takim jak `Object` w Javie lub `any` w Go (znany jako `interface{}`). Zamiast tego, w czasie kompilacji, Zig utworzy wersję funkcji `print` specjalnie dla wszystkich typów, które zostały do niej przekazane.

Nasuwa się pytanie: co do niej _przekazujemy_? Notację `.{...}` widzieliśmy już wcześniej, gdy pozwalaliśmy kompilatorowi wnioskować o typie naszej struktury. Tu jest podobnie: tworzy literał anonimowej struktury. Rozważmy ten kod:

```zig
pub fn main() void {
    std.debug.print("{any}\n", .{@TypeOf(.{.year = 2023, .month = 8})});
}
```

który wypisuje:

```
struct{comptime year: comptime_int = 2023, comptime month: comptime_int = 8}
```

Tutaj nadaliśmy naszej anonimowej strukturze nazwy pól, `year` i `month`. W naszym oryginalnym kodzie tego nie zrobiliśmy. W takim przypadku nazwy pól są generowane automatycznie jako "0", "1", "2" itd. Chociaż oba są przykładami literału anonimowej struktury, ta bez nazw pól jest często nazywana _krotką_. Funkcja `print` oczekuje krotki i używa pozycji porządkowej w formacie łańcuchowym, aby uzyskać odpowiedni argument.

Zig nie ma przeciążania funkcji i nie ma funkcji vardiadic (funkcji ze zmienną liczbą argumentów). Posiada jednak kompilator zdolny do tworzenia wyspecjalizowanych funkcji w oparciu o przekazane typy, w tym typy wywnioskowane i utworzone przez sam kompilator.
