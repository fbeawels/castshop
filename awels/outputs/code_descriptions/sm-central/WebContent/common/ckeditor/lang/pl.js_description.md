# pl.js

## Review

## 1. Summary
This file is the Polish language pack for **CKEditor**.  
It registers a large object (`CKEDITOR.lang.pl`) that maps all user‑interface text strings used by the editor to their Polish translations. The code follows CKEditor’s standard pattern for language files:

* A single global assignment to `CKEDITOR.lang.pl`.
* An object literal containing nested objects for UI elements, dialogs, tables, etc.
* No executable logic or custom functions – it is purely data.

The file is deliberately minimal, designed to be loaded by the CKEditor core via its language loading mechanism.

## 2. Detailed Description
### Structure
The file begins with a copyright notice and then declares `CKEDITOR.lang.pl`.  
The object contains key/value pairs for:

| Top‑level section | Purpose |
|-------------------|---------|
| `dir` | Text direction (“ltr”) |
| `editorTitle` | Title shown when the editor is focused |
| `source`, `newPage`, `save`, … | General toolbar button labels |
| `common` | Generic dialog fields (URL, protocol, upload button, etc.) |
| `specialChar` | Special character dialog |
| `link`, `anchor`, `findAndReplace`, `table`, `button`, … | Dialogs for specific editor features |
| `image`, `flash`, `spellCheck`, `smiley` | Media and utility dialogs |
| `elementsPath`, `numberedlist`, `bulletedlist`, … | Miscellaneous UI strings |
| `clipboard`, `pastefromword`, `pasteText` | Clipboard handling messages |
| `templates`, `showBlocks`, `stylesCombo`, `format`, `font`, … | Content styling tools |
| `colorButton`, `colors` | Colour picker |
| `scayt` | Real‑time spell‑checking UI |
| `about`, `maximize`, `minimize`, `fakeobjects`, `resize`, `colordialog` | Miscellaneous controls |

Each nested object is itself a simple key/value map. For example, `link` contains titles, button labels, and validation messages for the “Insert/Edit Link” dialog.

### Execution Flow
* When CKEditor loads the Polish locale, it executes this script, which immediately creates or overrides `CKEDITOR.lang.pl`.
* The CKEditor core then merges this object into its language system, making the translated strings available to the UI.
* No runtime logic is executed beyond the initial assignment; the file is effectively a data blob.

### Assumptions & Constraints
* The file assumes the global `CKEDITOR` object is already defined.
* It expects the rest of CKEditor’s infrastructure to handle localisation (e.g., lookup of `CKEDITOR.lang.pl['bold']`).
* The code is written in ES3‑style JavaScript (no `const/let`, no modules) for maximum browser compatibility, matching CKEditor’s target environments.

## 3. Functions/Methods
The file contains **no functions or methods** – it is purely a declaration of a data object.  
If one were to treat each top‑level key as a “function” for the purposes of a review, the “functions” would simply return the string value when accessed:

```javascript
CKEDITOR.lang.pl.bold   // returns 'Pogrubienie'
CKEDITOR.lang.pl.link.url  // returns 'Adres URL'
```

No side effects exist; the only side effect is the assignment to `CKEDITOR.lang.pl`.

## 4. Dependencies
| Dependency | Type | Notes |
|------------|------|-------|
| `CKEDITOR` global | Core CKEditor library | The file is a plugin/extension; it does not provide its own module system. |
| None else | - | All strings are hard‑coded; no external libraries or APIs are invoked. |

### Platform Specifics
* Designed to run in all major browsers supported by CKEditor (IE 6+, Firefox, Chrome, Safari, etc.).
* No ES6 module syntax or features, so it works in legacy environments.

## 5. Additional Notes
### Strengths
* **Simplicity** – The code is straightforward and follows CKEditor’s convention, making it easy to maintain.
* **Self‑contained** – All translations are inline, so there is no need for external resource loading.
* **Consistency** – Reuse of common keys (`common`, `button`, etc.) ensures UI elements are translated uniformly.

### Weaknesses / Edge Cases
1. **Duplicate Keys** – Within the `link` section, the key `target` appears twice. JavaScript will keep only the last value, potentially discarding an earlier translation. This could lead to an incorrect label in the UI.
2. **Hard‑coded Color Map** – The `colors` object is large and could be generated programmatically or stored in an external JSON file to reduce file size.
3. **Missing Context** – Some strings lack context for translators (e.g., “link”, “anchor”), which could lead to ambiguous translations in other languages.
4. **No Internationalization Helpers** – No interpolation functions or placeholders for dynamic values (beyond simple `%1`, `%l` placeholders). If a future version of CKEditor uses richer i18n features, the file would need updating.

### Possible Enhancements
* **Refactor Duplicate Keys** – Ensure unique keys; merge duplicate sections or remove redundancy.
* **Externalize Color Map** – Store colors in a separate JSON file or generate them via a build script to keep the language file lean.
* **Use Modern Module System** – For CKEditor 5 or future builds, convert to an ES6 module or JSON import, which allows tree‑shaking and lazy loading.
* **Add Comments for Translators** – Provide contextual comments or a separate PO file for easier translation workflows.
* **Validation** – Include a lightweight linting step in the build process to catch duplicate keys or missing placeholders.

### Final Remarks
The file is a textbook example of a CKEditor language pack. Its purpose is entirely data‑driven, so there is little room for algorithmic bugs. The main maintenance concern is ensuring the object stays free of duplicate keys and is updated when new UI strings are added to the core editor. With those caveats addressed, the file is robust, readable, and fully functional within the CKEditor ecosystem.

## Code Critique



## Code Preview

```javascript
﻿/*
Copyright (c) 2003-2009, CKSource - Frederico Knabben. All rights reserved.
For licensing, see LICENSE.html or http://ckeditor.com/license
*/

CKEDITOR.lang.pl={dir:'ltr',editorTitle:'Wzbogacony edytor treści, %1',source:'Źródło dokumentu',newPage:'Nowa strona',save:'Zapisz',preview:'Podgląd',cut:'Wytnij',copy:'Kopiuj',paste:'Wklej',print:'Drukuj',underline:'Podkreślenie',bold:'Pogrubienie',italic:'Kursywa',selectAll:'Zaznacz wszystko',removeFormat:'Usuń formatowanie',strike:'Przekreślenie',subscript:'Indeks dolny',superscript:'Indeks górny',horizontalrule:'Wstaw poziomą linię',pagebreak:'Wstaw odstęp',unlink:'Usuń hiperłącze',undo:'Cofnij',redo:'Ponów',common:{browseServer:'Przeglądaj',url:'Adres URL',protocol:'Protokół',upload:'Wyślij',uploadSubmit:'Wyślij',image:'Obrazek',flash:'Flash',form:'Formularz',checkbox:'Pole wyboru (checkbox)',radio:'Pole wyboru (radio)',textField:'Pole tekstowe',textarea:'Obszar tekstowy',hiddenField:'Pole ukryte',button:'Przycisk',select:'Lista wyboru',imageButton:'Przycisk-obrazek',notSet:'<nie ustawione>',id:'Id',name:'Nazwa',langDir:'Kierunek tekstu',langDirLtr:'Od lewej do prawej (LTR)',langDirRtl:'Od prawej do lewej (RTL)',langCode:'Kod języka',longDescr:'Długi opis hiperłącza',cssClass:'Nazwa klasy CSS',advisoryTitle:'Opis obiektu docelowego',cssStyle:'Styl',ok:'OK',cancel:'Anuluj',generalTab:'Ogólne',advancedTab:'Zaawansowane',validateNumberFailed:'Ta wartość nie jest liczbą.',confirmNewPage:'Wszystkie niezapisane zmiany zostaną utracone. Czy na pewno wczytać nową stronę?',confirmCancel:'Pewne opcje zostały zmienione. Czy na pewno zamknąć okno dialogowe?',unavailable:'%1<span class="cke_accessibility">, niedostępne</span>'},specialChar:{toolbar:'Wstaw znak specjalny',title:'Wybierz znak specjalny'},link:{toolbar:'Wstaw/edytuj hiperłącze',menu:'Edytuj hiperłącze',title:'Hiperłącze',info:'Informacje ',target:'Cel',upload:'Wyślij',advanced:'Zaawansowane',type:'Typ hiperłącza',toAnchor:'Odnośnik wewnątrz strony',toEmail:'Adres e-mail',target:'Cel',targetNotSet:'<nie ustawione>',targetFrame:'<ramka>',targetPopup:'<wyskakujące okno>',targetNew:'Nowe okno (_blank)',targetTop:'Okno najwyższe w hierarchii (_top)',targetSelf:'To samo okno (_self)',targetParent:'Okno nadrzędne (_parent)',targetFrameName:'Nazwa Ramki Docelowej',targetPopupName:'Nazwa wyskakującego okna',popupFeatures:'Właściwości wyskakującego okna',popupResizable:'Skalowalny',popupStatusBar:'Pasek statusu',popupLocationBar:'Pasek adresu',popupToolbar:'Pasek narzędzi',popupMenuBar:'Pasek menu',popupFullScreen:'Pełny ekran (IE)',popupScrollBars:'Paski przewijania',popupDependent:'Okno zależne (Netscape)',popupWidth:'Szerokość',popupLeft:'Pozycja w poziomie',popupHeight:'Wysokość',popupTop:'Pozycja w pionie',id:'Id',langDir:'Kierunek tekstu',langDirNotSet:'<nie ustawione>',langDirLTR:'Od lewej do prawej (LTR)',langDirRTL:'Od prawej do lewej (RTL)',acccessKey:'Klawisz dostępu',name:'Nazwa',langCode:'Kierunek tekstu',tabIndex:'Indeks tabeli',advisoryTitle:'Opis obiektu docelowego',advisoryContentType:'Typ MIME obiektu docelowego',cssClasses:'Nazwa klasy CSS',charset:'Kodowanie znaków obiektu docelowego',styles:'Styl',selectAnchor:'Wybierz etykietę',anchorName:'Wg etykiety',anchorId:'Wg identyfikatora elementu',emailAddress:'Adres e-mail',emailSubject:'Temat',emailBody:'Treść',noAnchors:'(W dokumencie nie zdefiniowano żadnych etykiet)',noUrl:'Podaj adres URL',noEmail:'Podaj adres e-mail'},anchor:{toolbar:'Wstaw/edytuj kotwicę',menu:'Właściwości kotwicy',title:'Właściwości kotwicy',name:'Nazwa kotwicy',errorName:'Wpisz nazwę kotwicy'},findAndReplace:{title:'Znajdź i zamień',find:'Znajdź',replace:'Zamień',findWhat:'Znajdź:',replaceWith:'Zastąp przez:',notFoundMsg:'Nie znaleziono szukanego hasła.',matchCase:'Uwzględnij wielkość liter',matchWord:'Całe słowa',matchCyclic:'Cykliczne dopasowanie',replaceAll:'Zastąp wszystko',replaceSuccessMsg:'%1 wystąpień zastąpionych.'},table:{toolbar:'Tabela',title:'Właściwości tabeli',menu:'Właściwości tabeli',deleteTable:'Usuń tabelę',rows:'Liczba wierszy',columns:'Liczba kolumn',border:'Grubość ramki',align:'Wyrównanie',alignNotSet:'<brak ustawień>',alignLeft:'Do lewej',alignCenter:'Do środka',alignRight:'Do prawej',width:'Szerokość',widthPx:'piksele',widthPc:'%',height:'Wysokość',cellSpace:'Odstęp pomiędzy komórkami',cellPad:'Margines wewnętrzny komórek',caption:'Tytuł',summary:'Podsumowanie',headers:'Nagłowki',headersNone:'Brak',headersColumn:'Pierwsza kolumna',headersRow:'Pierwszy wiersz',headersBoth:'Oba',invalidRows:'Liczba wierszy musi być liczbą większą niż 0.',invalidCols:'Liczba kolumn musi być liczbą większą niż 0.',invalidBorder:'Liczba obramowań musi być liczbą.',invalidWidth:'Szerokość tabeli musi być liczbą.',invalidHeight:'Wysokość tabeli musi być liczbą.',invalidCellSpacing:'Odstęp komórek musi być liczbą.',invalidCellPadding:'Dopełnienie komórek musi być liczbą.',cell:{menu:'Komórka',insertBefore:'Wstaw komórkę z lewej',insertAfter:'Wstaw komórkę z prawej',deleteCell:'Usuń komórki',merge:'Połącz komórki',mergeRight:'Połącz z komórką z prawej',mergeDown:'Połącz z komórką poniżej',splitHorizontal:'Podziel komórkę poziomo',splitVertical:'Podziel komórkę pionowo',title:'Właściwości komórki',cellType:'Typ komórki',rowSpan:'Scalenie wierszy',colSpan:'Scalenie komórek',wordWrap:'Zawijanie słów',hAlign:'Wyrównanie poziome',vAlign:'Wyrównanie pionowe',alignTop:'Góra',alignMiddle:'Środek',alignBottom:'Dół',alignBaseline:'Linia bazowa',bgColor:'Kolor tła',borderColor:'Kolor obramowania',data:'Dane',header:'Nagłowek',yes:'Tak',no:'Nie',invalidWidth:'Szerokość komórki musi być liczbą.',invalidHeight:'Wysokość komórki musi być liczbą.',invalidRowSpan:'Scalenie wierszy musi być liczbą całkowitą.',invalidColSpan:'Scalenie komórek musi być liczbą całkowitą.',chooseColor:'Wybierz'},row:{menu:'Wiersz',insertBefore:'Wstaw wiersz powyżej',insertAfter:'Wstaw wiersz poniżej',deleteRow:'Usuń wiersze'},column:{menu:'Kolumna',insertBefore:'Wstaw kolumnę z lewej',insertAfter:'Wstaw kolumnę z prawej',deleteColumn:'Usuń kolumny'}},button:{title:'Właściwości przycisku',text:'Tekst (Wartość)',type:'Typ',typeBtn:'Przycisk',typeSbm:'Wyślij',typeRst:'Wyzeruj'},checkboxAndRadio:{checkboxTitle:'Właściwości pola wyboru (checkbox)',radioTitle:'Właściwości pola wyboru (radio)',value:'Wartość',selected:'Zaznaczone'},form:{title:'Właściwości formularza',menu:'Właściwości formularza',action:'Akcja',method:'Metoda',encoding:'Kodowanie',target:'Cel',targetNotSet:'<nie ustawione>',targetNew:'Nowe okno (_blank)',targetTop:'Okno najwyższe w hierarchii (_top)',targetSelf:'To samo okno (_self)',targetParent:'Okno nadrzędne (_parent)'},select:{title:'Właściwości listy wyboru',selectInfo:'Informacje',opAvail:'Dostępne opcje',value:'Wartość',size:'Rozmiar',lines:'linii',chkMulti:'Wielokrotny wybór',opText:'Tekst',opValue:'Wartość',btnAdd:'Dodaj',btnModify:'Zmień',btnUp:'Do góry',btnDown:'Do dołu',btnSetValue:'Ustaw wartość zaznaczoną',btnDelete:'Usuń'},textarea:{title:'Właściwości obszaru tekstowego',cols:'Kolumnu',rows:'Wiersze'},textfield:{title:'Właściwości pola tekstowego',name:'Nazwa',value:'Wartość',charWidth:'Szerokość w znakach',maxChars:'Max. szerokość',type:'Typ',typeText:'Tekst',typePass:'Hasło'},hidden:{title:'Właściwości pola ukrytego',name:'Nazwa',value:'Wartość'},image:{title:'Właściwości obrazka',titleButton:'Właściwości przycisku obrazka',menu:'Właściwości obrazka',infoTab:'Informacje o obrazku',btnUpload:'Wyślij',url:'Adres URL',upload:'Wyślij',alt:'Tekst zastępczy',width:'Szerokość',height:'Wysokość',lockRatio:'Zablokuj proporcje',resetSize:'Przywróć rozmiar',border:'Ramka',hSpace:'Odstęp poziomy',vSpace:'Odstęp pionowy',align:'Wyrównaj',alignLeft:'Do lewej',alignAbsBottom:'Do dołu',alignAbsMiddle:'Do środka w pionie',alignBaseline:'Do linii bazowej',alignBottom:'Do dołu',alignMiddle:'Do środka',alignRight:'Do prawej',alignTextTop:'Do góry tekstu',alignTop:'Do góry',preview:'Podgląd',alertUrl:'Podaj adres obrazka.',linkTab:'Hiperłącze',button2Img:'Czy chcesz przekonwertować zaznaczony przycisk graficzny do zwykłego obrazka?',img2Button:'Czy chcesz przekonwertować zaznaczony obrazek do przycisku graficznego?',urlMissing:'Podaj adres URL obrazka.'},flash:{properties:'Właściwości elementu Flash',propertiesTab:'Właściwości',title:'Właściwości elementu Flash',chkPlay:'Autoodtwarzanie',chkLoop:'Pętla',chkMenu:'Włącz menu',chkFull:'Dopuść pełny ekran',scale:'Skaluj',scaleAll:'Pokaż wszystko',scaleNoBorder:'Bez Ramki',scaleFit:'Dokładne dopasowanie',access:'Dostęp skryptów',accessAlways:'Zawsze',accessSameDomain:'Ta sama domena',accessNever:'Nigdy',align:'Wyrównaj',alignLeft:'Do lewej',alignAbsBottom:'Do dołu',alignAbsMiddle:'Do środka w pionie',alignBaseline:'Do linii bazowej',alignBottom:'Do dołu',alignMiddle:'Do środka',alignRight:'Do prawej',alignTextTop:'Do góry tekstu',alignTop:'Do góry',quality:'Jakość',qualityBest:'Najlepsza',qualityHigh:'Wysoka',qualityAutoHigh:'Auto wysoka',qualityMedium:'Średnia',qualityAutoLow:'Auto niska',qualityLow:'Niska',windowModeWindow:'Okno',windowModeOpaque:'Nieprzeźroczyste',windowModeTransparent:'Przeźroczyste',windowMode:'Tryb okna',flashvars:'Zmienne dla Flasha',bgcolor:'Kolor tła',width:'Szerokość',height:'Wysokość',hSpace:'Odstęp poziomy',vSpace:'Odstęp pionowy',validateSrc:'Podaj adres URL',validateWidth:'Szerokość musi być liczbą.',validateHeight:'Wysokość musi być liczbą.',validateHSpace:'Odstęp poziomy musi być liczbą.',validateVSpace:'Odstęp pionowy musi być liczbą.'},spellCheck:{toolbar:'Sprawdź pisownię',title:'Sprawdź pisownię',notAvailable:'Przepraszamy, ale usługa jest obecnie niedostępna.',errorLoading:'Błąd wczytywania hosta aplikacji usługi: %s.',notInDic:'Słowa nie ma w słowniku',changeTo:'Zmień na',btnIgnore:'Ignoruj',btnIgnoreAll:'Ignoruj wszystkie',btnReplace:'Zmień',btnReplaceAll:'Zmień wszystkie',btnUndo:'Cofnij',noSuggestions:'- Brak sugestii -',progress:'Trwa sprawdzanie...',noMispell:'Sprawdzanie zakończone: nie znaleziono błędów',noChanges:'Sprawdzanie zakończone: nie zmieniono żadnego słowa',oneChange:'Sprawdzanie zakończone: zmieniono jedno słowo',manyChanges:'Sprawdzanie zakończone: zmieniono %l słów',ieSpellDownload:'Słownik nie jest zainstalowany. Chcesz go ściągnąć?'},smiley:{toolbar:'Emotikona',title:'Wstaw emotikonę'},elementsPath:{eleTitle:'element %1'},numberedlist:'Lista numerowana',bulletedlist:'Lista wypunktowana',indent:'Zwiększ wcięcie',outdent:'Zmniejsz wcięcie',justify:{left:'Wyrównaj do lewej',center:'Wyrównaj do środka',right:'Wyrównaj do prawej',block:'Wyrównaj do lewej i prawej'},blockquote:'Cytat',clipboard:{title:'Wklej',cutError:'Ustawienia bezpieczeństwa Twojej przeglądarki nie pozwalają na automatyczne wycinanie tekstu. Użyj skrótu klawiszowego Ctrl+X.',copyError:'Ustawienia bezpieczeństwa Twojej przeglądarki nie pozwalają na automatyczne kopiowanie tekstu. Użyj skrótu klawiszowego Ctrl+C.',pasteMsg:'Proszę wkleić w poniższym polu używając klawiaturowego skrótu (<STRONG>Ctrl+V</STRONG>) i kliknąć <STRONG>OK</STRONG>.',securityMsg:'Zabezpieczenia przeglądarki uniemożliwiają wklejenie danych bezpośrednio do edytora. Proszę dane wkleić ponownie w tym okienku.'},pastefromword:{toolbar:'Wklej z Worda',title:'Wklej z Worda',advice:'Proszę wkleić w poniższym polu używając klawiaturowego skrótu (<STRONG>Ctrl+V</STRONG>) i kliknąć <STRONG>OK</STRONG>.',ignoreFontFace:"Ignoruj definicje 'Font Face'",removeStyle:'Usuń definicje Stylów'},pasteText:{button:'Wklej jako czysty tekst',title:'Wklej jako czysty tekst'},templates:{button:'Szablony',title:'Szablony zawartości',insertOption:'Zastąp aktualną zawartość',selectPromptMsg:'Wybierz szablon do otwarcia w edytorze<br>(obecna zawartość okna edytora zostanie utracona):',emptyListMsg:'(Brak zdefiniowanych szablonów)'},showBlocks:'Pokaż bloki',stylesCombo:{label:'Styl',voiceLabel:'Styl',panelVoiceLabel:'Wybierz styl',panelTitle1:'Style blokowe',panelTitle2:'Style liniowe',panelTitle3:'Style obiektowe'},format:{label:'Format',voiceLabel:'Format',panelTitle:'Format',panelVoiceLabel:'Wybierz paragraf do sformatowania',tag_p:'Normalny',tag_pre:'Tekst sformatowany',tag_address:'Adres',tag_h1:'Nagłówek 1',tag_h2:'Nagłówek 2',tag_h3:'Nagłówek 3',tag_h4:'Nagłówek 4',tag_h5:'Nagłówek 5',tag_h6:'Nagłówek 6',tag_div:'Normalny (DIV)'},font:{label:'Czcionka',voiceLabel:'Czcionka',panelTitle:'Czcionka',panelVoiceLabel:'Wybierz czcionkę'},fontSize:{label:'Rozmiar',voiceLabel:'Rozmiar czcionki',panelTitle:'Rozmiar',panelVoiceLabel:'Wybierz rozmiar czcionki'},colorButton:{textColorTitle:'Kolor tekstu',bgColorTitle:'Kolor tła',auto:'Automatycznie',more:'Więcej kolorów...'},colors:{'000':'Black',800000:'Maroon','8B4513':'Saddle Brown','2F4F4F':'Dark Slate Gray','008080':'Teal','000080':'Navy','4B0082':'Indigo',696969:'Dim Gray',B22222:'Fire Brick',A52A2A:'Brown',DAA520:'Golden Rod','006400':'Dark Green','40E0D0':'Turquoise','0000CD':'Medium Blue',800080:'Purple',808080:'Gray',F00:'Red',FF8C00:'Dark Orange',FFD700:'Gold','008000':'Green','0FF':'Cyan','00F':'Blue',EE82EE:'Violet',A9A9A9:'Dark Gray',FFA07A:'Light Salmon',FFA500:'Orange',FFFF00:'Yellow','00FF00':'Lime',AFEEEE:'Pale Turquoise',ADD8E6:'Light Blue',DDA0DD:'Plum',D3D3D3:'Light Grey',FFF0F5:'Lavender Blush',FAEBD7:'Antique White',FFFFE0:'Light Yellow',F0FFF0:'Honeydew',F0FFFF:'Azure',F0F8FF:'Alice Blue',E6E6FA:'Lavender',FFF:'White'},scayt:{title:'Sprawdź pisownię podczas pisania (SCAYT)',enable:'Włącz SCAYT',disable:'Wyłącz SCAYT',about:'Na temat SCAYT',toggle:'Przełącz SCAYT',options:'Opcje',langs:'Języki',moreSuggestions:'Więcej sugestii',ignore:'Ignoruj',ignoreAll:'Ignoruj wszystkie',addWord:'Dodaj słowo',emptyDic:'Nazwa słownika nie może być pusta.',optionsTab:'Opcje',languagesTab:'Języki',dictionariesTab:'Słowniki',aboutTab:'Na temat SCAYT'},about:{title:'Na temat CKEditor',dlgTitle:'Na temat CKEditor',moreInfo:'Informacje na temat licencji można znaleźć na naszej stronie:',copy:'Copyright &copy; $1. Wszelkie prawa zastrzeżone.'},maximize:'Maksymalizuj',minimize:'Minimalizuj',fakeobjects:{anchor:'Kotwica',flash:'Animacja Flash',div:'Separator stron',unknown:'Nieznany obiekt'},resize:'Przeciągnij, aby zmienić rozmiar',colordialog:{title:'Wybierz kolor',highlight:'Zaznacz',selected:'Wybrany',clear:'Wyczyść'}};



```
