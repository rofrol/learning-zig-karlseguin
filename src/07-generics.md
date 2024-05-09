# Generyki (generics)

W poprzedniej części zbudowaliśmy tablicę dynamiczną o nazwie `IntList`. Celem tej struktury danych było przechowywanie dynamicznej liczby wartości. Chociaż algorytm, którego użyliśmy, działałby dla dowolnego typu danych, nasza implementacja była powiązana z wartościami `i64`. Z pomocą przychodzą generyki, których celem jest abstrahowanie algorytmów i struktur danych od konkretnych typów.

Wiele języków implementuje generyki ze specjalną składnią i regułami specyficznymi dla generyki. W Zigu generyki są mniej specyficzną cechą, a bardziej wyrazem tego, do czego zdolny jest język. W szczególności, generyki wykorzystują potężne metaprogramowanie Ziga w czasie kompilacji.

Zaczniemy od głupiego przykładu, aby się zorientować:

```zig
const std = @import("std");

pub fn main() !void {
    var arr: IntArray(3) = undefined;
    arr[0] = 1;
    arr[1] = 10;
    arr[2] = 100;
    std.debug.print("{any}\n", .{arr});
}

fn IntArray(comptime length: usize) type {
    return [length]i64;
}
```

Powyższe wypisuje `{ 1, 10, 100 }`. Interesujące jest to, że mamy funkcję, która zwraca `type` (stąd funkcja jest PascalCase). I to nie byle jaki typ, ale typ oparty na parametrze funkcji. Ten kod działa tylko dlatego, że zadeklarowaliśmy `length` jako `comptime`. Oznacza to, że wymagamy, aby każdy, kto wywołuje `IntArray`, przekazał parametr `length` znany w czasie kompilacji. Jest to konieczne, ponieważ nasza funkcja zwraca `type`, a typy muszą być zawsze znane w czasie kompilacji.

Funkcja może zwracać _dowolny_ typ, nie tylko typy podstawowe i tablice. Na przykład, wprowadzając niewielką zmianę, możemy sprawić, że funkcja będzie zwracać strukturę:

```zig
const std = @import("std");

pub fn main() !void {
    var arr: IntArray(3) = undefined;
    arr.items[0] = 1;
    arr.items[1] = 10;
    arr.items[2] = 100;
    std.debug.print("{any}\n", .{arr.items});
}

fn IntArray(comptime length: usize) type {
    return struct {
        items: [length]i64,
    };
}
```

Może się to wydawać dziwne, ale typ `arr` to tak naprawdę `IntArray(3)`. Jest to typ jak każdy inny typ, a `arr` jest wartością jak każda inna wartość. Gdybyśmy wywołali `IntArray(7)`, byłby to inny typ. Może uda nam się to zrobić schludniej:

```zig
const std = @import("std");

pub fn main() !void {
    var arr = IntArray(3).init();
    arr.items[0] = 1;
    arr.items[1] = 10;
    arr.items[2] = 100;
    std.debug.print("{any}\n", .{arr.items});
}

fn IntArray(comptime length: usize) type {
    return struct {
        items: [length]i64,

        fn init() IntArray(length) {
            return .{
                .items = undefined,
            };
        }
    };
}
```

Na pierwszy rzut oka może to nie wyglądać schludniej. Ale poza tym, że jest bezimienna i zagnieżdżona w funkcji, nasza struktura wygląda jak każda inna struktura, którą widzieliśmy do tej pory. Ma pola, ma funkcje. Wiesz, co mówią, _jeśli wygląda jak kaczka..._. Cóż, ta struktura wygląda, pływa i kwacze jak normalna struktura, ponieważ nią jest.

Podjęliśmy tę drogę, aby poczuć się komfortowo z funkcją zwracającą typ i towarzyszącą jej składnią. Aby uzyskać bardziej typowy generyk, musimy wprowadzić ostatnią zmianę: nasza funkcja musi przyjmować `type`. W rzeczywistości jest to niewielka zmiana, ale `type` może wydawać się bardziej abstrakcyjny niż `usize`, więc zrobiliśmy to powoli. Zróbmy krok naprzód i zmodyfikujmy naszą poprzednią funkcję `IntList`, aby działała z dowolnym typem. Zaczniemy od szkieletu:

```zig
fn List(comptime T: type) type {
    return struct {
        pos: usize,
        items: []T,
        allocator: Allocator,

        fn init(allocator: Allocator) !List(T) {
            return .{
                .pos = 0,
                .allocator = allocator,
                .items = try allocator.alloc(T, 4),
            };
        }
    };
}
```

Powyższa struktura jest prawie identyczna z naszą `IntList`, z wyjątkiem tego, że `i64` zostało zastąpione przez `T`. To `T` może wydawać się wyjątkowe, ale to tylko nazwa zmiennej. Mogliśmy ją nazwać `item_type`. Jednakże, zgodnie z konwencją nazewnictwa Ziga, zmienne typu `type` są PascalCase.

> Na dobre i na złe, używanie pojedynczej litery do reprezentowania parametru typu jest znacznie starsze niż Zig. `T` jest powszechną wartością domyślną w większości języków, ale można spotkać się z odmianami specyficznymi dla kontekstu, takimi jak hashmapy używające `K` i `V` dla klucza i wartości jako typów parametrów.

Jeśli nie jesteś pewien naszego szkieletu, rozważ dwa miejsca, w których używamy `T`: `items: []T i allocator.alloc(T, 4)`. Gdy będziemy chcieli użyć tego typu generycznego, utworzymy jego instancję przy użyciu:

```zig
var list = try List(u32).init(allocator);
```

Gdy kod zostanie skompilowany, kompilator utworzy nowy typ, znajdując każdy `T` i zastępując go `u32`. Jeśli ponownie użyjemy `List(u32)`, kompilator ponownie użyje utworzonego wcześniej typu. Jeśli określimy nową wartość dla `T`, powiedzmy `List(bool)` lub `List(User)`, zostaną utworzone nowe typy.

Aby ukończyć naszą generyczną `List`, możemy dosłownie skopiować i wkleić resztę kodu `IntList` i zastąpić `i64` przez `T`. Oto w pełni działający przykład:

```zig
const std = @import("std");
const Allocator = std.mem.Allocator;

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var list = try List(u32).init(allocator);
    defer list.deinit();

    for (0..10) |i| {
        try list.add(@intCast(i));
    }

    std.debug.print("{any}\n", .{list.items[0..list.pos]});
}

fn List(comptime T: type) type {
    return struct {
        pos: usize,
        items: []T,
        allocator: Allocator,

        fn init(allocator: Allocator) !List(T) {
            return .{
                .pos = 0,
                .allocator = allocator,
                .items = try allocator.alloc(T, 4),
            };
        }

        fn deinit(self: List(T)) void {
            self.allocator.free(self.items);
        }

        fn add(self: *List(T), value: T) !void {
            const pos = self.pos;
            const len = self.items.len;

            if (pos == len) {
        // zabrakło nam miejsca
        // utwórz nowy wycinek, który jest dwa razy większy
                var larger = try self.allocator.alloc(T, len * 2);

        // skopiuj elementy, które wcześniej dodaliśmy do naszej nowej przestrzeni
                @memcpy(larger[0..len], self.items);

                self.allocator.free(self.items);

                self.items = larger;
            }

            self.items[pos] = value;
            self.pos = pos + 1;
        }
    };
}
```

Nasza funkcja `init` zwraca `List(T)`, a nasze funkcje `deinit` i `add` pobierają `List(T)` i `*List(T)`. W naszej prostej klasie jest to w porządku, ale w przypadku dużych struktur danych pisanie pełnej nazwy ogólnej może stać się nieco uciążliwe, zwłaszcza jeśli mamy wiele parametrów typu (np. hashmapa, która przyjmuje oddzielny `type` dla swojego klucza i wartości). Wbudowana funkcja `@This()` zwraca najbardziej wewnętrzny `type`, z którego została wywołana. Najprawdopodobniej nasza funkcja `List(T)` zostałaby zapisana jako:

```zig
fn List(comptime T: type) type {
    return struct {
        pos: usize,
        items: []T,
        allocator: Allocator,

        // Dodano
        const Self = @This();

        fn init(allocator: Allocator) !Self {
        // ... ten sam kod
        }

        fn deinit(self: Self) void {
        // ... ten sam kod
        }

        fn add(self: *Self, value: T) !void {
        // ... ten sam kod
        }
    };
}
```

`Self` nie jest specjalną nazwą, jest po prostu zmienną i jest PascalCase, ponieważ jej wartość to `type`. Możemy użyć `Self` tam, gdzie wcześniej używaliśmy `List(T)`.

---

Możemy tworzyć bardziej złożone przykłady, z wieloma parametrami typu i bardziej zaawansowanymi algorytmami. Ale ostatecznie podstawowy kod generyczny nie różniłby się od prostych przykładów powyżej. W następnej części ponownie dotkniemy generyczności, gdy przyjrzymy się standardowym bibliotekom `ArrayList(T)` i `StringHashMap(V)`.
