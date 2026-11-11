# MoorlFormsExample – technischer Projektkontext

## Status und Zweck

Quellstand: `composer.json` Version **1.7.0**, Anforderungen Shopware `6.7.*`, Foundation `1.7.*`, MoorlForms `2.7.*`. Das Repository ist ein Entwicklerbeispiel für eigene Formular-, Aktions- und Elementtypen. Die Quelltexte enthalten jedoch ältere Extension-Verträge und können mit dem vorliegenden MoorlForms-Stand **nicht unverändert als funktionierendes Add-on geladen werden**.

Ein isolierter PHP-8.2-Ladetest der tatsächlichen Klassen und ihrer lokalen Basisklassen endet für `FormActionExample` und `ElementTypeExample` mit `Declaration ... must be compatible`. Dafür wurden keine Shopware-Installation gestartet, Daten angelegt, Newsletter registriert oder externe SOAP-Aufrufe ausgelöst. Syntaxprüfung einzelner PHP-Dateien allein erkennt diese Vererbungsfehler nicht.

## Struktur und registrierte Bausteine

Die [Pluginbasis](src/MoorlFormsExample.php) enthält nur Konstanten; eigene Tabellen, Shopware-Tabellen und Vererbungen sind leer. Keine eigenen DAL-Definitionen, Migrationen, Systemkonfiguration, Administration-Controller, API-Routen oder Demodatensätze vorhanden. Das Beispiel benutzt vorhandene Form-/Elemententitäten des Basisplugins sowie Produktdaten.

Alle folgenden Bausteine sind in [services.xml](src/Resources/config/services.xml) zur Registrierung vorgesehen. Die Registrierung ist vom erfolgreichen Laden ihrer Klassen zu unterscheiden:

| Baustein | Kennung/Tag | Beabsichtigtes Beispielverhalten |
| --- | --- | --- |
| `FormTypeExample` | `example`, `moorl.fb.form`, Priorität 0 | Produktbezogener Formulartyp, `relatedEntity = product`. |
| `FormActionExample` | `example`, `moorl.fb.action`, Priorität 0 | Newsletteranmeldung beim Verarbeiten eines Formulars mit E-Mail. |
| `ElementTypeExample` | `example`, `moorl.fb.element`, Priorität 0 | Textähnliches Feld mit eigener Option und exemplarischer USt-ID-Prüfung. |
| `ExampleSubscriber` | `kernel.event_subscriber` | Reagiert auf `ProductPageLoadedEvent`. |
| `FbExampleService` | Öffentlicher Service | Lädt Beispiel-Formulare und hängt sie als Produkterweiterung an. |

Keine Flow-Aktion implementiert. `example` ist hier jeweils ein separater Name in unterschiedlichen Registries, keine austauschbare Formular-/Aktions-/Elementkennung.

## Formulartyp und Produktanreicherung

[FormTypeExample](src/Core/Content/Form/Type/FormTypeExample.php) erweitert `FormTypeExtension` und liefert:

- Name `example`.
- Verknüpfte Entität `Shopware\Core\Content\Product\ProductDefinition::ENTITY_NAME`, also `product`.
- Optionen `mail`, `submit`, `newsletter`, `stylesheet`.

Diese Optionsliste beschreibt angebotene Formulareinstellungen und führt selbst keine Newsletter-/Mail-Aktion aus. Aktionen müssen separat ausgewählt und funktionierende Handler vorhanden sein.

[ExampleSubscriber](src/Storefront/Subscriber/ExampleSubscriber.php) verarbeitet ausschließlich `ProductPageLoadedEvent`, nicht das importierte, aber unbenutzte `SalesChannelContextResolvedEvent`. Er ruft `FbExampleService::enrichProduct()` für das geladene Produkt auf.

[FbExampleService](src/Core/Service/FbExampleService.php) speichert den SalesChannelContext, lädt über `FbService::initFormsByType('example', context)` die Formulare und behält diese Sammlung im Service. Für jedes Formular erfolgt `enrichFromEntity(form, context, product)`. Bei nicht leerer Sammlung hängt er sie unter `product.extensions.MoorlFormsExample` an. Der Code enthält keinen eigenen Produktzuordnungsfilter; Auswahl-/Aktivitätsregeln stammen aus `FbService`. `DefinitionInstanceRegistry` und `SystemConfigService` werden injiziert, im Beispiel aber nicht verwendet. Kein Reset für eine bereits geladene Formularsammlung.

Zwei Storefront-Templates erweitern den Kaufbereich:

- [page/product-detail/buy-widget-form.html.twig](src/Resources/views/storefront/page/product-detail/buy-widget-form.html.twig), Block `page_product_detail_buy_form_inner`.
- [component/buy-widget/buy-widget-form.html.twig](src/Resources/views/storefront/component/buy-widget/buy-widget-form.html.twig), Block `buy_widget_buy_form_inner`.

Beide iterieren die Produkterweiterung und binden `@MoorlForms/plugin/moorl-fb/component/form/index.html.twig` ein. Die jeweils aktive Shopware-Templatehierarchie entscheidet, welcher Override greift. Eine doppelte oder erfolgreiche Darstellung wurde nicht im Browser geprüft.

## Formularaktion: beabsichtigter Ablauf und belegte Blocker

[FormActionExample](src/Core/Content/Form/Action/FormActionExample.php) injiziert `AbstractNewsletterSubscribeRoute` (DI konkret `NewsletterSubscribeRoute`) und beabsichtigt:

1. Ohne `email` im Request keine Anmeldung.
2. Erste SalesChannel-Domain als `storefrontUrl` verwenden.
3. Requestparameter in `RequestDataBag` übernehmen.
4. `option = NewsletterSubscribeRoute::OPTION_SUBSCRIBE` und Storefront-URL ergänzen.
5. `subscribe(dataBag, salesChannelContext, false)` aufrufen.

Dies ist eine schreibende Newsletteraktion und kein reiner Anzeige-/Dummyhandler. Das Plugin enthält keine eigene Prüfung einer separaten Zustimmung; etwaige Validierung und Opt-in-Regeln liegen im Formular bzw. Shopware-Newsletterpfad.

Die vorhandenen Methoden `init(FormEntity)` und `process(FormEntity)` passen nicht zum aktuellen [FormActionInterface](../MoorlForms/src/Core/Content/Form/Action/FormActionInterface.php) bzw. [FormActionExtension](../MoorlForms/src/Core/Content/Form/Action/FormActionExtension.php). Aktuell werden in beiden Methoden zusätzlich `Request` und `SalesChannelContext` verlangt. `parent::init($form)` wäre ebenfalls unvollständig. Die verwendeten Eigenschaften `$this->request` und `$this->salesChannelContext` existieren in der aktuellen Basisklasse nicht mehr.

Vor einer Wiederverwendung müssen Methodenverträge, Context-/Requestzugriff, Domainverfügbarkeit und Newslettervalidierung an den aktuellen Stand angepasst werden. Dieser Dokumentationsauftrag verändert keinen Produktionscode.

## Elementtyp, Konfiguration und externe Prüfung

[ElementTypeExample](src/Core/Content/Element/Type/ElementTypeExample.php) erweitert `ElementGroupFields`. Es übernimmt dessen Optionen und ergänzt `example`. Die aktuelle Basiskette liefert Gruppen-/Feld-/Validierungsoptionen; die Beispieloption hat keinen Defaultwert und keinen eigenen Datentypvertrag.

Beabsichtigte Initialisierung:

- Ohne PHP-Klasse `SOAPClient`: Elementtyp auf `ElementTypeText::NAME` ändern, übersetzten Namen/Placeholder auf `SOAPClient not installed` setzen.
- Mit SOAP: Placeholder aus `element.config.exampleValue`, Tooltip aus `element.translated.custom1` übernehmen.

Beabsichtigte Validierung: Parentvalidierung, SOAP-Verfügbarkeitsprüfung, Bereinigung mit dem vorhandenen Muster `/[^a-zA-Z0-9]]/`, Formatprüfung `/^[a-z]{2}[a-z0-9]{0,12}$/i`, danach `SoapClient` mit `https://ec.europa.eu/taxation_customs/vies/checkVatService.wsdl` und `checkVatApprox(countryCode, vatNumber)`. Ungültige Werte führen zu `invalidFormat` bzw. `invalidTaxIdNumber` mit `MoorlForms::MSG_CODE`; fehlendes SOAP löst eine einfache Exception aus. Das ungewöhnliche Bereinigungsmuster ist zu prüfen, nicht als verlässliche Normalisierung zu übernehmen.

Auch `init(ElementEntity)` und `validate(ElementDataStruct, ?array $tree = null)` sind inkompatibel zu den aktuellen [ElementTypeInterface](../MoorlForms/src/Core/Content/Element/Type/ElementTypeInterface.php)/Basisklassen. Heute benötigt `init` zusätzlich Formular, Request und Context; `validate` zusätzlich Formular, `ContextualValidatorInterface`, Request und Context. Bereits die Klassendeklaration scheitert. Der beschriebene SOAP-Fallback macht diesen Ladefehler nicht unschädlich.

Das [Elementtemplate](src/Resources/views/plugin/moorl-fb/component/element/example.html.twig) erweitert ein Feldtemplate aus **MoorlFormsClassic** und setzt `input_type` auf `text`. Classic ist im Composer-Manifest dieses Beispielplugins nicht deklariert; für diesen konkreten Templatepfad ist es trotzdem eine Abhängigkeit. Es gibt keine lokale SOAP-Antwortsimulation, keinen eigenen Timeout und keine Fehlerbehandlung um den SOAP-Aufruf.

## Administration und Storefront-JavaScript

- [Administration main.js](src/Resources/app/administration/src/main.js) lädt `extension/index.js`, dieses lädt Form- und Elementsettings-Overrides. Keine eigene Administration-Navigation oder Route.
- Einstieg bei funktionsfähigen Verträgen: **Inhalte → Formular Baukasten**, bestehende Routen `moorl.fb.form.list`, `moorl.fb.form.detail`, `moorl.fb.form.element`.
- [Formsettings](src/Resources/app/administration/src/extension/form-settings/index.html.twig): „Example card“, bei Aktion `example` Eingaben `form.custom1`, `form.config.exampleValue`, sonst Hinweis. Die Kartenwerte werden von der gezeigten Newsletteraktion nicht verarbeitet.
- [Elementsettings](src/Resources/app/administration/src/extension/element-settings/options/example/index.html.twig): Karte bei `elementOption('example')`, Eingaben `element.custom1`, `element.config.exampleValue`. Letzteres wird bei der Elementinitialisierung verwendet.
- Alte Administrationsbausteine (`sw-card`, `sw-field`, `sw-alert`, `v-model` und `{% parent %}`) sind unverändert vorhanden. Ihre Kompatibilität mit dem heutigen Admin wurde nicht behauptet oder im Browser geprüft. Ein Snippetfile ist nur für `en-GB` vorhanden.
- [Storefront main.js](src/Resources/app/storefront/src/main.js) registriert `MoorlFbExample` auf `[data-moorl-fb-example]`. [fb-example.plugin.js](src/Resources/app/storefront/src/fb-example/fb-example.plugin.js) färbt das erste gefundene `input` rot (`#FF0000`); Nullprüfung und weitere Logik fehlen. Das Beispieltemplate setzt dieses Data-Attribut nicht ausdrücklich selbst; dessen Vorhandensein muss im tatsächlichen Rendering geprüft werden.

## Dokumentation, Repository und sichere Verifikation

[README.md](README.md) beschreibt die Bausteine und den verifizierten Kompatibilitätsstatus nun ausdrücklich. Keine `docs`-Handbücher, `*.tpl.md`, `.shopware-extension.yml`, Demoimporte oder Tests im Repository vorhanden. Der generische `store-info.yml`-Workflow verweist dennoch auf Manifest und `docs/store*`/`docs/manual*`; diese Dateien sind hier nicht vorhanden. Kein Workflow wurde ausgelöst.

Nach einer separat beauftragten Migration wären erforderlich: Klassenladeprüfung gegen aktuelle Interfaces, Containerkompilierung, Rendering der Produktformulare und Settings, gezielte Newslettertests mit freigegebenen Daten sowie SOAP-Prüfung mit kontrolliertem Testverfahren. Für reine Syntax-/Dokumentationsprüfung weder Newsletterroute noch SOAP-VIES aufrufen.

Screenshotkandidaten sind derzeit nur als geplante Motive zu verstehen: Beispiel-Formulartyp/-aktion/-element im Basisplugin, Beispielkarten mit `custom1`/`exampleValue`, Produktdarstellung und rotes Feld. Wegen der belegten Klassenladefehler ist eine verfügbare Beispieloberfläche für diesen Stand nicht zugesagt.

## Visueller Admin-Abgleich am 10.11.2026

Die Anmeldung am lokalen Shopware-Admin 6.7.15.1 ist inzwischen erfolgreich. Dieses Plugin ist dort laut Pluginrepository nicht registriert und daher nicht als aktive Admin-Erweiterung verfügbar. Für dieses Add-on wurden keine Featureaufnahmen behauptet oder nachgebildet. Es wurde weder installiert noch aktiviert; die Quell-/Dokumentationsbefunde bleiben von der ausstehenden visuellen Abnahme getrennt.

Der laufende Shop erlaubt derzeit die neuen Basis-/Order-Aufnahmen. Der konkrete Stand ist in `../MoorlForms/CODEX.md` und `../MoorlForms/docs/admin-screenshots.json` festgehalten; die Bilder sind direkt in den bestehenden Basis-/Order-Handbüchern eingebunden. Für die vollständige Add-on-Abnahme ist eine entsprechend eingerichtete Testumgebung erforderlich; der Login ist kein Hindernis mehr.


## Dokumentationsstruktur ab 11.11.2026

Neue englische FHD-Aufnahmen liegen flach unter `docs/images/` in Basis und Order und werden in DE/EN gemeinsam verwendet. Nicht passende historische Abbildungen sind entfernt; für nicht aktive Add-ons werden keine fremden Basisansichten als Funktionsnachweis ausgegeben. Fachanleitungen und Feature-Übersicht bleiben in den bestehenden Seiten. Interne Seitenverweise der öffentlichen Doku verwenden sprachabhängige URLs unter `https://docs.moori.net/plugin/` bzw. `/en/plugin/`; `.en.md` ist ein Quellname und kein öffentliches Linkziel. Bilder und Downloadbeispiele bleiben lokale Assets.
