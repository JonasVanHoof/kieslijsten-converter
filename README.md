# kieslijsten converter

create the appropriate data structures based on provided data

## configuring this converter

1. place the required data in `data/db/toLoad`.

- person uri's with a link to their identifier (rrn)
- previously generated administrative bodies for the new legislature
- administrative units, including links to administrative bodies
- mandates, linked to their respective administrative bodies

> [!NOTE]
> If you are using a dump of a pre-existing database instead, make sure that there are no pre-existing instances of RechstreekseVerkiezingen in your database for the target date.

2. place the required source data in `data/input`

- `lijsten.csv` a csv with candidate lists per township (required headers: `kieskring`, `lijstnr`, `lijst`, `datum` )
- `kandidaten.csv` a csv with candidates per list per township (required headers: `kieskring`, `lijstnr`, `verkregen zetels`, `volgnr`, `RRvoornaam `, `RRachternaam`, `RR`, `verkozen`, `opvolger`, `naamstemmen`, `geslacht`, `geboortedatum`)

The kandidaten.csv file contains sensitive data. Do not commit this to this repository. An example kandidaten file with fake information is added to this repo instead. It contains fake results for the Aalter and Aarschot administrative bodies.

3. add any necessary transformation queries in `data/transforms`.
   These queries will run after the input has been loaded, queries should have a `.rq` extension

4. configure the env variables:

- `ENDPOINT`: SPARQL endpoint to connect to ('http://database:8890/sparql')
- `KANDIDATENLIJST_TYPE_IRI`: iri of the type of the candidates list ('http://data.vlaanderen.be/id/concept/KandidatenlijstLijsttype/95de36e5-8c7a-4308-af7b-75afbd943dd2')
- `BESTUURSORGAAN_TYPE_IRI`: iri of the type of administrative body the 'http://data.vlaanderen.be/id/concept/BestuursorgaanClassificatieCode/5ab0e9b8a3b2ca7c5e000005' # gemeenteraad
- `ORGAAN_START_DATUM`: start date of the elected administrative body
- `LOG_LEVEL`: "info" or "debug"
- `INPUT_DATE_FORMAT`: date format used in the CSV's ("%d/%m/%Y")

## running this converter

```
  docker-compose up
```

the end result will be available in ./data/output/

- sensitive data will be written to ./data/output/[date]-[type]-sensitive.ttl
- other data will be written to ./data/output/[date]-[type].ttl

The data being produced is:

- **personen.ttl**: this contains the uri for every person in the kandidaten.csv, with their birth data, gender, first and last name. It also contains their election result (mandaat:Verkiezingsresultaat) with the number of votes, link to the list, mandaat:rangorde and mandaat:gevolg. If a person with a matching RRN is found in the loaded data, the uri for that person is reused.
- **personen-sensitive.ttl**: this contains the identificator for the persoon, holding the RRN
- **rechstreekse-verkiezingen-en-kandidatenlijsten.ttl**: the newly minted uris for instances of mandaat:RechstreekseVerkiezing and mandaat:Kandidatenlijst with all their properties.

A line is commented in the convert-to-ttl script where the function update_kieslijsten is called. It allows the lijstnummers for the kieslijsten to be updated given that the information generated in the previous step is already in the database (so it keeps the old uris but recomputes the kieslijst numbers based on the lijsten.csv).

## extra scripts

#### create-fusiegemeenten.rb

one off script to add some extra administrative units and organs.

#### import-burgemeester.rb

This script needs a `burgemeesters2019.csv` in `burgmeesters/data/input` and a (recent) backup to restore in `burgemeesters/data/db/backups`. The correct prefix needs to be set in docker-compose.yml. Output will be written to `burgmeesters/data/output`, it's a ttl that has the new `mandatarissen`. Errors are logged to stdout.

```
cd burgmeesters
docker-compose up
```

#### data/transforms/new-bestuursorganen-and-mandates

This generates new bestuursorganen and mandates for the 2018 election based on the organen that were previously available (but ignoring some that no longer exist).
