# Wskaźniki (pointers)

Zig nie zawiera garbage collectora. Ciężar zarządzania pamięcią spoczywa na programiście. To duża odpowiedzialność, ponieważ ma bezpośredni wpływ na wydajność, stabilność i bezpieczeństwo aplikacji.

Zaczniemy od omówienia wskaźników, co jest ważnym tematem do omówienia samym w sobie, ale także do rozpoczęcia szkolenia w zakresie postrzegania danych naszego programu z punktu widzenia pamięci. Jeśli jesteś już zaznajomiony ze wskaźnikami, alokacjami sterty i zwisającymi wskaźnikami, możesz pominąć kilka części do [pamięci sterty i alokatorów](06-heap_memory_and_allocators.md), które są bardziej specyficzne dla Zig.

---

Poniższy kod tworzy użytkownika o mocy (`power`) 100, a następnie wywołuje funkcję `levelUp`, która zwiększa moc użytkownika o 1. Czy potrafisz odgadnąć wynik?

```zig
const std = @import("std");

pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};

  // ta linia została dodana
	levelUp(user);
	std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

fn levelUp(user: User) void {
	user.power += 1;
}

pub const User = struct {
	id: u64,
	power: i32,
};
```

To była niemiła sztuczka; kod się nie skompiluje: zmienna lokalna nigdy nie jest mutowana. Jest to odniesienie do zmiennej `user` w `main`. Zmienna, która nigdy nie jest mutowana, musi być zadeklarowana jako `const`. Możesz pomyśleć: ale w `levelUp` _mutujemy_ `user`, co się dzieje? Załóżmy, że kompilator Zig jest w błędzie i oszukajmy go. Zmusimy kompilator do zobaczenia, że `user` jest zmutowany:

```zig
const std = @import("std");

pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};
	user.power += 0;
  // reszta kodu jest taka sama
```

Teraz otrzymujemy błąd w `levelUp`: _cannot assign to constant_. Widzieliśmy w części 1, że parametry funkcji są stałymi, więc `user.power += 1;` jest nieprawidłowe. Aby naprawić błąd kompilacji, możemy zmienić funkcję `levelUp` na:

```zig
fn levelUp(user: User) void {
	var u = user;
	u.power += 1;
}
```

Co się skompiluje, ale na wyjściu otrzymamy, że _User 1 has power of 100_, mimo że intencją naszego kodu jest wyraźnie, aby `levelUp` zwiększył moc użytkownika do `101`. Co się dzieje?

Aby to zrozumieć, warto myśleć o danych w odniesieniu do pamięci, a zmienne jako etykiety, które kojarzą typ z określoną lokalizacją pamięci. Na przykład w `main` tworzymy `User`. Prosta wizualizacja tych danych w pamięci wyglądałaby następująco:

```
user -> ------------ (id)
        |    1     |
        ------------ (power)
        |   100    |
        ------------
```

Należy zwrócić uwagę na dwie ważne rzeczy. Po pierwsze, nasza zmienna `user` wskazuje na początek naszej struktury. Drugą jest to, że pola są ułożone sekwencyjnie. Pamiętaj, że nasz `user` ma również typ. Ten typ mówi nam, że `id` jest 64-bitową liczbą całkowitą, a `power` jest 32-bitową liczbą całkowitą. Uzbrojony w odniesienie do początku naszych danych i typu, kompilator może przetłumaczyć `user.power` na: _dostęp do 32-bitowej liczby całkowitej znajdującej się 64 bity od początku_. Na tym polega moc zmiennych, odwołują się one do pamięci i zawierają informacje o typie niezbędne do zrozumienia i manipulowania pamięcią w znaczący sposób.

> Domyślnie Zig nie gwarantuje układu pamięci struktur. Może przechowywać pola w kolejności alfabetycznej, według rosnącego rozmiaru lub z przerwami. Może robić co chce, o ile jest w stanie poprawnie przetłumaczyć nasz kod. Ta swoboda może umożliwić pewne optymalizacje. Tylko jeśli zadeklarujemy `packed struct`, otrzymamy silne gwarancje dotyczące układu pamięci. Możemy również utworzyć `extern struct`, która gwarantuje, że układ pamięci będzie zgodny z binarnym interfejsem aplikacji C (ABI). Mimo to, nasza wizualizacja `user` jest rozsądna i użyteczna.

Oto nieco inna wizualizacja, która zawiera adresy pamięci. Adres pamięci początku tych danych jest losowym adresem, który wymyśliłem. Jest to adres pamięci, do którego odwołuje się zmienna `user`, która jest również wartością naszego pierwszego pola, `id`. Jednak biorąc pod uwagę ten początkowy adres, wszystkie kolejne adresy mają znany adres względny. Ponieważ `id` jest 64-bitową liczbą całkowitą, zajmuje 8 bajtów pamięci. Dlatego `power` musi znajdować się pod adresem $start_address + 8:

```
user ->   ------------  (id: 1043368d0)
          |    1     |
          ------------  (power: 1043368d8)
          |   100    |
          ------------
```

Abyś mógł to sprawdzić, chciałbym przedstawić operator adresu: `&`. Jak sama nazwa wskazuje, operator adresu zwraca adres zmiennej (może również zwrócić adres funkcji, prawda?!). Zachowując istniejącą definicję `User`, wypróbuj ten `main`:

```zig
pub fn main() void {
	const user = User{
		.id = 1,
		.power = 100,
	};
	std.debug.print("{*}\n{*}\n{*}\n", .{&user, &user.id, &user.power});
}
```

Ten kod drukuje adres `user`, `user.id` i `user.power`. Możesz uzyskać różne wyniki w zależności od platformy i innych czynników, ale mam nadzieję, że zobaczysz, że adresy `user` i `user.id` są takie same, podczas gdy `user.power` jest przesunięty o 8 bajtów. Otrzymałem:

```
learning.User@1043368d0
u64@1043368d0
i32@1043368d8
```

Operator adresu zwraca wskaźnik do wartości. Wskaźnik do wartości jest odrębnym typem. Adres wartości typu `T` to `*T`. Mówimy, że jest to _wskaźnik do T_. Dlatego, jeśli weźmiemy adres `user`, otrzymamy `*User` lub wskaźnik do `User`:

```zig
pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};
	user.power += 0;

	const user_p = &user;
	std.debug.print("{any}\n", .{@TypeOf(user_p)});
}
```

Naszym pierwotnym celem było zwiększenie mocy użytkownika o 1 za pomocą funkcji `levelUp`. Udało nam się skompilować kod, ale kiedy wypisaliśmy `powe`, wciąż była to oryginalna wartość. To trochę przeskok, ale zmieńmy kod, aby wydrukować adres `user` w `main` i w `levelUp`:

```zig
pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};
	user.power += 0;

	// dodano to
	std.debug.print("main: {*}\n", .{&user});

	levelUp(user);
	std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

fn levelUp(user: User) void {
	// dodaj to
	std.debug.print("levelUp: {*}\n", .{&user});
	var u = user;
	u.power += 1;
}
```

Jeśli to uruchomisz, otrzymasz dwa różne adresy. Oznacza to, że `user` modyfikowany w `levelUp` różni się od `user` w `main`. Dzieje się tak, ponieważ Zig przekazuje kopię wartości. Może się to wydawać dziwnym domyślnym rozwiązaniem, ale jedną z korzyści jest to, że wywołujący funkcję może być pewien, że funkcja nie zmodyfikuje parametru (ponieważ nie może). W wielu przypadkach jest to dobra rzecz do zagwarantowania. Oczywiście czasami, tak jak w przypadku `levelUp`, chcemy, aby funkcja zmodyfikowała parametr. Aby to osiągnąć, `levelUp` musi działać na rzeczywistym `user` w `main`, a nie na jego kopii. Możemy to zrobić, przekazując do funkcji adres naszego użytkownika:

```zig
const std = @import("std");

pub fn main() void {
	var user = User{
		.id = 1,
		.power = 100,
	};

  // już niepotrzebne
	// user.power += 1;

	// user -> &user
	levelUp(&user);
	std.debug.print("User {d} has power of {d}\n", .{user.id, user.power});
}

// User -> *User
fn levelUp(user: *User) void {
	user.power += 1;
}

pub const User = struct {
	id: u64,
	power: i32,
};
```

Musieliśmy wprowadzić dwie zmiany. Pierwszą z nich jest wywołanie `levelUp` z adresem użytkownika, czyli `&user`, zamiast `user`. Oznacza to, że nasza funkcja nie otrzymuje już `User`. Zamiast tego otrzymuje `*User`, co było naszą drugą zmianą.

Nie potrzebujemy już tego brzydkiego hacka wymuszającego mutację użytkownika poprzez `user.power += 0;`. Początkowo nie udało nam się skompilować kodu, ponieważ `user` był `var`, ale kompilator powiedział nam, że nigdy nie został zmutowany. Pomyśleliśmy, że może kompilator się mylił i "oszukał" to, wymuszając mutację. Ale, jak teraz wiemy, użytkownik zmutowany w `levelUp` był inny; kompilator miał rację.

Kod działa teraz zgodnie z przeznaczeniem. Nadal istnieje wiele subtelności związanych z parametrami funkcji i ogólnie naszym modelem pamięci, ale robimy postępy. To może być dobry moment, aby wspomnieć, że poza specyficzną składnią, nic z tego nie jest unikalne dla Ziga. Model, który tutaj badamy, jest najbardziej powszechny, niektóre języki mogą po prostu ukrywać wiele szczegółów, a tym samym elastyczność, przed programistami.

## Metody

Najprawdopodobniej napisałbyś `levelUp` jako metodę struktury `User`:

```zig
pub const User = struct {
	id: u64,
	power: i32,

	fn levelUp(user: *User) void {
		user.power += 1;
	}
};
```

Nasuwa się pytanie: jak wywołać metodę z odbiornikiem wskaźnika? Może musimy zrobić coś w stylu: `&user.levelUp()`? Właściwie wystarczy wywołać ją normalnie, tj. `user.levelUp()`. Zig wie, że metoda oczekuje wskaźnika i przekazuje wartość poprawnie (przez referencję).

Początkowo wybrałem funkcję, ponieważ jest ona jawna, a tym samym łatwiejsza do nauczenia.

## Stałe parametry funkcji

Więcej niż sugerowałem, że domyślnie Zig będzie przekazywał kopię wartości (zwaną "przekazywaniem przez wartość"). Wkrótce zobaczymy, że rzeczywistość jest nieco bardziej subtelna (podpowiedź: co ze złożonymi wartościami z zagnieżdżonymi obiektami?).

Nawet trzymając się prostych typów, prawda jest taka, że Zig może przekazywać parametry w dowolny sposób, o ile może zagwarantować, że intencja kodu zostanie zachowana. W naszym oryginalnym `levelUp`, gdzie parametrem był `User`, Zig mógł przekazać kopię użytkownika lub odwołanie do `main.user`, o ile mógł zagwarantować, że funkcja go nie zmutuje. (Wiem, że ostatecznie chcieliśmy go zmutować, ale tworząc typ `User`, mówiliśmy kompilatorowi, że tego nie chcemy).

Ta swoboda pozwala Zigowi na użycie najbardziej optymalnej strategii opartej na typie parametru. Małe typy, takie jak `User`, mogą być tanio przekazywane przez wartość (tj. kopiowane). Większe typy mogą być tańsze do przekazania przez referencję. Zig może stosować dowolne podejście, o ile intencje kodu zostaną zachowane. Do pewnego stopnia jest to możliwe dzięki stałym parametrom funkcji.

Teraz znasz już jeden z powodów, dla których parametry funkcji są stałe.

> Być może zastanawiasz się, w jaki sposób przekazywanie przez referencję może być wolniejsze, nawet w porównaniu do kopiowania naprawdę małej struktury. Zobaczymy to dokładniej w następnej części, ale sedno tkwi w tym, że wykonywanie `user.power`, gdy `user` jest wskaźnikiem, dodaje niewielki narzut. Kompilator musi rozważyć koszt kopiowania w stosunku do kosztu dostępu do pól pośrednio przez wskaźnik.

## Wskaźnik do wskaźnika

Poprzednio przyjrzeliśmy się, jak wygląda pamięć `user` w naszej głównej funkcji. Teraz, gdy zmieniliśmy `levelUp`, jak wyglądałaby jego pamięć?

```
main:
user -> ------------  (id: 1043368d0)  <---
        |    1     |                      |
        ------------  (power: 1043368d8)  |
        |   100    |                      |
        ------------                      |
                                          |
        .............  puste miejsce      |
        .............  lub inne dane      |
                                          |
levelUp:                                  |
user -> -------------  (*User)            |
        | 1043368d0 |----------------------
        -------------
```

W `levelUp`, `user` jest wskaźnikiem do `User`. Jego wartością jest adres. Oczywiście nie byle jaki adres, ale adres `main.user`. Warto wyraźnie zaznaczyć, że zmienna `user` w `levelUp` reprezentuje konkretną wartość. Wartość ta jest adresem. I nie jest to tylko adres, ale także typ, `*User`. Wszystko to jest bardzo spójne, nie ma znaczenia, czy mówimy o wskaźnikach, czy nie: zmienne wiążą informacje o typie z adresem. Jedyną specjalną rzeczą dotyczącą wskaźników jest to, że gdy używamy składni kropkowej, np. `user.power`, Zig, wiedząc, że `user` jest wskaźnikiem, automatycznie podąży za adresem.

> Niektóre języki wymagają innego symbolu podczas uzyskiwania dostępu do pola za pomocą wskaźnika.

Ważne jest, aby zrozumieć, że zmienna `user` w `levelUp` sama istnieje w pamięci pod jakimś adresem. Tak jak zrobiliśmy to wcześniej, możemy to zobaczyć na własne oczy:

```zig
fn levelUp(user: *User) void {
	std.debug.print("{*}\n{*}\n", .{&user, user});
	user.power += 1;
}
```

Powyższe wypisuje adres, do którego odwołuje się zmienna `user`, a także jej wartość, która jest adresem `user` w `main`.

Jeśli `user` jest `*User`, to czym jest `&user`? To `**User`, czyli wskaźnik do wskaźnika na użytkownika. Mogę to robić, dopóki jednemu z nas nie skończy się pamięć!

_Istnieją_ przypadki użycia dla wielu poziomów pośrednictwa (indirection), ale nie jest to nic, czego teraz potrzebujemy. Celem tej sekcji jest pokazanie, że wskaźniki nie są niczym specjalnym, są po prostu wartością, która jest adresem i typem.

## Zagnieżdżone wskaźniki

Do tej pory nasz `User` był prosty, zawierał dwie liczby całkowite. Łatwo jest zwizualizować jego pamięć, a kiedy mówimy o "kopiowaniu", nie ma żadnych niejasności. Ale co się stanie, gdy `User` stanie się bardziej złożony i będzie zawierał wskaźnik?

```zig
pub const User = struct {
	id: u64,
	power: i32,
	name: []const u8,
};
```

Dodaliśmy `name`, która jest wycinkiem. Przypomnijmy, że wycinek to długość i wskaźnik. Gdybyśmy zainicjowali naszego `user` nazwą `"Goku"`, jak wyglądałby on w pamięci?

```
user -> -------------  (id: 1043368d0)
        |     1     |
        -------------  (power: 1043368d8)
        |    100    |
        -------------  (name.len: 1043368dc)
        |     4     |
        -------------  (name.ptr: 1043368e4)
  ------| 1182145c0 |
  |     -------------
  |
  |     .............  puste miejsce
  |     .............  lub inne dane
  |
  --->  -------------  (1182145c0)
        |    'G'    |
        -------------
        |    'o'    |
        -------------
        |    'k'    |
        -------------
        |    'u'    |
        -------------
```

Nowe pole `name` jest wycinkiem, który składa się z pola `len` i `ptr`. Są one ułożone w kolejności wraz ze wszystkimi innymi polami. Na platformie 64-bitowej zarówno `len`, jak i `ptr` będą miały 64 bity lub 8 bajtów. Interesującą częścią jest wartość `name.ptr`: jest to adres do innego miejsca w pamięci.

> Ponieważ użyliśmy literału łańcuchowego, `user.name.ptr` będzie wskazywać na konkretną lokalizację w obszarze, w którym przechowywane są wszystkie stałe w naszym pliku binarnym.

Typy mogą stać się znacznie bardziej złożone dzięki głębokiemu zagnieżdżaniu. Ale proste czy złożone, wszystkie zachowują się tak samo. W szczególności, jeśli wrócimy do naszego oryginalnego kodu, w którym `levelUp` pobierał zwykłego `User`, a Zig dostarczał kopię, jak wyglądałoby to teraz, gdy mamy zagnieżdżony wskaźnik?

Odpowiedź jest taka, że tworzona jest tylko płytka kopia wartości. Lub, jak niektórzy to określają, kopiowana jest tylko pamięć bezpośrednio adresowalna przez zmienną. Mogłoby się wydawać, że `levelUp` otrzyma połowiczną kopię `user`, być może z nieprawidłową nazwą. Należy jednak pamiętać, że wskaźnik, taki jak nasz `user.name.ptr`, jest wartością, a ta wartość jest adresem. Kopia adresu to wciąż ten sam adres:

```
main: user ->    -------------  (id: 1043368d0)
                 |     1     |
                 -------------  (power: 1043368d8)
                 |    100    |
                 -------------  (name.len: 1043368dc)
                 |     4     |
                 -------------  (name.ptr: 1043368e4)
                 | 1182145c0 |-------------------------
levelUp: user -> -------------  (id: 1043368ec)       |
                 |     1     |                        |
                 -------------  (power: 1043368f4)    |
                 |    100    |                        |
                 -------------  (name.len: 1043368f8) |
                 |     4     |                        |
                 -------------  (name.ptr: 104336900) |
                 | 1182145c0 |-------------------------
                 -------------                        |
                                                      |
                 .............  puste miejsce         |
                 .............  lub inne dane         |
                                                      |
                 -------------  (1182145c0)        <---
                 |    'G'    |
                 -------------
                 |    'o'    |
                 -------------
                 |    'k'    |
                 -------------
                 |    'u'    |
                 -------------
```

Z powyższego widać, że płytkie kopiowanie będzie działać. Ponieważ wartość wskaźnika jest adresem, kopiowanie wartości oznacza, że otrzymamy ten sam adres. Ma to ważne implikacje w odniesieniu do mutowalności. Nasza funkcja nie może zmodyfikować pól bezpośrednio dostępnych dla `main.user`, ponieważ otrzymała kopię, ale ma dostęp do tej samej `name`, więc czy może ją zmodyfikować? W tym konkretnym przypadku nie, `name` jest `const`. Dodatkowo, nasza wartość "Goku" jest literałem łańcuchowym, który jest zawsze niemutowalny. Ale przy odrobinie pracy możemy zobaczyć implikacje płytkiego kopiowania:

```zig
const std = @import("std");

pub fn main() void {
	var name = [4]u8{'G', 'o', 'k', 'u'};
	const user = User{
		.id = 1,
		.power = 100,
		// wytnij to, [4]u8 -> []u8
		.name = name[0..],
	};
	levelUp(user);
	std.debug.print("{s}\n", .{user.name});
}

fn levelUp(user: User) void {
	user.name[2] = '!';
}

pub const User = struct {
	id: u64,
	power: i32,
	// []const u8 -> []u8
	name: []u8
};
```

Powyższy kod wypisuje "Go!u". Musieliśmy zmienić typ `name` z `[]const u8` na `[]u8` i zamiast literału łańcuchowego, które są zawsze niemutowalne, utworzyć tablicę i pokroić ją. Niektórzy mogą dostrzec tu niekonsekwencję. Przekazywanie przez wartość uniemożliwia funkcji mutowanie bezpośrednich pól, ale nie pól z wartością za wskaźnikiem. Gdybyśmy chcieli, aby nazwa była niezmienna, powinniśmy zadeklarować ją jako `[]const` u8 zamiast `[]u8`.

Niektóre języki mają inną implementację, ale wiele języków działa dokładnie w ten sposób (lub bardzo bliski). Choć wszystko to może wydawać się ezoteryczne, ma to fundamentalne znaczenie dla codziennego programowania. Dobrą wiadomością jest to, że można to opanować za pomocą prostych przykładów i fragmentów; nie staje się to bardziej skomplikowane wraz ze wzrostem złożoności innych części systemu.

## Struktury rekursywne

Czasami potrzebna jest struktura rekurencyjna. Zachowując nasz istniejący kod, dodajmy opcjonalnego `manager` typu `?User` do naszego `User`. W tym momencie utworzymy dwóch użytkowników i przypiszemy jednego jako menedżera do drugiego:

```zig
const std = @import("std");

pub fn main() void {
	const leto = User{
		.id = 1,
		.power = 9001,
		.manager = null,
	};

	const duncan = User{
		.id = 1,
		.power = 9001,
		// zmieniono z leto -> &leto
		.manager = &leto,
	};

	std.debug.print("{any}\n{any}", .{leto, duncan});
}

pub const User = struct {
	id: u64,
	power: i32,
	// zmieniono z ?const User -> ?*const User
	manager: ?*const User,
};
```

Ten kod nie skompiluje się: _struct 'learning.User' depends on itself_. Nie powiedzie się, ponieważ każdy typ musi mieć znany rozmiar w czasie kompilacji.

Nie napotkaliśmy tego problemu, gdy dodaliśmy `name`, mimo że nazwy mogą mieć różne długości. Problemem nie jest rozmiar wartości, ale rozmiar samych typów. Zig potrzebuje tej wiedzy, aby zrobić wszystko, o czym mówiliśmy powyżej, jak uzyskanie dostępu do pola na podstawie jego pozycji offsetu. `name` był wycinkiem, `[]const u8`, który ma znany rozmiar: 16 bajtów - 8 bajtów dla `len` i 8 bajtów dla `ptr`.

Można by pomyśleć, że będzie to problem z każdą opcją lub unią. Jednak zarówno w przypadku opcjonali, jak i unii, największy możliwy rozmiar jest znany i Zig może go użyć. Struktura rekurencyjna nie ma takiego górnego ograniczenia, struktura może wykonać rekurencję raz, dwa lub miliony razy. Liczba ta będzie się różnić w zależności od `User` i nie będzie znana w czasie kompilacji.

Widzieliśmy odpowiedź z `name`: użyj wskaźnika. Wskaźniki zawsze zajmują bajty `usize`. Na platformie 64-bitowej jest to 8 bajtów. Podobnie jak rzeczywista nazwa "Goku" nie była przechowywana z/wraz z naszym `user`, użycie wskaźnika oznacza, że nasz menedżer nie jest już powiązany z układem pamięci `user`.

```zig
const std = @import("std");

pub fn main() void {
	const leto = User{
		.id = 1,
		.power = 9001,
		.manager = null,
	};

	const duncan = User{
		.id = 1,
		.power = 9001,
    // zmieniono z leto -> &leto
		.manager = &leto,
	};

	std.debug.print("{any}\n{any}", .{leto, duncan});
}

pub const User = struct {
	id: u64,
	power: i32,
  // zmieniono z ?const User -> ?*const User
	manager: ?*const User,
};
```

Być może nigdy nie będziesz potrzebował struktury rekurencyjnej, ale tu nie chodzi o modelowanie danych. Chodzi o zrozumienie wskaźników i modeli pamięci oraz lepsze zrozumienie tego, co robi kompilator.

---

Wielu programistów zmaga się ze wskaźnikami, może być w nich coś nieuchwytnego. Nie są one tak konkretne jak liczba całkowita, łańcuch czy `User`. Nic z tego nie musi być krystalicznie czyste, abyś mógł iść naprzód. Ale warto je opanować, i to nie tylko dla Ziga. Te szczegóły mogą być ukryte w językach takich jak Ruby, Python i JavaScript, a w mniejszym stopniu C#, Java i Go, ale nadal tam są, wpływając na sposób pisania kodu i jego działania. Nie spiesz się więc, baw się przykładami, dodawaj debugujące instrukcje wypisywania, aby przyjrzeć się zmiennym i ich adresom. Im więcej odkryjesz, tym jaśniejsze stanie się to wszystko.
