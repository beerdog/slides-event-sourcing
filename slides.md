# Event sourcing
---
## Vad är event sourcing?

Ett alternativ till vanliga C(R)UD-operationer direkt mot databasen.

En serie av händelser som tillsammans beskriver ett tillstånd.

Beskriver också hur vi tog oss till det tillståndet.

--
## Definitioner


#### Ett event är 
* En effekt av ett kommando (t.ex. ett http-anrop)
* Påverkar nuvarande state av ett aggregat.
* Immutable

--
## Definitioner fortsättning
#### Ett aggregat är
* Subjektet som påverkas av ett kommando.
* Applicerar kommandot på sig själv och genererar ett event.

--



---
## Hur det används i Tickra
* Används framförallt för projekt och uppgifter.
* Snapshots i databasen.
* Events serialiseras ner som json med versionsnr i databasen.
--
## Implementationsdetaljer
* Ett kommando skickas in
* Vi läser upp aggregatet:
  * Från snapshot 
  * Genom att applicera alla event på ett aggregat.
* Aggregatet applicerar kommandot på sig själv.
* Aggregatet skickar ut ett event till eventulla lyssnare
  * Lyssnare är t.ex. projektioner som behöver uppdateras.
* Event och snapshot sparas ner i databasen.
  * OptimisticConcurrencyException
--
## Lösning av OptimisticConcurrencyException

Vi förväntade oss att aggregatet skulle ha version 2 men hade version 3.

Exceptionet innehåller event med version 3.

Eventuellt möjligt att lösa konflikten med informationen i version 3.

Annars returnerar vi ett fel till användaren.

Skulle kunna lösas genom att köra ett event åt gången. 
---
## Vad är det bra för?

* Logiken på ett ställe - i aggregatet.
* Lättare att debugga.
* Auditlog.
* Mer information som man kanske inte vet om man behöver.
* Flera olika projektioner utifrån multipla aggregat.
* Påverka andra delar som WebJob. 
---
## Färdiga alternativ med .NET api
* Eventstore
* Marten - Postgressql
* Streamstone - Azure Table Storage
---
## När ska man använda det?

Komplexa entiteter med mycket affärslogik.

Ger onödig overhead för simpla CRUD'ar.

---
## 