# Troubleshooting: zawieszanie się live Lubuntu 26.04 na tablecie Lamina (Cherry Trail)

Nov 23, 2026 · @Klaudia

## Streszczenie

Zwykły obraz Lubuntu 26.04.1 zapisany na pendrive 1:1 zawiesza się na tablecie Lamina albo kończy się na kernel panic w ciągu kilku minut od startu systemu live.&#32;

Najbardziej prawdopodobna przyczyna to wyścig czasowy między usypianiem łącza PCIe (ASPM L1) do karty Wi-Fi Intel 3165 a głębokimi stanami uśpienia procesora Cherry Trail.

Obejście dla zwykłego obrazu: ustawienie parametru jądra `pcie_aspm=off` (3 starty na 3 udane) albo `intel_idle.max_cstate=1` (również 3/3). Pendrive zrobiony aplikacją bootia32-usb (partycja FAT32 ze skopiowanymi plikami ISO) startuje z tego samego obrazu bez żadnych parametrów, 3 razy na 3. Zainstalowany system z pendrive utworzonego z tej aplikacji działa stabilnie, także po uśpieniu na 2 godziny.

## Środowisko i objawy

Problem dotyczy jednego tabletu, ale jego sprzęt jest typowy dla całej rodziny Intel Cherry Trail.

| Element | Wartość |
| --- | --- |
| Tablet | Lamina LAMINALTPRO1024 / SMB-H1005F |
| Firmware | Insyde EFI v2.4, 64-bitowy UEFI (`fw_platform_size` = 64), BIOS LTPRO1024.I22K6.9.01 z 2019 r. |
| Procesor / GPU | Intel Cherry Trail (Cherry View, id GPU 22b0) |
| Wi-Fi / Bluetooth | Intel Wireless-AC 3165 (rev 91), PCIe `0000:01:00.0`, sterownik iwlwifi, plus Intel BT |
| Pamięć | eMMC 58 GB (mmcblk0) |
| Obraz | Lubuntu 26.04.1; jądro w live 7.0.0-30-generic, w zainstalowanym systemie 7.0.0-43-generic |

Objawy zwykłego obrazu zapisanego na pendrive:

- Ekran startowy (splash) wisi bardzo długo, nawet około 10 minut. Rzadko potrafi się również pojawić kernel panic `Timeout: Not all CPUs entered broadcast exception handler`. To błąd sprzętowy typu machine check.
- Wybranie w GRUB-ie opcji safe graphics (`nomodeset`) pozwala uruchomić system live.
- Zmodyfikowanie parametru jądra na `pcie_aspm=off` albo `intel_idle.max_cstate=1` przed `---` również pozwalają uruchomić system.

## Przebieg diagnozy

Dwanaście kroków, od pierwszych prób na Fedorze do porównania parametrów ASPM. Każdy krok zmienia jedną rzecz, a wynik pokazuje, co można odrzucić lub zawlęzić.

| Nr | Test | Wynik | Co z tego wynika |
| --- | --- | --- | --- |
| 1 | Fedora LXQt Live 45, zwykły obraz | Startuje do pulpitu | Fedora ma w obrazie własne pliki ia32 i nie potrzebuje bloba bootia32.efi |
| 2 | &#32;Wynik komend:`fw_platform_size`, `dmesg` (EFI) | 64, Insyde EFI v2.4 | Firmware jest 64-bitowy, więc blob 32-bitowy nie jest tu potrzebny |
| 3 | Zwykły Lubuntu 26.04.1, bez parametrów | Długi splash, kernel panic lub zawieszenie, 3 próby na 3 | Problem jest powtarzalny |
| 4 | To samo plus `intel_idle.max_cstate=1` | Startuje, parametr potwierdzony w `/proc/cmdline` | Trop na głębokie stany C procesora (znany błąd Cherry Trail, kernel bugzilla 109051) |
| 5 | Safe graphics (`nomodeset`) | Startuje, stabilny ponad 20 min | Zmiana kolejności i tempa startu też omija problem |
| 6 | Pendrive z aplikacji bootia32-usb, ta sama ISO | Startuje 3 razy na 3 bez parametrów | Układ plików i partycji na pendrive wpływa na wynik |
| 7 | Zainstalowany system, uśpienie s2idle na 2 godziny | Wraca, Wi-Fi i dotyk działają | System po instalacji jest stabilny |
| 8 | Logi `casper.log` i `boot.log` z pendrive'a po zawieszeniach | Ostatnie wpisy: start `systemd-rfkill` (sesja .0), start iwlwifi i Bluetooth (sesja .3) | Pierwszy trop na radio; partycja `writable` powstaje tylko w jednej sesji, więc nie jest przyczyną |
| 9 | Zwykły obraz plus `modprobe.blacklist=iwlwifi,btusb` | Zawiesił się (1 próba) | Sam sterownik Wi-Fi i Bluetooth nie jest sprawcą |
| 10 | Zwykły obraz plus `pcie_aspm=off` | Startuje 3 razy na 3 | Wyłączenie oszczędzania energii na magistrali PCIe usuwa problem |
| 11 | Zwykły obraz plus `pcie_aspm.policy=performance` | Startuje (3 próby) | Zgodne z krokiem 10 |
| 12 | Zainstalowany system: `lspci -vv -s 01:00.0`, polityka ASPM | `01:00.0` to Wi-Fi 3165; ASPM L1 włączony, podstany L1.1 i L1.2 wyłączone, polityka `default` | ASPM jest tam włączony i system działa, więc sam L1 nie wystarcza do awarii; liczy się moment włączenia |

### Sesje zapisane na pendrive (kroki 3 i 8)

Po każdym starcie zwykłego obrazu system live zapisywał logi na partycji `writable` pendrive'a, w folderze `install-logs-2026-11-16.N`. W tekście „sesja .0”, „.3” itd. oznacza właśnie końcówkę `N` tej nazwy. Pendrive z aplikacji nie miał takich logów.

| Folder | Czas wg zegara tabletu (start → ostatnia zmiana w `log`) | Parametry jądra (wg przebiegu prób, do potwierdzenia w logu) | Partycja `writable` utworzona w tej sesji | Ostatnie ślady w logach | Wynik |
| --- | --- | --- | --- | --- | --- |
| `.0` | 01:23 → 01:27 (ok. 4 min) | Bez parametrów (pierwsza próba, zwykły obraz) | Tak (6 wpisów z `GPT` w `casper.log`: `GPT PMBR size mismatch`, `backup GPT table is not on the end of the device`) | Ostatnie linie `boot.log`: `Listening on systemd-rfkill.socket`, `Starting systemd-rfkill.service` | Zawieszenie / kernel panic |
| `.1` | 12:29 → 13:15 (ok. 46 min) | `intel_idle.max_cstate=1` | Nie (0 wpisów `GPT`) | Pełne logi (`syslog`, `kern.log`, `Xorg.0.log`); `boot.log` kończy się na `Finished systemd-user-sessions`, `Starting plymouth-quit`, `Starting setvtrgb` | Pełny start do pulpitu |
| `.2` | 15:20 → 15:22 (ok. 2 min) | Bez parametrów | Nie (0 wpisów `GPT`) | `boot.log` pusty (0 bajtów) | Zawieszenie |
| `.3` | 15:57 → 15:58 (ok. 1 min) | Bez parametrów | Nie (0 wpisów `GPT`) | `boot.log` pusty; ostatnie komunikaty jądra w dzienniku: start iwlwifi i Bluetooth | Zawieszenie |
| `.4` | 16:31 → 16:45 (ok. 14 min) | Safe graphics (`nomodeset`) | Nie (0 wpisów `GPT`) | Pełne logi; `boot.log` kończy się tak samo jak w `.1` (`Starting setvtrgb`) | Pełny start do pulpitu |

Czasy to orientacyjny dolny szacunek: znacznik zmiany folderu `log` zmienia się tylko wtedy, gdy powstaje lub znika w nim wpis, a nie przy każdym zapisie, więc sesja mogła trwać dłużej.

Wnioski z tabeli: wszystkie trzy starty bez parametrów (`.0`, `.2`, `.3`) skończyły się zawieszeniem, a oba pełne starty (`.1`, `.4`) miały parametr lub safe graphics. Partycja `writable` powstała tylko w sesji `.0`, więc jej tworzenie nie jest przyczyną: sesje `.2` i `.3` zawiesiły się bez niej. Zawieszenia `.0` i `.3` kończą się na starcie radia, co było pierwszym tropem na kartę Wi-Fi. Próby z `modprobe.blacklist` i `pcie_aspm` (kroki 9–11) były na innym pendrive'ie, więc nie ma ich w tej tabeli.

## Hipotezy odrzucone lub osłabione

Część pierwszych podejrzeń okazała się błędna; każde odrzucenie zawęziło poszukiwania.

- **Blob bootia32.efi rozwiązuje problem.** Odrzucone: firmware jest 64-bitowy, a pendrive z samym zwykłym obrazem na Fedorze startuje bez bloba.
- **Winne jest obciążenie USB lub konkretny pendrive.** Odrzucone: ten sam pendrive działał w obu trybach, zwykłym i z aplikacji.
- **Winna jest wersja jądra (-30 w live, -43 po instalacji).** Osłabione: ta sama ISO z jądrem -30 startuje z pendrive'a z aplikacji.
- **Winne jest tworzenie partycji `writable` przy pierwszym starcie.** Odrzucone: sesje .2 i .3 nie miały tego kroku, a zawiesiły się.
- **Winny jest sterownik Wi-Fi lub Bluetooth.** Osłabione: blacklista `iwlwifi` i `btusb` nie pomogła. Trop radia przetrwał w innej formie: chodzi o stan zasilania łącza PCIe do karty, który ustawia jądro przed załadowaniem sterownika, nie o sam sterownik.

## Wniosek i obejścia

Najlepiej pasujące wyjaśnienie: karta Wi-Fi 3165 schodzi w stan oszczędzania energii ASPM L1, a w tym samym ułamku sekundy procesor Cherry Trail jest w głębokim stanie C. Gdy oba się budzą w niefortunnej kolejności, procesor zgłasza machine check i system staje. Na starcie live jest dużo równoległych zadań (odczyt z USB, ładowanie modułów, start Wi-Fi i Bluetooth), więc wyścig trafia się tam najczęściej.

To spójne z wszystkimi testami: `pcie_aspm=off` blokuje uśpianie łącza, `intel_idle.max_cstate=1` blokuje głęboki sen procesora, a safe graphics zmienia czasy startu.

| Sytuacja | Co zrobić |
| --- | --- |
| Pendrive do tego tabletu | Użyć aplikacji bootia32-usb: partycja GPT z FAT32 (`parted mkpart ESP fat32 1MiB 100%`), skopiowane pliki ISO; start bez parametrów |
| Zwykły obraz zapisany dd | Dopisać `pcie_aspm=off` w linii `linux`, przed `---` |
| Alternatywa dla zwykłego obrazu | `intel_idle.max_cstate=1` albo safe graphics |
| System zainstalowany na eMMC | Bez zmian, działa stabilnie |

Uwaga o aplikacji: pomaga ona dzięki układowi pliku i partycji, nie dzięki blobowi bootia32.efi, bo ten tablet ma 64-bitowy firmware. Dlaczego ten układ omija wyścig, jest hipotezą (inny rytm odczytów z USB zmienia czasy startu), nie zmierzonym faktem.

## Do rozważenia

Wynik jest solidny w praktyce, ale kilka rzeczy pozostaje niepotwierdzonych.

- **Dlaczego układ FAT32 omija problem.** Hipoteza o innym rytmie odczytów z USB nie została zmierzona.
- **Dokładna chwila awarii.** Po twardym zawieszeniu logi urywają się wcześniej niż sama awaria, bo nie zdążyły się zapisać. Zapis z `pstore` albo konsola szeregowa mogłyby to pokazać, ale nie były próbowane.
- **Rozmiar próby.** Zawieszenia są losowe, a liczba prób mała (po 3 na wariant), więc pojedyncze wyniki mogą być szczęściem.

## Wnioski na przyszłość

Najwięcej dała zmiana jednej rzeczy naraz i porównywanie z działającym przypadkiem.

- Zacząć od faktu o sprzęcie (`fw_platform_size`), zanim założy się, że problem leży w architekturze firmware'u.
- Powtarzać test; przy błędzie losowym jedna udana próba nic nie dowodzi.
- Szukać różnicy między przypadkiem działającym a niedziałającym (ten sam obraz, dwa układy na pendrive) zamiast zgadywać w próżni.
- Zbierać logi z pendrive'a po każdej próbie i pamiętać, że po twardym zawieszeniu są niepełne.
- Publicznie odrzucać własne hipotezy, gdy dane im przeczyły (blob, wersja jądra, obciążenie USB).
- Rozróżniać w notatce to, co zmierzone, od tego, co jest wnioskiem z poszlak.

## Załącznik: komendy i parametry

Komendy użyte w diagnozie, z krótkim opisem tego, co sprawdzają.

| Komenda lub parametr | Do czego służył |
| --- | --- |
| `cat /sys/firmware/efi/fw_platform_size` | Czy firmware UEFI jest 32- czy 64-bitowy |
| `cat /proc/cmdline` | Jakie parametry jądra naprawdę działają |
| `journalctl -D <katalog> -k` | Odczyt logów jądra zapisanych na pendrive po zawieszeniu |
| `lspci -s 01:00.0` | Nazwa urządzenia na magistrali PCIe (tu Wi-Fi 3165) |
| `sudo lspci -vv -s 01:00.0 \| grep -iE 'ASPM\|LnkCtl'` | Stan ASPM i podstanów L1 dla karty |
| `cat /sys/module/pcie_aspm/parameters/policy` | Aktualna polityka ASPM w jądrze |
| `sudo dmesg \| grep -i aspm` | Co jądro zgłosiło o ASPM przy starcie |
| `pcie_aspm=off` | Wyłącza zarządzanie ASPM po stronie jądra |
| `pcie_aspm.policy=performance` | Zabrania używania L0s i L1 |
| `intel_idle.max_cstate=1` | Ogranicza głębokie stany uśpienia procesora |
| `modprobe.blacklist=iwlwifi,btusb` | Blokuje sterowniki Wi-Fi i Bluetooth |
| `nomodeset` | Odpowiednik safe graphics |

Jak dodać parametr w GRUB-ie z pendrive'a: w menu wcisnąć `e`, dopisać parametr w linii zaczynającej się od `linux`, **przed** znakami `---`, potem uruchomić klawiszem F10 lub Ctrl+X. Zmiana działa tylko na jedno uruchomienie.
