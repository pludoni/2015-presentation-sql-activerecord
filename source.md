name: inverse
class: center, middle, inverse
layout: true

---

name:  normal
class:
layout: true

---

class: center, middle

SQL + ActiveRecord


Handout


---
template: inverse

# Begriffe

---

### SQL

**SQL**: Structured Query Language &ndash; Abfragesprache für Datenbanken

**Datenbank**: normalerweise: Prozess (Programm) (oder Server auf dem das Programm läuft). Kümmert sich um Speicherung von Daten, oft mit besonderen Abfragefeatures

**relationale Datenbank**: spezielle Datenbanken, welche Daten in Relationen (Beziehungen) speichert = Tabellen

**Tabelle**: Wie Excel -> Zeilen + Spalten

**Datenbank-Schema**: Jede Tabelle hat bei relationalen Datenbanken ein Schema, d.h. eine Typdefinition, was für Spalten enthalten sind (=Excel erste Zeile)

**Migration**: Änderung am Schema (Hinzufügen von Spalten, Löschen von Spalten, Neue Tabelle, ...)

---

### SQL Statement:

Beginnt immer mit einem davon:

* SELECT
* INSERT
* DELETE
* UPDATE
* CREATE TABLE
* ALTER TABLE


Jeweils folgt eine eigene spezielle Syntax

Endet immer auf ein <code>``;``</code>

---

## SELECT


```sql
SELECT columns[, columns/functions AS name]
FROM table [,tables..]
[INNER/LEFT] JOIN tables ON join_condition
WHERE conditions
GROUP BY expression
HAVING conditions
ORDER BY expression
LIMIT number;
```

* Reihenfolge ist fest! Mindestens SELECT + FROM, Rest optional.


---

template: inverse

# Kommandos

---

### Anlegen

```sql
.schema
.headers on
.mode column
create table users (
  id INT PRIMARY KEY NOT NULL,
  name varchar(255),
  firma varchar(255),
  account_type int
);
select * from users;
insert into users (id, name, firma, account_type) values
 (1, 'pludoni GmbH', 'pludoni', 1),
 (2, 'TU Dresden', 'tud', 2),
 (3, 'T-Systems MMS', 'tsystemsmms', 1);
select * from users;

select account_type, count(*) from users group by account_type;
SELECT account_type, count(*) count_name
FROM users
GROUP BY account_type;

```

---

### 2. Tabelle

```sql
create table jobs (
  id INT PRIMARY KEY NOT NULL,
  title varchar(255),
  job_type integer,
  user_id int not null
);
insert into jobs (id, title, job_type, user_id)
  VALUES (1, 'Ruby Entwickler', 4, 1),
         (2, 'Java Entwickler', 4, 3),
         (3,  'Java Architekt', 4, 3),
         (4,  'Java Architekt', 5, 3);
select * from jobs;
```

---

### Joins

```sql
select * from jobs
inner join users on users.id = jobs.user_id;

select jobs.*, users.name as user_name, account_type
  from jobs
  inner join users on users.id = jobs.user_id
  WHERE job_type = 4
  ORDER BY "user_name"
  desc LIMIT 1;
select job_type, count(*) as count from jobs
  inner join users on users.id = jobs.user_id group by job_type;
```


---

### Left join + SUM()

```sql
alter table jobs add application_count INT;
.schema

update jobs set application_count = 1 where id = 1;
update jobs set application_count = 5 where id = 2;
update jobs set application_count = 21 where id = 3;
update jobs set application_count = 4 where id = 4;
select * from jobs;

select * from users INNER JOIN jobs on jobs.user_id = users.id;

select firma, sum(application_count)
  from users
  INNER JOIN jobs on jobs.user_id = users.id
  group by firma;
select firma, sum(application_count) from users
  LEFT JOIN jobs on jobs.user_id = users.id group by firma;
```

---

template: inverse

# ActiveRecord


---

# Was ist ActiveRecord

* Ruby-Gem
* SQL-Abstraktion in Rails
* ORM - Objekt-Relationales-Mapping <br/><small>d.h. Tabellen entsprechen Klassen (Model) und Zeilen konkreten Objekt-Instanzen</small>

---

# Grundkonzepte:

* Models erben von <i>ActiveRecord::Base</i>
* Ein Model entspricht 1:1 einer Tabelle
* AR nimmt Datenbank als gegeben (=Wahrheit) an <br /><small>Convention > Configuration</small>

---

* Seit Version 3 auch Query-Builder-Language (Gem: arel)

```ruby
sql = Job.where(visible: true)
sql = sql.where(organisation_id: Organisation.partner).
  limit(30)
if params[:order] == 'date'
  sql = sql.order('pubDate desc')
end

```

* <code>ActiveRecord::Relation</code> Objekte können verkettet werden und werden **lazy** verarbeitet <br/>
<small>(d.h. erst beim Aufruf von ``.each / map / group_by / reject / .count /.first``)</small>

* Reihenfolge der where/order/limit/... egal, AR sortiert sie dann zum Aufruf

---

template: inverse

## Migrationen

---

* Migrationen im Ordner ``db/migrate/20131015095346_create_users.rb``
* der Zeitstempel (20131015095346) ist die ID der Migration

rake db:migrate:
* Rails schaut in die Tabelle ``schema_migrations`` rein (oder legt sie an)
* Fehlende Migrationen werden ausgeführt
* Id der Migration kommt mit in die schema_migrations Tabelle

---

### Kommandos

rake db:migrate:redo [VERSION=20131015095346]
* rollback'd letzte Migration und führt sie erneut aus (Optional Angabe der VERSION)

rails g migration add_job_type_to_jobs job_type:integer
* Einzige "Magic" Migration ->
* erzeugt direkt eine Migration die die Spalte hinzufügt

rake db:schema:load / rake db:schema:dump
* schema:dump wird automatisch nach db:migrate ausgefuehrt, d.h. db/schema.rb wird erzeugt
* Nutzen: Mit db:schema:load direkt aktuellen Schema reinladen (loescht DB) -> Sinnvoll z.B. fuer CI, oder wenn es schon seeehr viele Migrationen gibt (oder die Migrationen gar nicht mehr ordentlich durchlaufen)

---

template: inverse

# Advanced

---

### pluck
```ruby
Job.visible.pluck(:title)

# gleiche wie:
Job.visible.select('title').map{|i| i.title }
# nur bedeutend effizienter
```

---
### update_all / delete_all

```ruby
Job.where('created_at < ?', 1.year.ago).update_all visible: false
# Extrem effizient

Job.where('created_at < ?', 1.year.ago).delete_all
```
---

### benannte Platzhalter

```ruby
Job.where('from_date <= :today and :today <= :today', today: Date.today)
```

(geht auch mit SQL)
```ruby
Job.where(':today between (from_date and to_date)', today: Date.today)
```

---

### Neuste Stelle je Nutzer

Kompliziert, da innerhalb einer Gruppierung ein bestimmtes Kritierium gewünscht ist

**Datenbankspezifisch** z.B. bei PostgreSQL am besten mit Distinct on:

```ruby
Organisation.
  joins(:jobs).
  order('name, jobs.pubDate').
  pluck('distinct on (name) organisations.name, jobs.id')
```

Bei Standard-SQL nur mittels Sub-Selects machbar.

---

### Komplexbeispiel

* "Ich brauche auf der Startseite die News der Partner, aber je Partner nur die neusten 2"
* "Die dann aber sortiert, sodass die neusten oben sind"
* "Ach, und nur welche wo Veroeffentlichungsdatum nicht in der Zukunft liegt"

Tabellen:  feed_items -> feed_id -> organisation_id

---


```sql
WITH newest_feed_items AS (
  SELECT * , ROW_NUMBER() OVER (
      PARTITION BY feed_id ORDER BY published_at DESC
    ) AS rank
  FROM feed_items
)
SELECT id FROM newest_feed_items
WHERE rank <= 2 AND published_at IS NOT NULL
AND published_at < now()
ORDER BY published_at DESC;
```

```ruby
sql = ".."
Organisation.connection.execute(sql).to_a.map{|i|i.values.first}
```

Reines FYI

---

### Lessions learned

* Spalten niemals so nennen wie AR Methoden heissen:
<pre>save, valid, update, type, frozen, changed, reload, touch, ...</pre>

* Transaktionen bei mehreren wichtigen Anweisungen:

```ruby
Organisation.connection do
  @user.save
  @job.save
end
```

---

### Noch offen:

* Relationships belongs_to, has_many, habtm, has_one
* Polymorphic Associations

---

# Erwähnung ACID

Anforderungen, die jede relationale Datenbank erfüllen muss:

* Atomicity
  <br/><small>Transaktionen - Alle Schritte oder keine</small>
* Consistency
  <br/><small>Wenn in die DB geschrieben / gelöscht wird, dann werden immer alle Randbedingungen geprüft (Constraints)</small>
* Isolation
  <br/><small>Transaktionen sind getrennt und laufen sich nicht in den Weg, z.B. durch LOCKs</small>
* Durability
  <br/><small>Nach Crash bleiben die Daten erhalten</small>

---
template: inverse

# NoSQL

---

# CAP Theorem

<img src='http://berb.github.io/diploma-thesis/community/resources/cap.svg' style='width: 50%; float:right;'/>

* Consistency
  <br/><small>(Daten stimmen immer)</small>
* Availability
  <br/><small>(Ergebnis ist sofort verfügbar)</small>
* Partition Tolerance
  <br/><small> (Mehr als 1 Server)</small>

Man kann immer nur 2 davon haben!

Anforderung: "High-Scale" --> mehr als 1 Server, d.h. entweder Consistency oder Availability wichtiger

---

## NoSQL Flavours

* Key-Value
  <br/><small>Redis, MemcacheD, Riak</small>
* Dokumentendatenbanken
  <br/><small>MongoDB, CouchDB, ElasticSearch</small>
* Graph-Datenbankenk
  <br/><small>Neo4j, Arango</small>
* Columnar-Databases
  <br/><small> Cassandra, HBase</small>

https://www.digitalocean.com/community/tutorials/a-comparison-of-nosql-database-management-systems-and-models

---

template: inverse

# Fin.

**Stefan Wienert**

pludoni GmbH

<a href='https://twitter.com/stefanwienert'>@stefanwienert</a> / github.com/zealot128

