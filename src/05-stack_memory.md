# Pamięć stosu

Zagłębienie się we wskaźniki zapewniło wgląd w relacje między zmiennymi, danymi i pamięcią. Mamy więc pojęcie o tym, jak wygląda pamięć, ale musimy jeszcze porozmawiać o tym, jak dane i, co za tym idzie, pamięć są zarządzane. W przypadku krótkich i prostych skryptów prawdopodobnie nie ma to znaczenia. W erze laptopów o pojemności 32 GB można uruchomić program, użyć kilkuset megabajtów pamięci RAM do odczytania pliku i przeanalizowania odpowiedzi HTTP, zrobić coś niesamowitego i wyjść. Po wyjściu z programu system operacyjny wie, że pamięć, którą dał programowi, może teraz zostać wykorzystana do czegoś innego.

Ale w przypadku programów, które działają przez dni, miesiące, a nawet lata, pamięć staje się ograniczonym i cennym zasobem, prawdopodobnie poszukiwanym przez inne procesy działające na tym samym komputerze. Po prostu nie ma sposobu, aby poczekać, aż program zakończy działanie, aby zwolnić pamięć. Jest to główne zadanie garbage collectora: wiedzieć, które dane nie są już używane i zwalniać pamięć. W Zig to ty jesteś garbage collectorem.

Większość pisanych programów będzie korzystać z trzech "obszarów" pamięci. Pierwszym z nich jest przestrzeń globalna, w której przechowywane są stałe programu, w tym literały łańcuchowe. Wszystkie dane globalne są wbudowane w plik binarny, w pełni znane w czasie kompilacji (a więc i w czasie wykonywania) i niezmienne. Dane te istnieją przez cały czas życia programu, nigdy nie potrzebując więcej lub mniej pamięci. Pomijając wpływ, jaki ma to na rozmiar naszego pliku binarnego, nie jest to coś, o co w ogóle musimy się martwić.

Drugim obszarem pamięci jest stos wywołań, który jest tematem tej części. Trzecim obszarem jest sterta, temat na następną część.

> Nie ma fizycznej różnicy między tymi obszarami pamięci, jest to koncepcja stworzona przez system operacyjny i program wykonywalny.

## Ramki stosu

Wszystkie dane, które widzieliśmy do tej pory, były stałymi przechowywanymi w sekcji danych globalnych naszych zmiennych binarnych lub lokalnych. "Lokalny" oznacza, że zmienna jest ważna tylko w zakresie, w którym została zadeklarowana. W Zig, zakresy zaczynają się i kończą nawiasami klamrowymi, `{ ... }`. Większość zmiennych ma zakres funkcji, w tym parametry funkcji lub blok przepływu sterowania, taki jak `if`. Jak jednak widzieliśmy, można tworzyć dowolne bloki, a tym samym dowolne zakresy.

W poprzedniej części zwizualizowaliśmy pamięć naszych funkcji `main` i `levelUp`, każda z `User`:

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

Jest powód, dla którego `levelUp` znajduje się zaraz po `main`: jest to nasz [uproszczony] stos wywołań. Kiedy nasz program się uruchamia, `main` wraz ze swoimi zmiennymi lokalnymi jest wciskany na stos wywołań. Gdy wywoływany jest `levelUp`, jego parametry i wszelkie zmienne lokalne są wypychane na stos wywołań. Co ważne, gdy `levelUp` powraca, jest zdejmowany ze stosu. Po powrocie `levelUp` i powrocie kontroli do `main`, nasz stos wywołań wygląda następująco:

```
main: user ->    -------------  (id: 1043368d0)
                 |     1     |
                 -------------  (power: 1043368d8)
                 |    100    |
                 -------------  (name.len: 1043368dc)
                 |     4     |
                 -------------  (name.ptr: 1043368e4)
                 | 1182145c0 |-------------------------
                 -------------
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

Gdy funkcja jest wywoływana, cała jej ramka stosu jest umieszczana na stosie wywołań. Jest to jeden z powodów, dla których musimy znać rozmiar każdego typu. Chociaż możemy nie znać długości nazwy naszego użytkownika do momentu wykonania tej konkretnej linii kodu (zakładając, że nie był to stały literał łańcuchowy), wiemy, że nasza funkcja ma `User` i oprócz innych pól będziemy potrzebować 8 bajtów na `name.len` i 8 bajtów `name.ptr`.

Gdy funkcja powraca, jej ramka stosu, która jako ostatnia została wepchnięta na stos wywołań, jest usuwana. Właśnie stało się coś niesamowitego: pamięć używana przez `levelUp` została automatycznie zwolniona! Chociaż technicznie pamięć ta mogłaby zostać zwrócona do systemu operacyjnego, o ile mi wiadomo, żadna implementacja nie zmniejsza stosu wywołań (choć w razie potrzeby dynamicznie go powiększa). Mimo to pamięć używana do przechowywania ramki stosu `levelUp` jest teraz wolna do wykorzystania w naszym procesie dla innej ramki stosu.

> W normalnym programie stos wywołań może być dość duży. Pomiędzy całym kodem frameworka i bibliotekami, z których korzysta typowy program, pojawiają się głęboko zagnieżdżone funkcje. Zwykle nie stanowi to problemu, ale od czasu do czasu można napotkać pewien rodzaj błędu przepełnienia stosu. Dzieje się tak, gdy na naszym stosie wywołań zabraknie miejsca. Najczęściej dzieje się tak w przypadku funkcji rekurencyjnych - funkcji, która wywołuje samą siebie.

Podobnie jak nasze dane globalne, stos wywołań jest zarządzany przez system operacyjny i program wykonywalny. Po uruchomieniu programu i dla każdego wątku, który uruchamiamy później, tworzony jest stos wywołań (którego rozmiar można zwykle skonfigurować w systemie operacyjnym). Stos wywołań istnieje przez cały czas trwania programu lub, w przypadku wątku, przez cały czas trwania wątku. Po zakończeniu programu lub wątku stos wywołań jest zwalniany. Ale podczas gdy nasze dane globalne zawierają wszystkie globalne dane programu, stos wywołań zawiera tylko ramki stosu dla aktualnie wykonywanej hierarchii funkcji. Jest to wydajne zarówno pod względem wykorzystania pamięci, jak i prostoty wkładania i zdejmowania ramek stosu na i ze stosu.

## Zwisające wskaźniki

Stos wywołań jest niesamowity zarówno ze względu na swoją prostotę, jak i wydajność. Ale jest też przerażający: kiedy funkcja powraca, wszystkie jej lokalne dane stają się niedostępne. Może to brzmieć rozsądnie, w końcu są to dane lokalne, ale może to wprowadzić poważne problemy. Rozważmy ten kod:

```zig
onst std = @import("std");

pub fn main() void {
	const user1 = User.init(1, 10);
	const user2 = User.init(2, 20);

	std.debug.print("User {d} has power of {d}\n", .{user1.id, user1.power});
	std.debug.print("User {d} has power of {d}\n", .{user2.id, user2.power});
}

pub const User = struct {
	id: u64,
	power: i32,

	fn init(id: u64, power: i32) *User{
		var user = User{
			.id = id,
			.power = power,
		};
		return &user;
	}
};
```

Na pierwszy rzut oka rozsądne byłoby oczekiwanie następujących danych wyjściowych:

```
User 1 has power of 10
User 2 has power of 20
```

Otrzymałem:

```
User 2 has power of 20
User 9114745905793990681 has power of 0
```

Możesz uzyskać inne wyniki, ale na podstawie moich danych wyjściowych `user1` odziedziczył wartości `user2`, a wartości `user2` są bezsensowne. Kluczowym problemem w tym kodzie jest to, że `User.init` zwraca adres lokalnego użytkownika, `&user`. Nazywa się to zwisającym wskaźnikiem, wskaźnikiem, który odwołuje się do nieprawidłowej pamięci. Jest to źródło wielu naruszeń ochrony pamięci (segfaults).

Gdy ramka stosu jest usuwana ze stosu wywołań, wszelkie odniesienia do tej pamięci są nieważne. Wynik próby uzyskania dostępu do tej pamięci jest niezdefiniowany. Prawdopodobnie otrzymasz bezsensowne dane lub segfault. Moglibyśmy spróbować wyciągnąć jakieś wnioski z moich danych wyjściowych, ale nie jest to zachowanie, na którym chcielibyśmy lub nawet moglibyśmy polegać.

Jednym z wyzwań związanych z tego typu błędami jest to, że w językach z garbage collectorami powyższy kod jest całkowicie w porządku. Na przykład Go wykryłby, że lokalny `user` przeżyje swój zakres, funkcję `init` i zapewniłby jej ważność tak długo, jak jest to potrzebne (sposób, w jaki Go to robi, jest szczegółem implementacji, ale ma kilka opcji, w tym przeniesienie danych na stertę, o czym jest następna część).

Inną kwestią, którą muszę z przykrością stwierdzić, jest to, że może to być trudny do wykrycia błąd. W naszym powyższym przykładzie wyraźnie zwracamy adres lokalny. Ale takie zachowanie może ukrywać się wewnątrz funkcji zagnieżdżonych i złożonych typów danych. Czy widzisz jakieś możliwe problemy z poniższym niekompletnym kodem?

```zig
fn read() !void {
	const input = try readUserInput();
	return Parser.parse(input);
}
```

Cokolwiek `Parser.parse` zwróci, przeżyje `input`. Jeśli `Parser` przechowuje odniesienie do `input`, będzie to zwisający wskaźnik, który tylko czeka na awarię naszej aplikacji. Idealnie, jeśli `Parser` potrzebuje `input` tak długo, jak to robi, utworzy ich kopię, a ta kopia będzie powiązana z jej własnym czasem życia (więcej na ten temat w następnej części). Nie ma tu jednak nic, co pozwoliłoby wyegzekwować ten kontrakt. Dokumentacja `Parser` może rzucić nieco światła na to, czego oczekuje od `input` lub co z nim robi. W przeciwnym razie będziemy musieli zagłębić się w kod, aby to rozgryźć.

---

Prostym sposobem na rozwiązanie naszego początkowego błędu jest zmiana `init` tak, aby zwracał `User`, a nie `*User` (wskaźnik do `User`). Moglibyśmy wtedy zrobić `return user;` zamiast `return &user;`. Ale nie zawsze będzie to możliwe. Dane często muszą żyć poza sztywnymi granicami zakresów funkcji. W tym celu mamy trzeci obszar pamięci, stertę (heap), temat następnej części.

Zanim zagłębimy się w stertę, warto wiedzieć, że przed końcem tego przewodnika zobaczymy ostatni przykład zwisających wskaźników. W tym momencie omówimy już wystarczająco dużo języka, aby podać znacznie mniej zawiły przykład. Chcę powrócić do tego tematu, ponieważ dla programistów wywodzących się z języków garbage collected może to powodować błędy i frustrację. Jest to coś, z czym sobie **poradzisz**. Sprowadza się to do bycia świadomym tego, gdzie i kiedy istnieją dane.
