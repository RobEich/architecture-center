Messaging oder Eventing Systeme sind weit verbreitet um z. B. asynchrone Kommunikation zwischen Mikroservices bzw. distributierten Systemen auf dem gleichen Nachrichten Kanal zu ermöglichen. Diese Systeme werden auch häufig benützt, um die Verarbeitung von Nachrichten auf leicht zu skalierende Worker Prozesse zu verteilen. 

Eventing Systeme bauen hierbei häufig auf zwei unterschiedlichen Architekturen auf: 

- [Competing Consumer](./competing-consumers.yml)
- Partitioned Consumer

Ein grundlegendes Verständniss dieser Architekturen und die Funktionalitäten die zur Verfügung gestellt werden ist hilfreich um ein System auszuwählen, welche die Anforderungen an die jeweiligen Applikationen unterstützt. 

### Delivery Patterns
Eventing Systemen liefern häufig eine sogenannte Delivery Guarantees welche Funktionaliät beschreibt die vom Eventing System zur Verfügung gestellt wird: 

- ***At-Most-Once***: A message is delivered zero or one times. This means a message could get lost and there's no guarantee that all messages published to the messaging system will be forwarded to potential consumers. 
- ***At-Least-Once***: A message can be potentially delivered multiple times. This means a message can't get lost but it can be duplicated.
- ***Exactly-Once***: A message will be delivered one time. This means a message can't get lost and can't get duplicated. 

At-Most-Once ist hierbei das günstigste mit der höchsten Performance und dem geringsten Aufwand zur Implementierung. Es muss z. B. nicht in einem Status gespeichert werden, ob das Event bereits geliefert wurde. Es kann eine einfache Fire-and-Forget Funktionalität implementiert werden. 

At-Least-Once muss Retry-Mechanismen implementieren um z. B. bei transienten Fehlern wie einem DNS Hick-up oder Connectivity loss das Event noch einmal zu erhalten. Ein Acknowledgesystem bei der empfangenden Applikation muss ebenfalls implementiert werden.

Exactly-Once wird sehr häufig von Applikationen vorausgesetzt ist aber am aufwendigsten zu implementieren und dadurch in der Skalierung beschränkt bzw. nur mit hohem Resourcenaufwand skalierbar. 

[Azure IoT Hub](https://learn.microsoft.com/en-us/azure/iot-fundamentals/) und [Azure Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-about) basieren beide auf dem Prinzip der Partitioned Consumers. Beide Systeme implementieren [At-Least-Once](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-event-processor-host) Delivery Guarantee mit hoher Skalierungsmöglichkeit und erwarten die Implementierung einer client seitigen Acknowledge Funktionalität. Diese Funktionalität wird im Zusammenhang mit IoT Hub und Event Hubs als Checkpointing bezeichnet. SDKs erleichtern hierbei die Implementierung des Checkpointings. 


## Partitioned Consumers
### Overview IoT Hub / Event Hubs

Einzelne Events werden in IoT Hub und Event Hubs in sogenannten Partitionen gespeichert. Sobald neue Events eintreffen, werden diese am Ende der jeweiligen Partition gespeichert. Eine Partition kann somit als "commit log" betrachtet werden.

![](./_images/partitioned-consumers-partitions.png)

In IoT Hub / Event Hubs werden nicht einzelne Events nach Überschreiten der maximalen [Retention Time](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-faq) aus dem "commit log" entfernt. Vielmehr werden größere Speicher Blöcke aus dem "commit log" entfernt. Dies erfolgt, wenn die Retention Time aller Events überschritten wurde ***und*** zusätzlich ein kompletter Speicherbereich innerhalb einer Partition gefüllt ist. Aus diesem Grund kann es zu Situationen kommen, in denen IoT Hub und Event Hubs Events liefern, deren Retention Time bereits überschritten wurde.

Innerhalb einer Partition speichert IoT Hub und Event Hubs zusätzlich zu den Event Daten Metadaten, wie z. B. den Stream Offset: 

![](./_images/partitioned-consumers-metadataoffset.png)


An offset is the position of an event within a partition. The offset is a byte numbering of the event and together with the Partition Id it uniquely identifies the event. Offset and Partition Id enables an event consumer or client application to specify a point in the event stream from which they want to begin reading events. The offset can be specified as a offset value or a timestamp using IoT Hub / Event Hubs.

### Processed Events
Im Unterschied zu Event Systemen mit Competing Consumers verwaltet ein Eventing System mit Partitioned Consumer keine Informationenen welche Events bereits von Clients abgerufen und verarbeitet wurden. Es wird erwartet, dass diese Funktionalität vom Client zur Verfügung gestellt wird. 

#### Partitions & Partition Reader
Bei der Implementierung dieser Funktionalität ist zu beachten, dass pro Partition nur ein Client, oder genauer ausgedrückt ein Partition Reader Events aus einer explizit zugeordneten Partition lesen kann. Im Gegenzug kann jeweils nur ein Partition Reader mit einer Partition verbunden sein und Events abrufen. Dies bedeuted auch, dass die Anzahl der Partitionen die Anzahl der maximal gleichzeitig lesenden Partition Reader, und damit auch die Performance der Verarbeitung bestimmt.

Das nachfolgende Beispiel zeigt eine Instanz von Event Hubs / IoT Hub mit 4 Partitionen. Eine Applikation mit einem Partition Reader wird verwendet um Events zu lesen und zu Verarbeiten: 
![](./_images/partitioned-consumers-reader01.png)

Um nun eine maximale parallelität beim Lesen der Events zu implementieren wird empfohlen pro Partition einen aktiven Partition Reader zu implementieren. D. h. in dem Beispiel werden 4 Instanzen der Applikation gestartet, die sich jeweils mit einer Partition verbinden um Events aus dieser Partition zu lesen. 

![](./_images/partitioned-consumers-reader02.png)


Theoretisch können maximal 5 Partition Reader konkurierend auf eine einzelne Partition zugreifen. Dies ist nicht empfohlen, da jeder Partition Reader Zugriff auf alle Events in der jeweiligen Partition hat und demenstprechend eine mehrfache Verarbeitung der Events erfolgt bzw. ein Mechanismus zur Synchronisierung der Partition Reader implementiert werden muss. Dies führt häufig zu aufwendigen Custom Lösungen. 

Soll ein Event mehrfach verarbeitet werden z. B. um das Event in einem Long Term Storage zu archivieren und gleichzeit ein NRT Dashboard mit Informationen zu befüllen bietet sich die Verwendung von Consumer Groups an. 

#### Consumer Groups
Die Kopplung eines Partition Readers mit einer Partition bedeuted nicht, dass ein Event nur einmal von einer Applikation gelesen werden kann. Consumer groups enable multiple consuming application to each have a separate view of the events in the partitions, and to read the stream independently at their own pace and with their own offsets. [Details](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-features)


#### Checkpointing
Um Events von einer Partition zu lesen muss die jeweilige Applikation bzw. der Partition Reader sich mit einer Partition verbinden und dem Eventing System mitteilen ab welchem Offset Events gelesen werden sollen. Dies setzt voraus, dass ein Client Informationen darüber speichert, welches Event bereits verarbeitet wurde. Im weitesten Sinne kann diese Information mit einem "Client-Side-Cursor" verglichen werden. 

Dieser Prozess wird ***Checkpointing*** genannnt. Checkpointing is a process by which clients mark or commit their position within a partition event sequences. It is the responsibility of the client application to store the checkpointing information (partition id & offset of already processed events). Um die Checkpointing Informationen jetzt nicht nach einem evtl. Neustart zu verlieren, wird empfohlen dies mit Hilfe eines externen Service zu speichern. 
![](./_images/partitioned-consumers-checkpointing.png)

Für Checkpointing bieten sich verschiedene Dienste wie z. B. [Azure Blob Storage](https://azure.microsoft.com/en-us/products/storage/blobs/), [Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/introduction), [Azure Cache for Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-overview) etc. an. 

Bei der Auswahl des Service zum Persistieren der Checkpoint Informationen müssen Anforderungen an Skalierbarkeit und Reliability abgewogen werden. Best Practice ist, nicht jedes einzelne gelesene Event zu checkpointen sondern Event Batches zu checkpointen. Dies erfolgt um den gewählten Service nicht zu überlasten bzw. um kein Bottle-Neck in der Eventverarbeitung zu kreieren.

## SDK Unterstützung
Microsoft stellt [SDKs](https://learn.microsoft.com/en-us/azure/event-hubs/sdks) für viele Sprachen zur Verfügung. Diese Vereinfachen den Prozess des Checkpointing. Zusätzlich bieten die SDKs Funktionalität die Zuordnung von Partition Readers zu Partitionen mit Anzahl der lesenden Instanzen anzupassen.

## Next steps


## Related resources
