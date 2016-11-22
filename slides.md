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
* Applicerar affärslogik utifrån kommandot och det state agregatet själv innehåller.
* Aggregatet genererar ett event som den applicerar på sig själv.




---
## Hur det används i Tickra
* Används framförallt för projekt och uppgifter.
* Snapshots i databasen.
* Events serialiseras ner som json med versionsnr i databasen.
--
## Fixa bild som beskriver föregående slide


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
## Fixa bild som beskriver föregående slide

---

## Skapa en uppgift i Tickra
```cs
public class ProjectTask : AggregateRoot<ProjectTask.State>
{
    public class State
    {
        public ProjectTaskId ProjectTaskId { get; set; }
        public string Name { get; set; }
    }
    // Public method is called from controller or similar. 
    public ProjectTask(ProjectTaskId projectTaskId, projectId....)
    {
        ValidateProjectState(projectState, projectTaskId);
        ValidateDetails(name, description);
        ApplyChange(new ProjectTaskCreated(projectTaskId, projectId...));
    }
    // Apply the event to the state / snapshot
    private void Apply(ProjectTaskCreated @event)
    {
        Snapshot.ProjectTaskId = @event.ProjectTaskId;
        Snapshot.ProjectId = @event.ProjectId;
        Snapshot.Name = @event.TaskName;
    }
}
```
--
## Sätta uppskattad tidåtgång
```cs
// Public method is called from controller or similar. 
public void SetEstimate(IProjectTaskService projectTaskService, ProjectState projectState, int? estimate)
{
    ValidateProjectState(projectState, Snapshot.ProjectTaskId);

    if (Snapshot.Estimate != estimate)
    {
        ApplyChange(new ProjectTaskEstimateSet(Snapshot.ProjectTaskId, Snapshot.ProjectId, estimate, DateTimeOffset.Now));
    }
}

// Apply the event to the state / snapshot
private void Apply(ProjectTaskEstimateSet @event)
{
    Snapshot.Estimate = @event.Estimate;
    Snapshot.IsOverdrawn = false;
}
```
--
## Projektioner
```cs
public class ProjectTaskEstimateSet : UserIssuedEvent
{
    public readonly ProjectTaskId ProjectTaskId;
    public readonly ProjectId ProjectId;
    public readonly int? Estimate;

    public ProjectTaskEstimateSet(ProjectTaskId projectTaskId, ProjectId projectId, int? estimate, DateTimeOffset timeStamp): base(timeStamp)
    {
        ProjectTaskId = projectTaskId;
        ProjectId = projectId;
        Estimate = estimate;
    }
}
// In the projections we listen to the event and update the projection with the event.
public async Task HandleAsync(ProjectTaskEstimateSet @event)
{
    await UpdateProjectLastModified(@event.ProjectId, @event.TimeStamp);
    await UpdateProgressWithEstimateOrStatus(@event.ProjectId, @event.ProjectTaskId, @event.Estimate, null);
}
```
--
## Läsa upp ett aggregat
```cs
public async Task<TEventSource> GetAsync(Guid aggregateId, int? version = null)
{
    var aggregate = new TEventSource();
    // Läser upp ett snapshot om det finns sparat.
    var snapshot = await _snapshotStore.LoadSnapshotAsync(aggregateId, version);
    EventStream eventStream = null;
    if (snapshot != null)
    {
        // Har vi ett snapshot så sätter populerar vi aggregatet från det.
        aggregate.LoadFromSnapshot(snapshot);
        // Läser upp eventuella events med högre version än vårat snapshot.
        eventStream = await _eventStore.LoadEventStreamAsync(aggregateId, snapshot.Version, version);
    }
    else
    {
        // Har vi inget snapshot så läser vi upp alla events.
        eventStream = await _eventStore.LoadEventStreamAsync(aggregateId, 0, version);
        if (eventStream == null)
            return null;
    }

    // Hittade vi några events så populerar vi aggregatet med dessa event.
    if (eventStream != null)
        aggregate.LoadFromHistory(eventStream);
    return aggregate;
}
```
--
## Spara ner ett aggregat
```cs
public async Task SaveAsync(TEventSource aggregate, int? expectedVersion)
{
    var changes = aggregate.GetChanges();

    foreach (var change in changes)
    {
        var userIssuedEvent = change as UserIssuedEvent;
        if (userIssuedEvent != null)
        {
            if (_principal == null)
            {
                throw new InvalidOperationException("A change that requires a userId cannot be saved if no userId is available!");
            }
            userIssuedEvent.IssuedByUserId = _principal.Identity.GetUserId<long>();
        }
    }

    while (true)
    {
        try
        {
            await _eventStore.AppendStreamAsync(aggregate.Id, changes, aggregate.Version);
            break;
        }
        catch (OptimisticConcurrencyException ex)
        {
            throw ex;
            //aggregate.ResolveConflicts(ex.CommittedEvents);
        }
    }

    var snapshot = aggregate.GenerateSnapshot();
    await _snapshotStore.SaveSnapshotAsync(snapshot);

    foreach (var @event in changes)
    {
        await _eventBus.PublishAsync(@event);
    }
    aggregate.ClearChanges();
}
```
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
## När ska man använda det?

Komplexa entiteter med mycket affärslogik.

Eller för ökad spårbarhet.

Ger onödig overhead för simpla CRUD'ar.

---
## Färdiga alternativ med .NET api
* Eventstore
* Marten - Postgressql
* Streamstone - Azure Table Storage
---
## THE END