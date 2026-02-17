# Staging Test Checklist - Schakel AI Meeting Automation Platform

## Doel

Deze checklist is bedoeld om **handmatig te verifi&euml;ren** dat alle functionaliteit op de staging-omgeving (`map-dev.schakel.ai`) correct werkt voordat we de `dev`-branch mergen naar `main` (productie).

**Scope**: 163 commits, 99 bestanden gewijzigd, +25.000 regels code.

**Staging URL**: `https://map-dev.schakel.ai`

**Belangrijke afspraken**:
- Voer tests uit in een **incognito/privebrowser** om cache-problemen te voorkomen
- Test zowel in **desktop** als **mobiel formaat** (responsive)
- Noteer eventuele **console errors** (F12 > Console) bij elke pagina
- Gebruik deze checklist als afrekenlijst: vink af wat werkt, noteer issues onderaan

---

## 1. Login & Authenticatie

> Prioriteit: KRITISCH - zonder werkende auth is niets testbaar.

### Inloggen

- [ ] **1.1** Ga naar `https://map-dev.schakel.ai` -- de landingspagina wordt getoond (niet-ingelogd)
  - Verwacht: "Meeting Intelligence Platform" landingspagina met hero ("Gesprekken zijn geen eindpunt, maar het startpunt"), links naar `/pricing`, `/login`
- [ ] **1.2** Klik op "Inloggen" of navigeer naar `/login` -- het loginformulier wordt getoond
  - Verwacht: E-mail en wachtwoord velden, "Inloggen" knop, "Wachtwoord vergeten" link
- [ ] **1.3** Log in met geldige credentials
  - Verwacht: Redirect naar `/dashboard`, gebruikersnaam zichtbaar in user menu (rechts bovenin)
- [ ] **1.4** Probeer in te loggen met een fout wachtwoord
  - Verwacht: Duidelijke foutmelding, geen crash, geen redirect
- [ ] **1.5** Probeer in te loggen met een niet-bestaand e-mailadres
  - Verwacht: Foutmelding (maar geen hint of het e-mailadres bestaat)

### Sessie & CSRF

- [ ] **1.6** Na inloggen: ververs de pagina (F5) -- je blijft ingelogd
  - Verwacht: Dashboard laadt opnieuw, geen login redirect
- [ ] **1.7** Open een nieuw tabblad naar `https://map-dev.schakel.ai/dashboard` -- sessie is nog geldig
  - Verwacht: Dashboard laadt zonder opnieuw in te loggen
- [ ] **1.8** Sluit alle tabbladen, wacht 5 minuten, open de URL opnieuw -- sessie is nog geldig
  - Verwacht: Dashboard laadt (sessie is persistent)

### Uitloggen

- [ ] **1.9** Klik op het user-icoon (rechts bovenin) en kies "Uitloggen"
  - Verwacht: Redirect naar loginpagina, toast "Uitgelogd"
- [ ] **1.10** Na uitloggen: navigeer direct naar `/dashboard`
  - Verwacht: Redirect naar loginpagina (geen toegang zonder sessie)

### Wachtwoord vergeten

- [ ] **1.11** Klik op "Wachtwoord vergeten" op de loginpagina
  - Verwacht: Formulier met e-mail veld
- [ ] **1.12** Voer een geldig e-mailadres in en verstuur
  - Verwacht: Bevestigingsmelding (ongeacht of het adres bestaat)

### MFA (indien ingeschakeld voor de tenant)

- [ ] **1.13** Log in bij een tenant waar MFA verplicht is -- MFA stap verschijnt
  - Verwacht: E-mail met verificatiecode, invoerveld voor code
- [ ] **1.14** Voer de juiste MFA-code in
  - Verwacht: Succesvol ingelogd, redirect naar dashboard
- [ ] **1.15** Voer een foute MFA-code in
  - Verwacht: Foutmelding, mag opnieuw proberen

---

## 2. Dashboard

> Prioriteit: HOOG - eerste scherm dat gebruikers zien na inloggen.

### Laden & weergave

- [ ] **2.1** Na inloggen laadt het dashboard zonder fouten
  - Verwacht: Geen witte pagina, geen console errors, laad-indicatoren verdwijnen
- [ ] **2.2** KPI-kaart "Totaal Meetings" toont een getal
  - Verwacht: Getal komt overeen met het werkelijke aantal meetings
- [ ] **2.3** Als Productive is geconfigureerd: KPI-kaart "Taken" toont "(review) / (totaal)"
  - Verwacht: Taken teller is zichtbaar, klikbaar of informatief
- [ ] **2.4** "Komende Opnames" widget toont geplande Recall.ai opnames (indien bot geconfigureerd)
  - Verwacht: Lijst met aankomende meetings en hun status (Gepland, Opnemen, etc.)
- [ ] **2.5** Eigenaar-verdeling grafiek laadt
  - Verwacht: Staaf/donut grafiek met eigenaar namen en aantallen
- [ ] **2.6** Categorie-verdeling grafiek laadt
  - Verwacht: Grafiek met categorieen en aantallen
- [ ] **2.7** "Recente fouten" widget toont meetings met status "error" (als die er zijn)
  - Verwacht: Lijst met foutmeldingen en een "Opnieuw" knop per item
- [ ] **2.8** Statistieken panel laadt (gem. meetings/week, drukste dag, etc.)
  - Verwacht: Cijfers zijn zichtbaar, geen "NaN" of "undefined"
- [ ] **2.9** Categorie trend grafiek toont data over tijd
  - Verwacht: Lijn- of staafgrafiek met datapunten

### Dashboard filters

- [ ] **2.10** Klik op een eigenaar in de eigenaar-verdeling grafiek
  - Verwacht: Dashboard filtert op die eigenaar, filter-samenvatting verschijnt bovenaan
- [ ] **2.11** Klik op een categorie in de categorie-verdeling grafiek
  - Verwacht: Dashboard filtert op die categorie
- [ ] **2.12** Verwijder het filter (klik op X in de filter-samenvatting)
  - Verwacht: Alle data is weer zichtbaar

### Quick actions

- [ ] **2.13** Klik op "Nieuwe Meeting" knop vanuit het dashboard
  - Verwacht: Manual Ingest dialog opent (Fireflies ID of handmatig transcript)
- [ ] **2.14** Bij een meeting met "error" status: klik op de "Opnieuw" knop in Recente Fouten
  - Verwacht: Meeting wordt opnieuw verwerkt, status verandert naar "processing"

---

## 3. Meetings

> Prioriteit: HOOG - kernfunctionaliteit van het platform.

### Meetings lijst (Verwerkte meetings tab)

- [ ] **3.1** Navigeer naar "Meetings" via de sidebar
  - Verwacht: Pagina laadt, tab "Verwerkte meetings" is standaard actief
- [ ] **3.2** Meetings worden getoond in een tabel met kolommen: titel, datum, status, categorie, eigenaar
  - Verwacht: Tabel is gevuld (als er meetings zijn), juiste datumweergave in lokale tijdzone
- [ ] **3.3** Zoek op titel met het zoekveld
  - Verwacht: Tabel filtert real-time op zoekterm
- [ ] **3.4** Filter op klant via de dropdown
  - Verwacht: Alleen meetings van die klant worden getoond
- [ ] **3.5** Filter op status via de dropdown (Wachtend/Bezig/Voltooid/Fout/Deels mislukt)
  - Verwacht: Tabel toont alleen meetings met die status
- [ ] **3.6** Filter op categorie via de dropdown
  - Verwacht: Alleen meetings in die categorie
- [ ] **3.7** Combineer meerdere filters (bijv. klant + status)
  - Verwacht: Filters werken samen (AND-logica)
- [ ] **3.8** Reset alle filters
  - Verwacht: Alle meetings worden weer getoond

### Geplande meetings tab (Recall.ai)

- [ ] **3.9** Klik op "Geplande meetings" tab
  - Verwacht: Overzicht van aankomende meetings uit de kalender met bot-status
- [ ] **3.10** Toggle bot aan/uit voor een specifieke meeting (klik op de toggle knop)
  - Verwacht: Bot wordt ingepland of verwijderd, status icon wijzigt
- [ ] **3.11** Vernieuw de kalenderdata (als er een refresh knop is)
  - Verwacht: Lijst wordt bijgewerkt met nieuwe kalender-events

### Nieuwe meeting toevoegen

- [ ] **3.12** Klik op "Nieuwe Meeting" knop
  - Verwacht: Dialoog opent met twee tabs: "Fireflies" en "Handmatig"
- [ ] **3.13** Tab "Fireflies": voer een Fireflies meeting ID in en verzend
  - Verwacht: Meeting wordt opgehaald en verwerking start (als Fireflies geconfigureerd is)
- [ ] **3.14** Tab "Handmatig": vul titel, datum, transcript in en verzend
  - Verwacht: Meeting wordt aangemaakt, status "pending" of "processing"
- [ ] **3.15** Validatie: probeer te verzenden zonder verplichte velden
  - Verwacht: Validatiefoutmeldingen bij lege velden

### Meeting verwijderen

- [ ] **3.16** Selecteer een meeting en klik op "Verwijderen"
  - Verwacht: Bevestigingsdialoog verschijnt ("Weet je het zeker?")
- [ ] **3.17** Bevestig de verwijdering
  - Verwacht: Meeting verdwijnt uit de lijst, toast bevestiging

---

## 4. Meeting Detail

> Prioriteit: HOOG - hier worden AI-resultaten getoond.

### Detailpagina laden

- [ ] **4.1** Klik op een meeting in de lijst
  - Verwacht: Detailpagina laadt met meeting titel, datum, status badge
- [ ] **4.2** Terugknop werkt (pijl links bovenin)
  - Verwacht: Terug naar meetings lijst
- [ ] **4.3** Meeting metadata is zichtbaar: datum, tijd, deelnemers, categorie, eigenaar, klant
  - Verwacht: Alle velden zijn gevuld (of leeg als niet beschikbaar), datums in lokale tijdzone

### Tabs

- [ ] **4.4** Tab "Notities" toont AI-gegenereerde samenvatting
  - Verwacht: Geformateerde HTML-notities, geen lege pagina
- [ ] **4.5** Tab "Taken" toont gegenereerde taken (als Productive geconfigureerd)
  - Verwacht: Lijst met taak-titels en review-status
- [ ] **4.6** Tab "Transcript" toont het volledige transcript
  - Verwacht: Transcript met sprekers en tijdstempels
- [ ] **4.7** Tab "Audio" toont audio player (indien beschikbaar)
  - Verwacht: Player laadt en kan afspelen voor meetings met audioStatus "stored" of "available" (Recall URL fallback). Download knop beschikbaar.
- [ ] **4.8** Tab "Logs" toont de pipeline stappen
  - Verwacht: Timeline met statussen per pipeline stap (categorize, notes, etc.)

### Acties op meeting detail

- [ ] **4.9** Klik op "Titel bewerken" (potlood-icoon naast de titel) en wijzig de titel
  - Verwacht: Titel wordt opgeslagen, toast bevestiging
- [ ] **4.10** Klik op "Klant bewerken" en wijzig de klantnaam
  - Verwacht: Klantnaam wordt opgeslagen
- [ ] **4.11** Klik op "Categorie wijzigen" -- dialoog opent
  - Verwacht: Dropdown met beschikbare categorieen, optie om opnieuw te verwerken
- [ ] **4.12** Download transcript (download knop)
  - Verwacht: Bestand wordt gedownload
- [ ] **4.13** Download notities als HTML of PDF
  - Verwacht: Bestand wordt gedownload in het gekozen formaat
- [ ] **4.14** "Opnieuw uploaden naar SharePoint" knop (als SharePoint geconfigureerd)
  - Verwacht: Upload start, toast bevestiging bij succes
- [ ] **4.15** "E-mail concept" knop -- draft dialoog opent
  - Verwacht: Dialoog met voorgevuld e-mail concept op basis van de meeting

### Pipeline herverwerking

- [ ] **4.16** Klik op "Alles opnieuw verwerken" knop
  - Verwacht: Hele pipeline start opnieuw, status gaat naar "processing"
- [ ] **4.17** Klik op een individuele pipeline stap opnieuw uitvoeren (als beschikbaar)
  - Verwacht: Alleen die stap wordt opnieuw uitgevoerd
- [ ] **4.18** Badge "Herverwerkt (Nx)" verschijnt na herverwerking
  - Verwacht: Counter verhoogt bij elke herverwerking

---

## 5. Recall.ai Bot (Opnemen)

> Prioriteit: HOOG - nieuwe kernfunctionaliteit.

### Bot instellingen (Settings > Meeting Bot)

- [ ] **5.1** Navigeer naar Instellingen > Opnemen > Meeting Bot
  - Verwacht: Sectie laadt met Recall.ai configuratie-opties
- [ ] **5.2** Toggle "Auto-join" aan/uit
  - Verwacht: Instelling wordt opgeslagen, toast bevestiging
- [ ] **5.3** Controleer "Bot join lead time" instelling (minuten voor de meeting)
  - Verwacht: Waarde is instelbaar en wordt opgeslagen

### Handmatig bot inplannen

- [ ] **5.4** Gebruik de "Handmatig bot inplannen" functie (in Settings > Meeting Bot)
  - Verwacht: Dialoog opent met velden voor meeting URL en starttijd
- [ ] **5.5** Voer een meeting URL in (bijv. Teams/Zoom link) en plan in
  - Verwacht: Bot wordt ingepland, verschijnt in de geplande meetings lijst
- [ ] **5.6** Voer een ongeldige URL in
  - Verwacht: Validatiefoutmelding

### Bot toggle op geplande meetings

- [ ] **5.7** Ga naar Meetings > Geplande meetings
  - Verwacht: Kalender-events worden getoond
- [ ] **5.8** Toggle bot aan voor een meeting die geen bot heeft
  - Verwacht: Bot wordt ingepland, icoon verandert
- [ ] **5.9** Toggle bot uit voor een meeting die een bot heeft
  - Verwacht: Bot wordt verwijderd, icoon verandert

### Dashboard widget

- [ ] **5.10** Controleer of de "Komende Opnames" widget op het dashboard zichtbaar is
  - Verwacht: Widget toont aankomende opnames met tijden en platforms (Teams/Zoom/Meet)

---

## 6. Pipeline Processing

> Prioriteit: HOOG - controleer of AI-verwerking end-to-end werkt.

- [ ] **6.1** Maak een nieuwe meeting aan via "Nieuwe Meeting" (handmatig transcript)
  - Verwacht: Meeting verschijnt in de lijst met status "pending" of "processing"
- [ ] **6.2** Wacht tot de verwerking voltooid is (of refresh de pagina)
  - Verwacht: Status verandert naar "completed", "Live" badge verdwijnt
- [ ] **6.3** Open de verwerkte meeting en controleer of AI-notities gegenereerd zijn
  - Verwacht: Tab "Notities" bevat inhoud (niet leeg)
- [ ] **6.4** Controleer of een categorie is toegewezen
  - Verwacht: Categorie badge is zichtbaar in de metadata
- [ ] **6.5** Controleer of een eigenaar is toegewezen
  - Verwacht: Eigenaar is zichtbaar in de metadata
- [ ] **6.6** Controleer de pipeline logs (tab "Logs" op de meeting detail pagina)
  - Verwacht: Alle stappen tonen status "completed" of "skipped" (geen "error")
- [ ] **6.7** Herverwerk de meeting via "Alles opnieuw verwerken"
  - Verwacht: Pipeline draait opnieuw, resultaten worden bijgewerkt

---

## 7. Inzichten (AI Agent)

> Prioriteit: MIDDEL - optionele feature, maar belangrijk voor waarde-propositie.

### Toegang

- [ ] **7.1** Controleer of "Inzichten" zichtbaar is in de sidebar
  - Verwacht: Menu-item met sparkle-icoon (alleen als feature ingeschakeld voor tenant)
- [ ] **7.2** Klik op "Inzichten" in de sidebar
  - Verwacht: Chat-interface laadt

### Chat functionaliteit

- [ ] **7.3** Maak een nieuw gesprek aan (klik op "+" of "Nieuw gesprek")
  - Verwacht: Lege chat thread wordt aangemaakt
- [ ] **7.4** Stel een vraag over meetings, bijv. "Hoeveel meetings had ik deze maand?"
  - Verwacht: Agent antwoordt met relevante informatie, bronvermeldingen (citations) zijn klikbaar
- [ ] **7.5** Stel een opvolgvraag in dezelfde thread
  - Verwacht: Agent herinnert de context van het gesprek
- [ ] **7.6** Controleer of citations (bronvermeldingen) links naar meeting detail pagina's bevatten
  - Verwacht: Klikbare links die naar de juiste meeting navigeren
- [ ] **7.7** Hernoem een gesprek (klik op het potlood-icoon)
  - Verwacht: Titel van het gesprek wordt bijgewerkt
- [ ] **7.8** Verwijder een gesprek
  - Verwacht: Gesprek verdwijnt uit de lijst

### Mutaties (agent acties)

- [ ] **7.9** Vraag de agent om een actie uit te voeren (bijv. "Wijzig de categorie van meeting X")
  - Verwacht: Agent stelt een actie voor ("Pending Action"), wacht op goedkeuring
- [ ] **7.10** Keur de voorgestelde actie goed
  - Verwacht: Actie wordt uitgevoerd, bevestiging in de chat
- [ ] **7.11** Weiger een voorgestelde actie
  - Verwacht: Actie wordt niet uitgevoerd, agent bevestigt annulering

---

## 8. Instellingen

> Prioriteit: MIDDEL - alle configuratie-secties moeten laden en opslaan.

### Sidebar navigatie

- [ ] **8.1** Navigeer naar "Instellingen" via de sidebar
  - Verwacht: Settings pagina laadt als een enkele scrollbare pagina
- [ ] **8.2** Sidebar toont een platte lijst met alle instellingen-items (geen groepen)
  - Verwacht: Items: Meeting Bot, Categorieën, Werkruimtes, AI Samenvatting, Huisstijl, Automatisering, Eigenaar Toewijzing, Productive, Gebruikers, API Verbindingen, Abonnement, Beveiliging, Privacy & Gegevens, Hulp
- [ ] **8.3** Klik op een item in de sidebar (bijv. "Categorieën")
  - Verwacht: Pagina scrollt naar de juiste sectie, item is gemarkeerd in de sidebar

### Instellingen secties

- [ ] **8.4** **Meeting Bot** sectie laadt
  - Verwacht: Recall.ai instellingen zichtbaar (auto-join, lead time, etc.)
- [ ] **8.5** **Categorieën** sectie laadt
  - Verwacht: Lijst met bestaande categorieën, mogelijkheid om nieuwe toe te voegen
- [ ] **8.6** Voeg een nieuwe categorie toe
  - Verwacht: Categorie verschijnt in de lijst na opslaan
- [ ] **8.7** Probeer een categorie aan te maken zonder naam of mappad
  - Verwacht: Inline validatiefout ("Naam is verplicht" / "Mappad is verplicht"), dialoog blijft open
- [ ] **8.8** Bewerk een bestaande categorie (naam, beschrijving)
  - Verwacht: Wijzigingen worden opgeslagen
- [ ] **8.9** Verwijder een categorie
  - Verwacht: Categorie verdwijnt (eventueel met bevestigingsdialoog)
- [ ] **8.10** Wijzig de volgorde van categorieën (drag & drop of pijltjes)
  - Verwacht: Nieuwe volgorde wordt opgeslagen
- [ ] **8.11** **AI Samenvatting** sectie laadt
  - Verwacht: AI prompt/instructies veld, model instellingen
- [ ] **8.12** Wijzig de AI samenvatting instructies en sla op
  - Verwacht: Toast "Opgeslagen", wijzigingen blijven na pagina refresh
- [ ] **8.13** **Huisstijl** sectie laadt
  - Verwacht: Logo upload, kleuren, branding instellingen
- [ ] **8.14** Upload een logo
  - Verwacht: Logo wordt getoond na upload, toast bevestiging
- [ ] **8.15** Upload een bot avatar
  - Verwacht: Avatar wordt getoond na upload
- [ ] **8.16** **Automatisering** sectie laadt
  - Verwacht: Automatisering opties (bijv. auto-upload naar SharePoint, auto-email)
- [ ] **8.17** Toggle een automatisering aan/uit en sla op
  - Verwacht: Instelling wordt opgeslagen
- [ ] **8.18** **Eigenaar Toewijzing** sectie laadt
  - Verwacht: Regels voor automatische eigenaar-toewijzing
- [ ] **8.19** **Productive** sectie laadt (indien geconfigureerd)
  - Verwacht: Productive.io API configuratie
- [ ] **8.20** **Gebruikers** sectie laadt
  - Verwacht: Lijst met tenant gebruikers, rollen, e-mailadressen
- [ ] **8.21** Nodig een nieuwe gebruiker uit (klik "Uitnodigen")
  - Verwacht: Formulier met e-mail, naam, rol velden; uitnodiging wordt verstuurd
- [ ] **8.22** Wijzig de rol van een bestaande gebruiker
  - Verwacht: Rol wordt bijgewerkt (admin/member)
- [ ] **8.23** Verwijder een gebruiker uit de tenant
  - Verwacht: Gebruiker verdwijnt uit de lijst (met bevestiging)
- [ ] **8.24** **API Verbindingen** sectie laadt
  - Verwacht: Overzicht van externe API connecties (Microsoft Graph, etc.)
- [ ] **8.25** **Abonnement** sectie laadt (alleen voor admins)
  - Verwacht: Huidig abonnement tier, status badge, verloopdatum
- [ ] **8.26** "Beheer abonnement" knop opent Stripe Customer Portal
  - Verwacht: Redirect naar Stripe portaal (in test modus)
- [ ] **8.27** **Beveiliging** sectie laadt (alleen voor admins)
  - Verwacht: MFA instellingen, wachtwoord beleid
- [ ] **8.28** Toggle "MFA verplicht voor alle gebruikers" aan/uit
  - Verwacht: Instelling wordt opgeslagen
- [ ] **8.29** **Privacy & Gegevens** sectie laadt
  - Verwacht: "Gegevens downloaden" knop, "Account verwijderen" knop
- [ ] **8.30** Klik op "Gegevens downloaden"
  - Verwacht: JSON bestand wordt gedownload met persoonlijke data
- [ ] **8.31** **Hulp** sectie laadt
  - Verwacht: Hulpinformatie of contactgegevens

---

## 9. Billing / Stripe

> Prioriteit: MIDDEL - essentieel voor SaaS, maar staging draait in test modus.

- [ ] **9.1** Navigeer naar Instellingen > Account > Abonnement
  - Verwacht: Billing status is zichtbaar (tier, status, periode)
- [ ] **9.2** Status badge toont correct: "Actief", "Proefperiode", "Waarschuwing" of "Geblokkeerd"
  - Verwacht: Badge kleur komt overeen met de status
- [ ] **9.3** Klik op "Beheer abonnement" -- Stripe Customer Portal opent
  - Verwacht: Redirect naar Stripe (test modus), portaal laadt
- [ ] **9.4** Navigeer naar `/pricing` (publieke pagina)
  - Verwacht: Pricing tiers worden getoond (Essentials €29/mo, Professional €79/mo, Enterprise op aanvraag)
- [ ] **9.5** Klik op een "Kies plan" knop op de pricing pagina
  - Verwacht: Stripe Checkout sessie start (test modus)
- [ ] **9.6** Bij een geblokkeerd abonnement: SubscriptionGuard blokkeert toegang
  - Verwacht: Geblokkeerd scherm met uitleg en link naar billing
- [ ] **9.7** Bij een verlopend abonnement: waarschuwingsbanner wordt getoond
  - Verwacht: Gele banner bovenaan met waarschuwingstekst en dagen tot blokkering

---

## 10. Gebruikersbeheer (vanuit Settings)

> Prioriteit: MIDDEL - multi-tenant gebruikersbeheer.

- [ ] **10.1** Navigeer naar Instellingen > Organisatie > Gebruikers
  - Verwacht: Tabel met alle gebruikers van de huidige tenant
- [ ] **10.2** Klik op "Uitnodigen" en vul het formulier in
  - Stappen: Voer e-mail in, optioneel naam, selecteer rol (admin/member), klik "Uitnodigen"
  - Verwacht: Uitnodiging wordt verstuurd, toast bevestiging
- [ ] **10.3** Controleer of de uitgenodigde gebruiker in de lijst verschijnt (als pending/uitgenodigd)
  - Verwacht: Gebruiker is zichtbaar met status "Uitgenodigd"
- [ ] **10.4** Wijzig de rol van een gebruiker van "member" naar "admin"
  - Verwacht: Rol wordt bijgewerkt, toast bevestiging
- [ ] **10.5** Verwijder een gebruiker
  - Verwacht: Bevestigingsdialoog, gebruiker verdwijnt na bevestiging

---

## 10b. Werkruimtes (Workspaces)

> Prioriteit: HOOG - nieuwe kernfunctionaliteit voor intra-tenant access control.

### Werkruimtes pagina

- [ ] **10b.1** Navigeer naar "Werkruimtes" in de sidebar
  - Verwacht: Overzichtspagina met werkruimte-kaarten (of lege staat als er nog geen zijn)
- [ ] **10b.2** Klik op een werkruimte-kaart
  - Verwacht: Detailpagina laadt met naam, type badge, kleur, leden, en vergaderingen

### Werkruimtes beheer (Instellingen)

- [ ] **10b.3** Navigeer naar Instellingen > Werkruimtes
  - Verwacht: Tabel met bestaande werkruimtes (of upgrade-melding bij Starter plan)
- [ ] **10b.4** Klik op "Nieuwe Werkruimte" en vul het formulier in (naam, type, beschrijving, kleur)
  - Verwacht: Werkruimte verschijnt in de tabel na opslaan, toast "succesvol aangemaakt"
- [ ] **10b.5** Bewerk een werkruimte (klik potlood-icoon)
  - Verwacht: Dialoog opent met huidige waarden, wijzigingen worden opgeslagen
- [ ] **10b.6** Verwijder een werkruimte (klik prullenbak-icoon)
  - Verwacht: Werkruimte verdwijnt na bevestiging (niet mogelijk voor standaard werkruimte)

### Werkruimte leden

- [ ] **10b.7** Klik op het leden-icoon bij een werkruimte
  - Verwacht: Dialoog opent met huidige leden
- [ ] **10b.8** Voeg een lid toe (selecteer uit tenant gebruikers)
  - Verwacht: Gebruiker verschijnt in de ledenlijst
- [ ] **10b.9** Verwijder een lid
  - Verwacht: Gebruiker verdwijnt uit de ledenlijst

### Meeting-werkruimte toewijzing

- [ ] **10b.10** Open een meeting detailpagina
  - Verwacht: Werkruimte badges zijn zichtbaar (of leeg als nog niet toegewezen)
- [ ] **10b.11** Klik op "+" om een werkruimte toe te wijzen
  - Verwacht: Dropdown met beschikbare werkruimtes, badge verschijnt na selectie
- [ ] **10b.12** Klik op "×" bij een werkruimte-badge om de toewijzing te verwijderen
  - Verwacht: Badge verdwijnt

### Access filtering (werkruimte-gebaseerd)

- [ ] **10b.13** Maak twee werkruimtes aan (bijv. "Sales" en "HR")
  - Verwacht: Beide verschijnen in de werkruimtes lijst
- [ ] **10b.14** Wijs een meeting toe aan werkruimte "Sales"
  - Verwacht: Meeting heeft "Sales" badge
- [ ] **10b.15** Voeg gebruiker A toe als lid van "Sales" maar NIET van "HR"
  - Verwacht: Gebruiker A ziet de Sales meeting, maar niet HR meetings
- [ ] **10b.16** Meetings zonder werkruimte zijn zichtbaar voor iedereen
  - Verwacht: Niet-toegewezen meetings verschijnen in de vergaderingenlijst voor alle gebruikers
- [ ] **10b.17** Admin gebruikers zien ALLE meetings ongeacht werkruimte-lidmaatschap
  - Verwacht: Admins worden niet gefilterd op werkruimte

---

## 11. Tier Limits & Feature Gating

> Prioriteit: LAAG - controleer of limits correct worden weergegeven.

- [ ] **11.1** Controleer of het huidige tier correct wordt weergegeven in de billing sectie
  - Verwacht: "Essentials", "Professional" of "Enterprise" met juiste prijs
- [ ] **11.2** Verbruik sectie toont token gebruik
  - Verwacht: Cijfers zijn realistisch, geen negatieve waarden
- [ ] **11.3** Bij billing exempt tenant: geen Stripe-gerelateerde restricties
  - Verwacht: Volledige toegang zonder abonnements-checks

---

## 12. Bestandsuploads

> Prioriteit: LAAG - maar visueel storend als het niet werkt.

- [ ] **12.1** Upload een logo via Instellingen > Verwerken > Huisstijl
  - Stappen: Sleep een afbeelding of klik om te uploaden
  - Verwacht: Logo wordt getoond, opgeslagen, zichtbaar in documenten
- [ ] **12.2** Upload een bot avatar via Instellingen > Opnemen > Meeting Bot
  - Verwacht: Avatar wordt opgeslagen, bot gebruikt deze avatar in meetings
- [ ] **12.3** Upload een te groot bestand (> limiet)
  - Verwacht: Foutmelding over bestandsgrootte
- [ ] **12.4** Upload een ongeldig bestandstype (bijv. .exe)
  - Verwacht: Foutmelding over bestandstype

---

## 13. Security & Multi-Tenant Isolatie

> Prioriteit: HOOG - fundamenteel voor B2B SaaS.

### Cross-tenant isolatie

- [ ] **13.1** Log in als een gebruiker van Tenant A. Noteer een meeting ID.
  - Verwacht: Meetings van Tenant A zijn zichtbaar
- [ ] **13.2** Log in als een gebruiker van Tenant B (ander account/incognito).
  - Verwacht: Meetings van Tenant A zijn NIET zichtbaar, alleen Tenant B data
- [ ] **13.3** Probeer als Tenant B de meeting URL van Tenant A te openen (`/meetings/{id}`)
  - Verwacht: 404 of "Niet gevonden" - GEEN data van Tenant A zichtbaar

### Superadmin functionaliteit

- [ ] **13.4** Log in als superadmin
  - Verwacht: "Platformbeheer" sectie zichtbaar in sidebar
- [ ] **13.5** Superadmin tenant switcher: wissel naar een andere tenant
  - Verwacht: Dashboard toont data van de geselecteerde tenant
- [ ] **13.6** Superadmin kan alle tenants zien in Platformbeheer > Tenants
  - Verwacht: Lijst met alle tenants, mogelijkheid om te beheren
- [ ] **13.7** Superadmin > Gebruikers toont alle platform gebruikers
  - Verwacht: Tabel met alle gebruikers over alle tenants
- [ ] **13.8** Superadmin > LLM Kosten toont platform-brede kosten
  - Verwacht: Overzicht van Claude API gebruik per tenant

### Rollen en rechten

- [ ] **13.9** Log in als een "member" (niet admin) gebruiker
  - Verwacht: "Instellingen" menu is NIET zichtbaar in de sidebar
- [ ] **13.10** Log in als een "admin" gebruiker van de tenant
  - Verwacht: "Instellingen" menu IS zichtbaar met alle configuratie-opties

---

## 14. Logging

> Prioriteit: LAAG - monitoring feature.

- [ ] **14.1** Navigeer naar "Logging" in de sidebar (indien zichtbaar)
  - Verwacht: Logging pagina laadt met activiteitenlog
- [ ] **14.2** Log entries worden getoond met tijdstempel, actie, gebruiker
  - Verwacht: Recente acties zijn zichtbaar

---

## 15. UI & UX Cross-checks

> Prioriteit: MIDDEL - gebruikerservaring en polish.

### Responsive design

- [ ] **15.1** Open de app op mobiel formaat (F12 > device toolbar, of echt mobiel)
  - Verwacht: Sidebar is ingeklapt, content is leesbaar
- [ ] **15.2** Sidebar toggle knop werkt op mobiel
  - Verwacht: Sidebar opent/sluit als overlay

### Dark mode

- [ ] **15.3** Klik op de theme toggle (zon/maan icoon in de header)
  - Verwacht: UI schakelt tussen light/dark mode
- [ ] **15.4** Controleer leesbaarheid in dark mode (geen witte tekst op wit, etc.)
  - Verwacht: Alle tekst is leesbaar, grafieken zijn zichtbaar

### Cookie consent

- [ ] **15.5** Open de app in een incognito venster
  - Verwacht: Cookie banner verschijnt onderaan
- [ ] **15.6** Accepteer cookies
  - Verwacht: Banner verdwijnt, keuze wordt onthouden

### Publieke pagina's

- [ ] **15.7** Navigeer naar `/privacy` (niet ingelogd)
  - Verwacht: Privacy Policy pagina laadt (in het Nederlands)
- [ ] **15.8** Navigeer naar `/terms` (niet ingelogd)
  - Verwacht: Algemene Voorwaarden pagina laadt
- [ ] **15.9** Navigeer naar `/dpa` (niet ingelogd)
  - Verwacht: Verwerkersovereenkomst pagina laadt
- [ ] **15.10** Navigeer naar een niet-bestaande URL (bijv. `/xyz123`)
  - Verwacht: 404 pagina wordt getoond

### Toast notificaties

- [ ] **15.11** Voer een actie uit die een toast oplevert (bijv. instelling opslaan)
  - Verwacht: Toast verschijnt rechtsonder, verdwijnt automatisch na enkele seconden

---

## 16. Onboarding

> Prioriteit: LAAG - alleen voor nieuwe gebruikers.

- [ ] **16.1** Log in als een gebruiker die nog geen onboarding heeft doorlopen
  - Verwacht: Welcome modal verschijnt met tier-specifieke features
- [ ] **16.2** Start de onboarding flow vanuit de welcome modal
  - Verwacht: Stap-voor-stap guide door het platform
- [ ] **16.3** Voltooi alle onboarding stappen
  - Verwacht: Onboarding wordt afgerond, verschijnt niet opnieuw bij volgende login

---

## Gevonden Issues

| # | Gebied | Beschrijving | Prioriteit | Screenshot/URL |
|---|--------|-------------|------------|----------------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |
| 4 | | | | |
| 5 | | | | |

---

## Na de Test

### Bij geen blokkerende issues

1. **Merge PR**: Merge de `dev` branch naar `main` via GitHub PR
2. **Vercel auto-deploy**: Frontend deployt automatisch naar productie na merge naar `main`
3. **Railway auto-deploy**: Backend deployt automatisch naar productie na merge naar `main`
4. **Smoke test productie**: Voer tests 1.1-1.3, 2.1, 3.1, en 8.1 uit op `map.schakel.ai`
5. **Monitor**: Controleer Railway logs en Vercel deployment logs de eerste 30 minuten

### Bij blokkerende issues

1. **Documenteer** het issue in de tabel hierboven
2. **Fix op dev**: Maak een fix commit op de `dev` branch
3. **Hertest**: Voer de gefaalde test(s) opnieuw uit op staging
4. **Pas merge**: Merge pas naar `main` als alle KRITISCH en HOOG issues zijn opgelost
5. MIDDEL en LAAG issues mogen als known issues worden gedocumenteerd

### Checklist afronden

- [ ] Alle KRITISCH tests (sectie 1, 13) zijn geslaagd
- [ ] Alle HOOG tests (secties 2, 3, 4, 5, 6, 10b) zijn geslaagd
- [ ] MIDDEL tests (secties 7, 8, 9, 10, 15) zijn beoordeeld
- [ ] Gevonden issues zijn gedocumenteerd
- [ ] Go/no-go beslissing is genomen
- [ ] PR naar `main` is aangemaakt (of issues zijn geescaleerd)

---

*Laatst bijgewerkt: 13 februari 2026*
*Platform versie: v0.2 (dev branch, pre-merge)*
