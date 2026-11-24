# troubleshooting-notes

*English below · [Po polsku poniżej](#po-polsku)*

Write-ups of IT problems I diagnosed myself: Linux, hardware, networking. Each note keeps the whole path, not just the fix: symptoms, tests, dead ends, and what was measured versus what is only a guess.

I'm an IT/ICT technician working towards Linux/Windows administration and networking. I keep these notes to learn from them, and so that someone with the same problem can find them.

## Notes

| Date | Problem | Area | Status | Read |
| --- | --- | --- | --- | --- |
| 2026-11 | Live Lubuntu 26.04 freezes on a Cherry Trail tablet (Lamina) | Linux, hardware | Workaround found; probable cause (ASPM / C-state race), not proven | [EN](notes/en/2026-11-lamina-lubuntu-live-hang.md) · [PL](notes/pl/2026-11-lamina-lubuntu-live-hang.md) |

**Status values:** *Solved* (cause found and fixed) · *Workaround found* (it works, the cause is not fully proven) · *Unsolved* (what I ruled out is still worth keeping).

## Pattern

- One note per problem, also when it ends unresolved.
- One change per test; repeat a test when the failure is random.
- Measured facts are kept apart from inferences.
- Rejected hypotheses stay in the note, with the reason.


## Repository layout

```
troubleshooting-notes/
├── README.md
├── LICENSE
├── TEMPLATE.md
├── TEMPLATE.pl.md
└── notes/
    ├── en/    notes in English
    └── pl/    the same notes in Polish
```


## Related projects

- [bootia32-usb](https://github.com/mro73/bootia32-usb): a small Linux app that makes bootable USB sticks for the tablet in the first note. It automates the procedure from [lamadotcare/bootia32-efi](https://github.com/lamadotcare/bootia32-efi).

## License

[CC0 1.0](LICENSE): do whatever you like with these notes.

---

## Po polsku

Notatki o problemach IT, które sama zdiagnozowałam: Linux, sprzęt, sieci. Każda notatka zachowuje całą drogę, a nie tylko rozwiązanie: objawy, testy, ślepe uliczki oraz to, co zostało zmierzone, a co jest tylko przypuszczeniem.

Jestem technikiem teleinformatykiem i zmierzam w stronę administracji Linux/Windows oraz sieci. Prowadzę te notatki, żeby się z nich uczyć i żeby ktoś z takim samym problemem mógł je znaleźć.

**Spis notatek** jest w tabeli powyżej (kolumna *Read*, link PL).

**Zasady pisania notatek:**

- Jedna notatka na jeden problem, także gdy kończy się bez rozwiązania.
- Jedna zmiana na test; test do powtórzenia gdy awaria jest losowa.
- Zmierzone fakty oddzielane od wniosków z poszlak.
- Odrzucone hipotezy zostają w notatce razem z powodem.


**Licencja:** [CC0 1.0](LICENSE).
