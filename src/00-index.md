# Nauka Ziga

Witamy w Nauce Ziga, wprowadzeniu do języka programowania Zig. Ten przewodnik ma na celu ułatwienie korzystania z Ziga. Zakłada on wcześniejsze doświadczenie w programowaniu, choć nie w żadnym konkretnym języku.

Zig jest intensywnie rozwijany i zarówno język Zig, jak i jego standardowa biblioteka stale ewoluują. Niniejszy przewodnik dotyczy najnowszej wersji rozwojowej Ziga. Może się jednak zdarzyć, że część kodu nie będzie zsynchronizowana. Jeśli pobrałeś najnowszą wersję Ziga i masz problemy z uruchomieniem kodu, [zgłoś ten problem](https://github.com/karlseguin/blog/issues) (w języku angielskim).

## Tłumaczenia

- [Chińskie](https://zigcc.github.io/learning-zig/index.html) - przez Jiacai Liu
- [Rosyjskie](https://github.com/dee0xeed/learning-zig-rus/blob/main/src/ch01.md) - przez dee0xeed
- [Koreańskie](https://faultnote.github.io/posts/learning-zig/) - przez faultnote
- [Brazylijskie](https://jkitajima.github.io/learning-zig-karlseguin/) - przez jkitajima

## Spis treści

1. [Instalacja Ziga](#instalacja ziga)
2. [Przegląd języka - część 1](./01-language_overview_part_1.md)
3. [Przegląd języka - część 2](./02-language_overview_part_2.md)
4. [Przewodnik po stylach](./03-style_guide.md)
5. [Wskaźniki](./04-pointers.md)
6. [Pamięć stosu](./05-stack_memory.md)
7. [Pamięć sterty i alokatory](./06-heap_memory_and_allocators.md)
8. [Generyczność (polimorfizm parametryczny)](./07-generics.md)
9. [Kodowanie w Zigu](./08-coding_in_zig.md)
10. [Wnioski](./09-conclusion.md)

## Instalacja Ziga

[Strona pobierania](https://ziglang.org/download/) Zig zawiera prekompilowane pliki binarne dla popularnych platform. Na tej stronie znajdziesz pliki binarne dla najnowszej wersji rozwojowej, a także dla głównych wydań. Najnowsza wersja, której dotyczy niniejszy przewodnik, znajduje się na górze strony.

Dla mojego komputera będę pobierał zig-macos-aarch64-0.12.0-dev.2777+2176a73d6.tar.xz. Być może korzystasz z innej platformy lub nowszej wersji. Po rozwinięciu archiwum powinieneś mieć plik binarny `zig` (oprócz innych rzeczy), który będziesz chciał aliasować lub dodać do swojej ścieżki; w zależności od tego, do czego jesteś przyzwyczajony.

Teraz powinieneś być w stanie uruchomić `zig zen` i `zig version`, aby przetestować swoją konfigurację.
