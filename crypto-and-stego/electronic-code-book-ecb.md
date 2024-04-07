<details>

<summary><strong>Naučite hakovanje AWS-a od nule do heroja sa</strong> <a href="https://training.hacktricks.xyz/courses/arte"><strong>htARTE (HackTricks AWS Red Team Expert)</strong></a><strong>!</strong></summary>

Drugi načini podrške HackTricks-u:

* Ako želite da vidite **vašu kompaniju reklamiranu na HackTricks-u** ili da **preuzmete HackTricks u PDF formatu** proverite [**PLANOVE ZA PRIJEM**](https://github.com/sponsors/carlospolop)!
* Nabavite [**zvanični PEASS & HackTricks swag**](https://peass.creator-spring.com)
* Otkrijte [**Porodicu PEASS**](https://opensea.io/collection/the-peass-family), našu kolekciju ekskluzivnih [**NFT-ova**](https://opensea.io/collection/the-peass-family)
* **Pridružite se** 💬 [**Discord grupi**](https://discord.gg/hRep4RUj7f) ili [**telegram grupi**](https://t.me/peass) ili nas **pratite** na **Twitteru** 🐦 [**@hacktricks_live**](https://twitter.com/hacktricks_live)**.**
* **Podelite svoje hakovanje trikove slanjem PR-ova na** [**HackTricks**](https://github.com/carlospolop/hacktricks) i [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) github repozitorijume.

</details>


# ECB

(ECB) Electronic Code Book - simetrična šema šifrovanja koja **zamenjuje svaki blok čistog teksta** sa **blokom šifrovanog teksta**. To je **najjednostavnija** šema šifrovanja. Osnovna ideja je da se **podeli** čisti tekst u **blokove od N bitova** (zavisi od veličine bloka ulaznih podataka, algoritma šifrovanja) i zatim da se šifruje (dešifruje) svaki blok čistog teksta koristeći samo ključ.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e6/ECB_decryption.svg/601px-ECB_decryption.svg.png)

Korišćenje ECB ima višestruke sigurnosne implikacije:

* **Blokovi iz šifrovanje poruke mogu biti uklonjeni**
* **Blokovi iz šifrovane poruke mogu biti premesteni**

# Otkrivanje ranjivosti

Zamislite da se prijavljujete na aplikaciju nekoliko puta i uvek dobijate **isti kolačić**. To je zato što je kolačić aplikacije **`<korisničko_ime>|<lozinka>`**.\
Zatim, generišete dva nova korisnika, oba sa **istom dugom lozinkom** i **skoro** **istim** **korisničkim imenom**.\
Otkrijete da su **blokovi od 8B** gde je **informacija oba korisnika** ista **jednaki**. Tada, pretpostavljate da se to možda dešava jer se koristi **ECB**.

Kao u sledećem primeru. Posmatrajte kako ova **2 dekodirana kolačića** imaju nekoliko puta blok **`\x23U\xE45K\xCB\x21\xC8`**
```
\x23U\xE45K\xCB\x21\xC8\x23U\xE45K\xCB\x21\xC8\x04\xB6\xE1H\xD1\x1E \xB6\x23U\xE45K\xCB\x21\xC8\x23U\xE45K\xCB\x21\xC8+=\xD4F\xF7\x99\xD9\xA9

\x23U\xE45K\xCB\x21\xC8\x23U\xE45K\xCB\x21\xC8\x04\xB6\xE1H\xD1\x1E \xB6\x23U\xE45K\xCB\x21\xC8\x23U\xE45K\xCB\x21\xC8+=\xD4F\xF7\x99\xD9\xA9
```
Ovo je zato što su **korisničko ime i lozinka tih kolačića sadržali više puta slovo "a"** (na primer). **Blokovi** koji su **različiti** su blokovi koji su sadržali **barem 1 različit karakter** (možda razdelnik "|" ili neka neophodna razlika u korisničkom imenu).

Sada napadač samo treba da otkrije da li je format `<korisničko ime><razdelnik><lozinka>` ili `<lozinka><razdelnik><korisničko ime>`. Da bi to uradio, može jednostavno **generisati nekoliko korisničkih imena** sa **sličnim i dugim korisničkim imenima i lozinkama dok ne otkrije format i dužinu razdelnika:**

| Dužina korisničkog imena: | Dužina lozinke: | Dužina korisničkog imena+Lozinke: | Dužina kolačića (nakon dekodiranja): |
| -------------------------- | ---------------- | ---------------------------------- | -------------------------------------- |
| 2                          | 2                | 4                                  | 8                                      |
| 3                          | 3                | 6                                  | 8                                      |
| 3                          | 4                | 7                                  | 8                                      |
| 4                          | 4                | 8                                  | 16                                     |
| 7                          | 7                | 14                                 | 16                                     |

# Iskorišćavanje ranjivosti

## Uklanjanje celih blokova

Znajući format kolačića (`<korisničko ime>|<lozinka>`), kako bi se predstavio kao korisnik `admin`, kreirajte novog korisnika pod imenom `aaaaaaaaadmin` i dobijte kolačić, zatim ga dekodirajte:
```
\x23U\xE45K\xCB\x21\xC8\xE0Vd8oE\x123\aO\x43T\x32\xD5U\xD4
```
Možemo videti obrazac `\x23U\xE45K\xCB\x21\xC8` koji je prethodno kreiran sa korisničkim imenom koje sadrži samo `a`.\
Zatim, možete ukloniti prvi blok od 8B i dobićete validan kolačić za korisničko ime `admin`:
```
\xE0Vd8oE\x123\aO\x43T\x32\xD5U\xD4
```
## Pomeranje blokova

U mnogim bazama podataka isto je tražiti `WHERE username='admin';` ili `WHERE username='admin    ';` _(Primetite dodatne razmake)_

Dakle, još jedan način da se predstavite kao korisnik `admin` bio bi:

* Generišite korisničko ime koje: `len(<username>) + len(<delimiter) % len(block)`. Sa veličinom bloka od `8B` možete generisati korisničko ime nazvano: `username       `, sa delimiterom `|` isečak `<username><delimiter>` će generisati 2 bloka od 8B.
* Zatim, generišite lozinku koja će popuniti tačan broj blokova koji sadrže korisničko ime koje želimo da predstavimo i razmake, kao što je: `admin   `

Kolačić ovog korisnika će biti sastavljen od 3 bloka: prva 2 su blokovi korisničkog imena + delimitera, a treći je lozinka (koja predstavlja korisničko ime): `username       |admin   `

**Zatim, jednostavno zamenite prvi blok sa poslednjim i predstavljate korisnika `admin`: `admin          |username`**

## Reference

* [http://cryptowiki.net/index.php?title=Electronic_Code_Book\_(ECB)](http://cryptowiki.net/index.php?title=Electronic_Code_Book_\(ECB\))