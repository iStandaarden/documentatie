# Technische verdieping Abonneren &amp; Notificeren

In het document &quot;Concretisering architectuur en ontwerp iWlz Netwerkmodel&quot; is het proces rondom Abonneren en Notificeren beschreven in het hoofdstuk Applicatie architectuur. Hier is onder andere in terug te vinden dat abonnementen en notificaties via het Netwerkpunt worden uitgewisseld, maar dat het vastleggen van abonnementen en het afgeven van triggers voor notificaties de verantwoordelijkheid is van de backoffice applicatie.

In dit document worden deze functionaliteiten nader toegelicht. De inhoud van dit document zal na akkoord worden opgenomen in het hiervoor genoemde stuk.

## ![](RackMultipart20210323-4-1eo1q20_html_7b65b5634574bdd2.png)

##


## Abonneren

Een netwerkpartij kan zich abonneren op een gebeurtenissen (events) die plaatsvinden in een register, die relevant zijn voor het interne werkproces van de netwerkpartij. Een abonnement wordt geplaatst op een record. Een record is een algemene term voor een gegeven dat bekend is binnen een register, bijvoorbeeld een WLZ indicatie of een Zorglevering. We onderkennen 3 typen abonnementen:

1. Een abonnement op events met betrekking tot een specifiek record

→ Sleutel = recordId

1. Een abonnement op events met betrekking tot records in relatie tot een persoon

→ Sleutel = BSN (of pseudoniem)

1. Een abonnement op events met betrekking tot records in relatie tot een organisatie

→ Sleutel = Geaccepteerde organisatie code (Uzovi, AGB, enz)

Bij het plaatsen van een abonnement wordt aangegeven bij welk Event type de partij een notificatie wenst te ontvangen en optioneel bij wijziging van welke velden een organisatie een notificatie wenst te ontvangen.

_Bijvoorbeeld: Als [CAK] abonneer ik me op alle [Zorgleveringen] waarbij het gegeven [leveringsvorm] wijzigt._

| **Abonnement gegeven** | **Toelichting** |
| --- | --- |
| Abonnement type | Soort abonnement dat moet worden geplaatst: Record, Persoon, Organisatie |
| Identificatie type | Identificatie type dat gebruikt wordt in het abonnement, bijvoorbeeld RecordId, BSN, Uzovi, AGB |
| Identificatie | Identificatie van hetgeen waar het abonnement op geplaatst moet worden |
| Record type | Soort record waar het abonnement op geplaatst dient te worden. Bijvoorbeeld: Wlz indicatie, Zorglevering, enz |
| Event type | Soort event waarbij de abonnee een notificatie wil ontvangen. [Create, Update, Delete] |
| Velden (optioneel) | Alleen wanneer een wijziging in één van deze velden optreed ontvangt de abonnee een notiticatie. Indien leeg ontvangt de abonnee een notificatie bij wijziging van alle velden |
| Startdatum | Datum waarop het abonnement in moet gaan |
| Einddatum | Datum waarop het abonnement moet worden beëindigd |

Wanneer er een event waarop een partij zich heeft geabonneerd zich voltrekt, ontvangt de partij die het abonnement heeft ingesteld hier een notificatie van. De notificatie bevat geen persoonlijke of inhoudelijke gegevens, enkel een referentie naar het abonnement, het event type en het recordID (bijvoorbeeld de URI van de gewijzigde Zorglevering).

Een succesvol geplaatst abonnement is direct actief. Een abonnee zal dus direct na het plaatsen van het abonnement genotificeerd worden over het voltrekken van het event.

Het onderstaande diagram geeft de &quot;route&quot; die een abonnement verzoek aflegt grafisch weer.

## ![](RackMultipart20210323-4-1eo1q20_html_a0cc943205f999df.png)

### Procesverloop abonneren

1. Een gegevensafnemer (bijvoorbeeld: zorgleverancier, zorgkantoor, e.d.) verstuurt een abonnements verzoek via het interne koppelvlak van het Netwerkpunt.
2. De Train Dispatcher (netwerkpunt 1) routeert het verzoek naar het ontvangende Netwerkpunt.
  1. _Indien nodig raadpleegt de TrainDispatcher de Verwijsindex_
  2. _Uit de verwijsindex ontvangt de Subscription Handler de netwerkpartijen die informatie bezitten over de cliënt_
  3. _Via de Station Directory haalt het Netwerkpunt de bijbehorende endpoints op als deze niet voorkomen in de cache._
3. De Train Dispatcher (netwerkpunt 1) stuurt vervolgens een abonnement verzoek naar de abonneer-service van de betreffende Netwerkpunten van de gegevens producent(en) (bronhouder(s)).
4. De Subscription Handler (netwerkpunt 2) stuurt het abonnement verzoek door naar de backoffice (Koppelvlak Backoffice, operatie &quot;postSubscription&quot;
  1. Indien de backoffice niet beschikbaar is geeft het Netwerkpunt een foutcode 400 terug aan de afzender
5. Backoffice applicatie legt het abonnement vast en retourneert een abonnementId (200 OK + abonnementId of 400 indien er een fout optreed) (synchroon).
6. De Subscription Handler (Netwerkpunt 2) persisteert het abonnementId, organisatieId en endpoint.
7. De Subscription Handler (Netwerkpunt 2) retourneert het abonnementId naar het vragende Netwerkpunt (synchroon).
8. De Train Dispatcher (netwerkpunt 1) persisteert het abonnementId, abo info en endpoint in de database van het Netwerkpunt en zet het abonnementId door naar de backoffice applicatie.

![](RackMultipart20210323-4-1eo1q20_html_603591764545bf7a.png)

### Abonnement intrekken

Een abonnement kan worden ingetrokken door een DELETE operatie uit te voeren op het abonnementId bij het eigen netwerkpunt. Het netwerkpunt zet het intrekkingsverzoek door naar het netwerkpunt waar het abonnement geplaatst is.

![](RackMultipart20210323-4-1eo1q20_html_bedd79476ce2c1d2.png)

## Notificeren op abonnement

Wanneer een record wijzigt (event) waar een abonnement op geplaatst is, verstuurt de backoffice een notificatie naar het netwerkpunt. Deze notificatie bevat het abonnementId, het event type, het timestamp en het recordId (unieke identifier).

Notificaties worden reliable verstuurd door de Event Publisher. Reden hiervoor is dat deze notificaties essentieel zijn voor het soepel laten verlopen van het administratieve proces, notificaties moeten dus altijd worden afgeleverd.

Het onderstaande diagram geeft de &quot;route&quot; die een notificatie aflegt grafisch weer.

![](RackMultipart20210323-4-1eo1q20_html_99ccb6e64873a90.png)

### Procesverloop Notificeren

1. Het backoffice systeem detecteert een wijziging in een gegeven (recordID) waarop een abonnement is geplaatst (monitor event).
2. Het backoffice systeem meldt via het interne koppelvlak van het Netwerkpunt dat er een wijziging heeft plaatsgevonden voor de betreffende recordID door een notificatie te posten naar het eigen netwerkpunt (Koppelvlak Netwerkpunt, operatie &quot;sendNotification&quot;)
3. De Event Publisher zoekt het endpoint van het ontvangende netwerkpunt op op basis van het abonnementId.
4. Vervolgens wordt via de Event Publisher de notificatie naar Event Handler van het betreffende DataStation verzonden.
  1. _Indien de notificatie niet kan worden afgeleverd wordt middels een retry-mechanisme later nog een poging gedaan. Na x onsuccesvolle pogingen komt de notificatie op een uitvallijst te staan._
5. Het ontvangende DataStation persisteert de notificatie en zet de notificatie door naar het backoffice systeem (koppelvlak Backoffice, Verwerken notificatie)
6. Het backoffice systeem bepaalt vervolgens hoe de notificatie verwerkt wordt en haalt indien nodig de nieuwe gegevens op via het proces Raadplegen.

![](RackMultipart20210323-4-1eo1q20_html_b432828b5c2741d4.png)

## Raadplegen abonnementen

Via het interne koppelvlak Netwerkpunt kan een organisatie bij zijn eigen netwerkpunt raadplegen welke abonnementen de organisatie heeft geplaatst bij andere registers. Ook kan via dit koppelvlak worden nagegaan welke netwerkpartijen zich op gegevens van de organisatie hebben geabonneerd.

## Intrekken abonnementen andere organisatie

Een organisatie heeft de mogelijkheid om abonnementen die door een andere organisatie zijn geplaatst, met opgave van reden, in te trekken. Wanneer dit gebeurt ontvangt de abonnee een notificatie met de melding dat het abonnement is ingetrokken en wat de reden van de intrekking.

![](RackMultipart20210323-4-1eo1q20_html_b49cfb721ba75a77.png)

##


## API specificaties

[Koppelvlak Indicatieregister](https://github.com/iStandaarden/iWlz-indicatie)

[Koppelvlak Netwerkpunt](https://github.com/iStandaarden/iWlz-generiek)

[Koppelvlak Backoffice](https://github.com/iStandaarden/iWlz-generiek)

//AANTEKENINGEN

Intrekken abonnement ophangen aan geldigheid consent

Vastleggen consentID in Netwerkpunt bij zetten abonnement, toetsen of consentID geldig is bij versturen notificatie

Abonnementen intrekken bij ongeldige consent als blijkt bij versturen notificatie

Notificatie service consents definiëren voor Backoffice

Bij verstrekken en intrekken consents (die demo)

Service voor het ophalen van consents bij NP?

Uitgever trekt consent in

Autoservice geeft dit door aan de ontvanger van consent

NP trekt vervolgens abo

BO kan dit opvragen bij NP

Uitwerken tabellen NP

Voorkomen dubbele administratie

Minimaliseren impact backoffice

Notificatie bij verwijderen abonnement door bron + reden

Extra duiding elementen Swagger files

Doel notificatie is het bevorderen van de voortgang van het proces.

Hiervoor moet een abonnement op een specifiek gegeven kunnen worden gezet.

Het toetsen van de impact van de wijziging ligt bij de afnemer.
